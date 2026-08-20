---
title: "Kendi Background Worker Kütüphanemizi Neden Yazdık?"
date: 2026-08-19 21:00:00 +0300
categories: [DSO,Architecture, Integration]
tags: [dso,csharp, dotnet, distributed-systems, queue]
description: "Minimal Background Job Queue & Worker for .NET"
---

# Hazır Kütüphaneler (Hangfire vb.) Yetmedi mi? Hangi Performans İhtiyacı Doğdu?

.NET ekosisteminde arka plan işleri (background jobs), kuyruk yönetimi ve eşzamanlı görev işleme (concurrent task processing) denildiğinde akla ilk gelen çözümler genelde **Hangfire**, **Quartz.NET** veya **RabbitMQ / MassTransit** gibi popüler kütüphane ve araçlardır.

Ancak yüksek hacimli veri entegrasyonları, milyonlarca mesajın milisaniyeler içinde işlenmesi gereken ETL süreçleri, IoT veri akışları ve mikro-saniye hassasiyetindeki servislerde bu "hazır ve genel geçer" kütüphaneler zamanla ciddi birer **performans darboğazına (bottleneck)** dönüşebilir.

Peki tam olarak nerede tıkandık? Neden Hangfire gibi rüştünü ispatlamış kütüphaneler yetmedi ve **DSO.Core.QueueWorker** gibi özel, ultra-hafif ve performansa odaklı bir mimari geliştirmek zorunda kaldık?

---

## 1. Hangfire ve Geleneksel Kuyruk Kütüphanelerinin Sınırları

Hangfire ve benzeri araçlar **"Developer Experience" (Geliştirici Deneyimi)** ve **zengin özellik seti** (UI Dashboard, persistent storage, Cron scheduling, retry mekanizmaları) için optimize edilmiştir. Ancak bu zenginlik beraberinde kaçınılmaz maliyetler getirir:

### A. I/O ve Veritabanı Bağımlılığı (Storage Bottleneck)
Hangfire doğası gereği durum yönetimini (State Management) persistence katmanına (SQL Server, Redis vb.) yazar. 
- Her bir job oluşturma, durum değiştirme (Processing, Succeeded, Failed) ve silme işlemi bir **Network I/O** ve **Database I/O** talebi üretir.
- Saniyede 50.000+ iş parçacığının üretildiği ve tüketildiği senaryolarda, işin kendisinden çok **kuyruk durumunun veritabanına yazılması** sistemi kilitler.

### B. Garbage Collector (GC) Baskısı ve Allocation Maliyeti
Geleneksel kütüphaneler genel kullanım senaryolarına uyum sağlamak için yoğun biçimde `object` boxing/unboxing, dynamic serialization (JSON/Binary) ve `Task` allocate eder.
- Yüksek throughput altında bu durum **Gen0 / Gen1 / Gen2 Garbage Collection** frekansını tavan yaptırır.
- GC "Stop-the-World" (LOH pauses) duraklamaları nedeniyle sisteminizde **latency (gecikme) dalgalanmaları** başlar.

### C. Abstraction Layer ve Context Switching
Genel amaçlı framework'ler soyutlama (abstraction) katmanlarıyla doludur. Arka planda çalışan Thread Pool yönetimi, `System.Threading.Channels` veya ham bellek yapıları yerine yüksek seviyeli event/delegate wrapper'ları kullandığında, işlemci çekirdekleri üzerinde gereksiz **Context Switch** yükü oluşur.

---

## 2. Hangi Performans İhtiyacı Doğdu?

Geliştirdiğimiz kurumsal veri entegrasyonu ve yüksek hızlı veri taşıma mimarilerinde hedeflerimiz şunlardı:

