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
`<svc>-security` does not redeclare `<svc>-entity` (it arrives via
`<svc>-service`). The single exception is `<svc>-api`, which declares all four
siblings (`service`, `entity`, `security`, `shared`) explicitly even though
`entity` and `shared` also arrive transitively — the API module uses types
from each directly (entities in mappers, DTOs in controllers), and declaring
what it uses keeps its `pom.xml` self-documenting about what it assembles.
(`<svc>-service` declaring `<svc>-shared` is NOT redundant — `entity` does not
depend on `shared`, and the service layer serializes event payloads from it.)

## Packaging plugin

Only `<svc>-api` applies the `spring-boot-maven-plugin` `repackage` execution
(`platform-parent` pins the plugin's version but binds no execution — see
`platform-parent/pom.xml`). No other module produces an executable jar.

**The repackage execution MUST set `<classifier>exec</classifier>`.** Without
it, `repackage` replaces `target/<finalName>.jar` in place with the executable
(nested `BOOT-INF/classes/**`) jar — but Failsafe substitutes that same
artifact path for `target/classes` when it builds the forked test JVM's
classpath, and a plain classloader cannot see classes nested inside
`BOOT-INF/`. Any `@SpringBootTest(classes = ...Application.class)`
integration test then fails to bootstrap with `IllegalStateException: Failed
to find merged annotation for @BootstrapWith(...)` — Spring silently can't
resolve the application class literal and drops `@SpringBootTest`'s
meta-annotations rather than erroring clearly. With the classifier,
`target/app.jar` stays the plain, classes-mirroring primary artifact (safe
for Failsafe) and the runnable jar is attached separately as
`target/app-exec.jar` — **that's the one to run** (`java -jar
identity-api/target/app-exec.jar`, not `app.jar`).

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
          <configuration>
            <classifier>exec</classifier>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

## Method-security parameter names

If a service uses `@PreAuthorize` with a named-parameter SpEL reference (e.g.
`@PreAuthorize("@orgAuth.hasRole(#orgId, 'ADMIN')")`), `platform-parent`'s
compiler plugin already sets `<parameters>true</parameters>` (required since
Spring Framework 6.1 removed bytecode-debug-info-based parameter name
discovery — without it, `#orgId` silently resolves to `null` and every such
check fails closed for every caller, with no error at compile or boot time).
No per-service action needed; this is documented here because the failure
mode is silent and easy to mistake for "the annotation just doesn't work
here" during debugging — write an end-to-end test asserting a legitimately
privileged caller *succeeds*, not only that an under-privileged one is
denied, or a fail-closed bug like this reads identical to a passing test
suite.

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
