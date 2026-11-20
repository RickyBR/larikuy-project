larikuy-backend/
├── docker-compose.yml                  # Orkestrasi 6 kontainer + memory limits + healthchecks
├── .env.example                        # Konfigurasi environment variables
├── core-activity-service/              # Microservice 1 (Port 8081)
│   ├── Dockerfile                      # Multi-stage build (Maven -> Temurin 21 JRE Alpine)
│   └── src/main/
│       ├── java/com/larikuy/activity/
│       │   ├── config/                 # RabbitMQ Producer, Redisson, & Security Config
│       │   ├── controller/             # ActivityController & SimulatorWebhookController
│       │   ├── domain/                 # Entity & JSONB Converters
│       │   ├── service/                # IngestionService (RLock) & TelemetryStreamEngine
│       │   └── messaging/              # ActivityEventPublisher
│       └── resources/db/migration/     # Flyway SQL scripts (schema: activity_db)
│
├── coach-discovery-service/            # Microservice 2 (Port 8082)
│   ├── Dockerfile                      # Multi-stage build
│   └── src/main/
│       ├── java/com/larikuy/coach/
│       │   ├── aspect/                 # Spring AOP (@AuditLlmQuota)
│       │   ├── config/                 # RabbitMQ Consumer+DLQ, Redis Cache, Elastic Config
│       │   ├── controller/             # AnalyticsController, RouteController, CoachController
│       │   ├── messaging/              # ActivityStreamConsumer
│       │   ├── repository/             # AdvancedNativeSqlRepository (ACWR, Streak, PostGIS)
│       │   ├── search/                 # RouteElasticRepository & GeoSearchService
│       │   └── strategy/               # Spring IoC Interfaces & Beans (LLM & GAP Pace)
│       └── resources/db/migration/     # Flyway SQL scripts + Seed Data (schema: coach_db)
│
└── frontend-minimal/                   # Minimalist Next.js Dashboard (Opsional / Seadanya)