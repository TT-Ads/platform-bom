# AdPilot Service Template

This document is the canonical layout for a new AdPilot service. Stamp a new
service (`<svc>`) by copying this structure and swapping the `<svc>` token —
do not invent a different shape per service.

## Module layout

Every service is a 5-module Maven reactor, parented by `platform-parent`.
Modules **must** be listed in the parent's `<modules>` in this exact
(dependency) order:

```xml
<modules>
  <module><svc>-shared</module>
  <module><svc>-entity</module>
  <module><svc>-service</module>
  <module><svc>-security</module>
  <module><svc>-api</module>
</modules>
```

## Dependency graph

```
<svc>-shared   (no Spring/JPA deps — DTOs + event payloads only)
     ^
<svc>-entity   (JPA entities; depends on shared)
     ^
<svc>-service  (services, repositories, exceptions; depends on entity — and
                 transitively shared)
     ^
<svc>-security (JWT / resource-server mechanisms; depends on service — and
                 transitively entity, shared)
     ^
<svc>-api      (Boot application; depends explicitly on service + entity +
                 security — this trio is the one deliberate exception to the
                 no-redundant-sibling-deps rule below)
```

Rule: **do not redeclare a dependency you already get transitively.**
`<svc>-security` does not redeclare `<svc>-entity`; `<svc>-service` does not
redeclare `<svc>-shared`. The single exception is `<svc>-api`, which declares
its three direct dependencies (`service`, `entity`, `security`) explicitly
even though `entity` and `shared` also arrive transitively — this keeps the
API module's own `pom.xml` self-documenting about what it assembles.

## Packaging plugin

Only `<svc>-api` applies the `spring-boot-maven-plugin` `repackage` execution
(`platform-parent` pins the plugin's version but binds no execution — see
`platform-parent/pom.xml`). No other module produces an executable jar.

```xml
<build>
  <finalName>app</finalName>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <executions>
        <execution>
          <goals><goal>repackage</goal></goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

## Packages

Base package: `io.github.ttads.<svc>`. Sub-packages:

```
io.github.ttads.<svc>.api            # <svc>-api: Boot app, SecurityConfig, OpenApiConfig
io.github.ttads.<svc>.api.v1.controllers
io.github.ttads.<svc>.api.v1.mappers # MapStruct mappers only, no hand-mapping
io.github.ttads.<svc>.services       # <svc>-service
io.github.ttads.<svc>.repositories   # <svc>-service
io.github.ttads.<svc>.exceptions     # <svc>-service
io.github.ttads.<svc>.entities       # <svc>-entity
io.github.ttads.<svc>.shared         # <svc>-shared: DTOs + event payloads
io.github.ttads.<svc>.security       # <svc>-security
```

## Entity conventions

Every JPA entity in `<svc>-entity` follows this annotation stack, no
exceptions without a documented reason:

```java
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@Builder
@Entity
@Table(name = "…")
public class Widget {

    @Id
    @UuidGenerator(style = UuidGenerator.Style.TIME)
    private UUID id;

    @Enumerated(EnumType.STRING)
    private WidgetStatus status;

    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, Object> metadata; // JSONB column

    @CreationTimestamp
    private Instant createdAt;

    @UpdateTimestamp
    private Instant updatedAt;

    @Version
    private Long version; // only on aggregates a user can mutate concurrently
}
```

- IDs: `UUID`, generated with `@UuidGenerator(style = UuidGenerator.Style.TIME)`
  (time-ordered, index-friendly).
- Enums: always `@Enumerated(EnumType.STRING)` — never ORDINAL.
- Semi-structured data: `@JdbcTypeCode(SqlTypes.JSON)` mapped to a `jsonb`
  column.
- Auditing: every table has `created_at` / `updated_at`, populated via
  Hibernate's `@CreationTimestamp` / `@UpdateTimestamp` (or a shared
  `@EntityListeners(AuditingEntityListener.class)` base, per service
  convention).
- Optimistic locking: `@Version` only on aggregates that users can mutate
  concurrently (e.g. an `Organization` or `AppUser`, not an immutable event
  record).

## API conventions

- Every endpoint lives under `/api/v1/...`.
- Controllers: `io.github.ttads.<svc>.api.v1.controllers`.
- DTO <-> entity mapping: MapStruct only (`io.github.ttads.<svc>.api.v1.mappers`),
  never hand-written mapping code.
- Errors: a single `GlobalExceptionHandler` per service returns RFC 7807
  `application/problem+json` bodies.
- Docs: springdoc-openapi is wired per service (`<svc>-api`), not shared
  centrally — each service publishes its own OpenAPI document.

## Persistence

- Flyway migrations live per-service under `<svc>-api/src/main/resources/db/migration`
  (or wherever the service's Boot app owns its `DataSource`).
- Row-Level Security (RLS) policies are applied on every tenant-scoped table.
- Where a service produces domain events, it uses the transactional outbox
  pattern (an `outbox_event` table written in the same transaction as the
  business change, drained by a poller) rather than dual-writing to the
  database and a broker.

## Testing

- JUnit 5 only, everywhere — no JUnit 4, no vintage engine.
- Unit tests (`*Test.java`) run under Surefire (`mvn test`).
- Integration tests (`*IT.java`) run under Failsafe (`mvn verify`), backed by
  Testcontainers (Postgres, Keycloak, etc. as needed).
- `@Disabled` is never committed without a comment linking the tracking
  issue/reason.

## Versioning policy

- Breaking changes to a published API are introduced as `/api/v2/...`
  alongside the existing `/api/v1/...` — the old version is never mutated
  in place.
- The deprecated version responds with a `Deprecation` header and a
  documented sunset date before removal.
