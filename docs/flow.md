# LariKuy — Backend Engineering Specification & Implementation Guide

Dokumen spesifikasi teknis, arsitektur sistem terdistribusi, alur pemrosesan data, serta panduan implementasi bertahap untuk **LariKuy (AI Personal Running Coach & Spatial Route Engine)**.

> **Catatan Ruang Lingkup (Backend-Focused):** Proyek ini dirancang untuk mendemonstrasikan kapabilitas **Senior Backend Engineering**. Antarmuka pengguna (*frontend*) dibatasi hanya sebagai dashboard minimalis (Next.js / Swagger UI) untuk memicu API dan memvisualisasikan respons JSON. Seluruh logika bisnis, *stream processing*, *spatial calculation*, dan *orchestration* berada di sisi *backend*.

---

## 1. Spesifikasi Kebutuhan Sistem (Requirements)

### A. Functional Requirements (FR)
1. **FR-01 — Activity Ingestion & Webhook Idempotency:** Menerima data aktivitas lari (baik dari Strava API maupun *Simulator Webhook Endpoint*), mencegah duplikasi pemrosesan menggunakan *Distributed Lock*, dan menyimpan payload mentah ke dalam kolom `JSONB`.
2. **FR-02 — Real-Time Telemetry Stream Processing:** Memproses array titik koordinat/split aktivitas secara asinkron untuk menghitung distribusi *Pace Zones*, zona detak jantung (*HR Zones*), dan mendeteksi *Personal Record* (PR) baru.
3. **FR-03 — Injury Risk & Consistency Analytics (SQL Engine):** Menghitung **ACWR (*Acute-to-Chronic Workload Ratio*)** berbasis jendela waktu 28 hari dan **Training Streak** (konsistensi hari latihan) langsung di level database menggunakan *Advanced Native SQL*.
4. **FR-04 — Spatial Route & GAP (Grade Adjusted Pace) Calculator:** Menghitung estimasi waktu tempuh (*ETA*) rute berdasarkan jarak segmen dan kemiringan elevasi (*slope*) menggunakan fungsi spasial **PostGIS**.
5. **FR-05 — Geospatial Route Discovery:** Menyediakan pencarian rute lari publik berdasarkan radius lokasi (`geo_distance`), rentang jarak, dan elevasi menggunakan **Elasticsearch**.
6. **FR-06 — AI Running Coach Orchestration:** Menghasilkan jadwal latihan mingguan berdasarkan target lomba dan ringkasan metrik pelari menggunakan pola *Strategy/Adapter* berbasis **Spring IoC** (mendukung mode *Live LLM* maupun *Deterministic Mock* tanpa API Key eksternal).

### B. Non-Functional Requirements (NFR) & Target Rubrik
* **Spring IoC:** *Map/List Injection*, `@Qualifier`, `@ConditionalOnProperty`, dan *Spring AOP* (`@Aspect`) untuk *observability/rate-quota*.
* **Java Stream API:** *Nested stream flattening* (`flatMap`), multi-level `groupingBy`, `Collectors.teeing()`, dan *custom* `ForkJoinPool` untuk *batch processing*.
* **Advanced Native SQL:** *Common Table Expressions* (`WITH`), *Window Functions* (`RANGE BETWEEN INTERVAL`, `LAG`, `DENSE_RANK`), *Gaps and Islands*, serta *PostGIS Spatial Lateral Joins*.
* **Containerization & Microservices:** Pemisahan domain menjadi 2 *microservices* mandiri dengan skema database terisolasi (`Schema-per-Service`), *multi-stage Dockerfile*, dan `docker-compose.yml` dengan batas memori (*resource limits*) serta `healthcheck`.
* **Message Broker & Stream:** **RabbitMQ** dengan *Topic Exchange*, *Routing Keys*, *Consumer Retry*, dan *Dead Letter Queue (DLQ)*.
* **Caching & Data Grid:** **Redis + Redisson** untuk *Cache-Aside* (dengan *jitter TTL*), *Cache Eviction* berbasis *event*, dan *Distributed Lock* (`RLock`).
* **Non-Relational & Search:** **Elasticsearch** (`geo_point` & `nested` query) serta **PostgreSQL `JSONB`** dengan `GIN Index`.

---

## 2. Arsitektur Microservices & Topologi Kontainer

Sistem terdiri dari **6 kontainer ringan** (total estimasi konsumsi RAM ~4 GB) yang berjalan di atas satu jaringan internal Docker (`larikuy-net`):

```text
+-----------------------------------------------------------------------------------+
|                        CLIENT (Swagger UI / Postman / Minimal UI)                 |
+-----------------------------------------+-----------------------------------------+
                                          | HTTP / REST
                    +---------------------+---------------------+
                    v                                           v
+-----------------------------------+       +---------------------------------------+
|   core-activity-service (:8081)   |       |   coach-discovery-service (:8082)     |
|-----------------------------------|       |---------------------------------------|
| - Auth & AES-256-GCM Token Crypto |       | - RabbitMQ Stream Consumer + DLQ      |
| - Strava & Simulator Webhook      |       | - Advanced Native SQL (ACWR & Streak) |
| - Redisson Distributed Lock       |       | - Redis Cache-Aside & Eviction        |
| - Java Stream Telemetry Engine    |       | - PostGIS GAP ETA Calculator          |
| - RabbitMQ Event Publisher        |       | - Elasticsearch Geo-Route Search      |
+--------+----------------+---------+       | - Spring IoC AI Coach & AOP Aspect    |
         |                |                 +---+---------------+---------------+---+
         | JDBC           | AMQP                | AMQP          | JDBC          | REST
         v                v                     v               v               v
+----------------+  +---------------------------+      +----------------+  +--------+
| PostgreSQL 16  |  |        RabbitMQ 3         |      |    Redis 7     |  |Elastic |
|   + PostGIS    |  | (Topic Exchange & DLQ)    |      | (Cache & Lock) |  |Search 8|
| (2 Schemas)    |  +---------------------------+      +----------------+  +--------+
+----------------+