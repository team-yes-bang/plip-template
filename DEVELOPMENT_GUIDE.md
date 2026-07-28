# PLIP Backend Service Development Guide

This document defines common architecture and development rules for PLIP microservice developers.
For **post-clone template setup** (rename, ports, env), see [START.md](START.md).
For **AI agents**, see [AI_CODING_GUIDELINES.md](AI_CODING_GUIDELINES.md).

Eureka and API Gateway integration procedures are out of scope for this guide (initial YAML toggles remain in START.md only).

---

## 1. Reference sample code (`src/test/java/com/sample`)

- Code under `src/test/java/com/sample/**` (e.g. `auth_service`, `chatting`) is **reference-only**. It is **not** part of the running service.
- `build.gradle` `sourceSets` excludes `**/com/sample/**` from compile and `./gradlew test`.
- Use it only for **structure / layering ideas**. Write real code under `src/main/java/com/plip/{service}`.
- Sample naming (`adaptor`, `vo`, `mysql`/`mongodb`) is **legacy**. Follow the team standard in section 2; do **not** copy sample package spelling or `com.unionclass.*` imports.

### Sample vs team standard

| Area | Team standard (new code) | Sample (`com.sample`) |
|------|--------------------------|------------------------|
| Adapter folder | `adapter` | `adaptor` |
| Web models | `adapter.in.web.dto` | `adaptor.in.web.vo` |
| Persistence package | `adapter.out.persistence` | `adaptor.out.mysql` / `mongodb` |
| Outbound adapter class | `*PersistenceAdapter` | `*RepositoryAdapter` etc. |
| Inbound port / impl | `*UseCase` / `*Service` | same |
| Domain | `domain.model` (no JPA) | same intent |
| Root package | `com.plip.{service}` | `com.sample.*` (stale imports may remain) |

---

## 2. Architecture standard (Hexagonal)

Dependencies must point inward: adapters -> application -> domain.

```text
src/main/java/com/plip/{service}/
├── adapter/
│   ├── in/web/               # Controller, Request/Response DTOs
│   └── out/persistence/      # PersistenceAdapter, JPA Entity, Spring Data Repository, mappers
├── application/
│   ├── port/in/              # UseCase interfaces (+ port DTOs if needed)
│   ├── port/out/             # Outbound ports
│   └── service/              # UseCase implementations
├── domain/
│   └── model/                # Pure domain (no JPA / Spring annotations)
└── global/config/            # SwaggerConfig, shared config
```

Rules:

1. `domain.model` must not import JPA, Spring, or infrastructure libraries.
2. Do not expose JPA entities outside `adapter`; map to domain before returning to `application`.
3. Package names must not contain hyphens; use `com.plip.agit` style.
4. Naming: `*UseCase`, `*Service`, `*Port` / `*PersistencePort`, `*PersistenceAdapter`.

---

## 3. API and documentation

### 3.1 REST (Springdoc OpenAPI)

- Code-first with Springdoc. Annotate controllers/DTOs (`@Tag`, `@Operation`, `@Schema`).
- Run `./gradlew test` so `OpenApiGeneratorTest` refreshes `docs/openapi.yaml`.
- Commit `docs/openapi.yaml` whenever the API surface changes.

### 3.2 Kafka / EDA (when needed)

- Document events under `docs/events/{event-name}.v1.md`.

---

## 4. DB and entity standards

Each microservice owns one DB (`plip_{servicename}`). No cross-service joins.

1. Tables/columns: lowercase `snake_case`.
2. PK: `id` (`BIGINT`, auto-increment). Cross-service refs: `{service}_id`.
3. Prefer a shared timestamp base (`created_at`, `updated_at`) written by the developer.
4. Associations: `FetchType.LAZY` for `@ManyToOne` / `@OneToOne`.
5. Keep DDL in `docs/sql/schema.sql` as tables change.
6. Entities: `@Getter` + `@NoArgsConstructor(access = AccessLevel.PROTECTED)`; do not use `@Data` on entities.

Add libraries (Security, Redis, etc.) in `build.gradle` only when the feature needs them — see START.md step 6.