1. **In-Memory Zero-Allocation Processing:** Dış bir veritabanına veya Redis'e ihtiyaç duymadan, uygulama belleğinde (RAM) minimum bellek ayak izi (footprint) ile çalışmak.
2. **Lock-Free / High-Concurrency Throttling:** Çekirdek seviyesinde kilitleme (lock/semaphore) maliyetlerini sıfıra yakın tutarak yüksek eşzamanlılık sağlamak.
3. **Predictable Throughput & Low Latency:** İşlem süresinin anlık GC duraklamalarından veya I/O darboğazlarından etkilenmediği, kararlı bir akış elde etmek.
4. **Hafif ve Bağımlısız (Zero-Dependency) Mimari:** Web API, Worker Service veya konsol uygulamalarına ek yüklü paketler çekmeden entegre olabilmek.

İşte **`DSO.Core.QueueWorker`** tam olarak bu ihtiyaçların bir sonucu olarak doğdu.

---

## 3. DSO.Core.QueueWorker Mimarisi Nasıl Fark Yaratıyor?

`DSO.Core.QueueWorker`, yüksek performanslı C#/.NET yapısını ve low-level optimizasyon tekniklerini odağına alır:

### 🚀 1. Memory Optimized Storage (System.Threading.Channels & Ring Buffers)
Geleneksel kuyruklar veritabanına yaslanırken, `DSO.Core.QueueWorker` belleği doğrudan ve verimli kullanacak şekilde tasarlanmıştır. `ChannelReader<T>` / `ChannelWriter<T>` ve thread-safe yapıların modern C# (Modern .NET / C# 10+) imkanlarıyla harmanlanması sayesinde kilitlenme (deadlock) riski ortadan kaldırılır.

### 🚀 2. Low Allocations ve Minimal Thread Overhead
`DSO.Core.QueueWorker`, her bir iş için bağımsız ağır objeler türetmek yerine **ValueTask**, **Span<T>** / **Memory<T>** mantığına uygun bir veri akışını destekler. Bu sayede RAM kullanımı düz bir çizgide kalır ve Heap üzerinde gereksiz çöp birikmez.

### 🚀 3. Doğrudan Kestrel ve Host Integration
Worker mimarisi, `.NET Generic Host` (`IHostedService`) ile tam uyumlu çalışır. Uygulamanız ayağa kalktığı anda worker'lar CPU çekirdek sayınıza (veya yapılandırdığınız ideal concurrency seviyesine) göre ayağa kalkar, bellekteki iş parçacıklarını **non-blocking** olarak tüketir.

---

## 4. Karşılaştırma Tablosu

| Özellik / Kriter | Hangfire / Quartz | DSO.Core.QueueWorker |
| :--- | :--- | :--- |
| **Odak Noktası** | Feature-rich, UI Dashboard, Persistent Jobs | Ultra-High Throughput, Low Latency, In-Memory |
| **Persistence** | Zorunlu (SQL, Redis, Mongo vb.) | In-Memory (Zero I/O Latency) |
| **Throughput (İş/Saniye)** | ~1.000 - 5.000 op/sec (Storage'a bağlı) | **100.000+ op/sec** |
| **GC / Memory Footprint** | Yüksek Allocation & GC Baskısı | Minimal / Near-Zero Allocation |
| **Dependency** | Ağır dış kütüphaneler & DB Sürücüleri | Bağımsız / Lightweight Core Library |
| **Kullanım Alanı** | Zamanlanmış Cron işleri, Admin panelli job'lar | Yüksek hızlı ETL, Event Processing, Stream Data |

---

## Sonuç

"Hazır kütüphaneler kötü müdür?" **Kesinlikle hayır.** 
Amacımız **milisaniyelerin ve işlemci çekirdeklerinin sınırlarını zorlamak**, saniyede yüz binlerce veriyi bellek üzerinde sıfıra yakın gecikmeyle işlemek ve Garbage Collector ile savaşmak zorunda kalmamak, **DSO.Core.QueueWorker** gibi terzilik işi (custom-tailored), low-level odaklı çözümler bir tercih değil **zorunluluktur**.

---

* `DSO.Core.QueueWorker` açık kaynak kodlarına ve kullanım detaylarına GitHub repository'miz üzerinden ulaşabilirsiniz:*  
👉 **[DSOpenServer/DSO.Core.QueueWorker](https://github.com/DSOpenServer/DSO.Core.QueueWorker)**


