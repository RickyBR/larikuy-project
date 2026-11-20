sequenceDiagram
    autonumber
    actor Client as Strava / Simulator
    participant S1 as core-activity-service
    participant Redis as Redis (Redisson)
    participant PG as PostgreSQL (activity_db)
    participant RMQ as RabbitMQ (TopicExchange)
    participant S2 as coach-discovery-service

    Client->>S1: POST /api/v1/webhooks/strava (Activity JSON)
    S1->>Redis: tryLock("lock:activity:{externalId}", 5s, 15s)
    alt Lock Gagal (Duplicate Request)
        S1-->>Client: 202 Accepted (Ignored - Duplicate)
    else Lock Berhasil
        S1->>PG: INSERT raw_payload (JSONB) & base metadata
        S1->>S1: Java Stream: flatMap splits, hitung HR/Pace Zones & PR
        S1->>RMQ: Publish "activity.synced" (ActivityEnrichedEvent)
        S1->>Redis: unlock()
        S1-->>Client: 201 Created (Processed Summary)
    end

    RMQ->>S2: Consume from "analytics.update.queue"
    alt Proses Sukses
        S2->>PG: Upsert daily_load (coach_db)
        S2->>Redis: @CacheEvict("runner:metrics:{userId}")
        S2->>RMQ: BasicAck
    else Proses Gagal (3x Retry Exhausted)
        RMQ->>RMQ: Route to "activity.dlq" (Dead Letter Queue)
    end





    flowchart LR
    Req([POST /api/v1/coach/plan]) --> AOP[Spring AOP: Cek Kuota Redis]
    AOP --> Cache{Cek Redis Cache\nrunner:metrics:userId}
    
    Cache -->|Cache Hit| Metrics[Ambil Metrik ACWR & Streak]
    Cache -->|Cache Miss| SQL[Eksekusi Advanced Native SQL\n28-Day ACWR & Gaps-Islands]
    SQL --> SaveCache[Simpan ke Redis\nTTL 1 Jam + Random Jitter]
    SaveCache --> Metrics

    Metrics --> IoC[Spring IoC Map Lookup\napp.llm.provider = MOCK / GROQ]
    IoC --> Plan[Generate Periodized Training JSON]
    Plan --> Res([200 OK: Training Schedule])





    flowchart LR
    Input([POST /api/v1/routes]) --> PostGIS[PostGIS Native SQL:\nST_LineInterpolatePoints & Slope]
    PostGIS --> GAP[Spring IoC Strategy:\nMinettiGapStrategy ETA]
    GAP --> Masking[PostGIS Privacy Masking:\nTrim 500m Start/End Point]
    Masking --> SavePG[(Simpan ke PostgreSQL coach_db)]
    SavePG --> PubRMQ[Publish "route.published" ke RabbitMQ]
    PubRMQ --> IndexES[Consumer Index ke Elasticsearch\ngeo_point + elevation metadata]