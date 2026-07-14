# platform-bom

Shared Maven build-of-materials for the AdPilot platform.

This repository is a small multi-module reactor with two things every
AdPilot microservice depends on:

- **`platform-dependencies`** (`io.github.ttads:platform-dependencies:0.1.0`) —
  a pure BOM. `dependencyManagement` only, no plugins, no build section. It
  imports the Spring Boot, Spring Cloud, and Testcontainers BOMs and pins
  the handful of libraries every service needs (springdoc, Lombok,
  MapStruct, Keycloak admin client, etc.), each version expressed as a
  single property.
- **`platform-parent`** (`io.github.ttads:platform-parent:0.1.0`) — the
  parent POM every service's root `pom.xml` inherits from. It imports
  `platform-dependencies` and centralizes all plugin configuration
  (compiler + annotation processor ordering, Surefire/Failsafe test wiring,
  JaCoCo, the `spring-boot-maven-plugin` version pin) so individual
  services stay thin.

See `docs/service-template.md` for the canonical 5-module service layout
(`<svc>-shared` / `-entity` / `-service` / `-security` / `-api`) that every
AdPilot service is stamped from.

## How a service consumes this today

There is no published artifact yet (see **Publishing** below), so a
service builds against a local install:

```bash
git clone <platform-bom-repo-url>
cd platform-bom
mvn install
```

That puts `io.github.ttads:platform-dependencies:0.1.0` and
`io.github.ttads:platform-parent:0.1.0` into your local `~/.m2/repository`.
A service's root `pom.xml` then declares:

```xml
<parent>
  <groupId>io.github.ttads</groupId>
  <artifactId>platform-parent</artifactId>
  <version>0.1.0</version>
</parent>
```

Re-run `mvn install` here whenever `platform-bom` changes and you need the
update locally.

## Version policy

`platform-bom`, `platform-dependencies`, and `platform-parent` all move
together at `0.x` while the platform is still churning — expect breaking
changes between minor versions with no deprecation window during this
phase. Once the service surface stabilizes we'll cut `1.0.0` and start
following semver with real deprecation notice for breaking changes.

## Publishing (deferred)

Today, consuming this repo means `git clone` + `mvn install` into your
local repository, as above. Publishing `platform-dependencies` and
`platform-parent` to GitHub Packages (or another shared Maven registry) so
CI and other machines don't need a local clone is a deferred follow-up, not
yet wired up.
