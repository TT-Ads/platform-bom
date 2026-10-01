# platform-bom

Shared Maven build-of-materials for the AdPilot platform. A small multi-module reactor with three published
artifacts, all released together under one version:

- **`platform-bom`** — the reactor root.
- **`platform-dependencies`** — a pure BOM. `dependencyManagement` only, no plugins. It imports the Spring Boot,
  Spring Cloud and Testcontainers BOMs and pins the libraries every service needs (springdoc MVC and WebFlux UI,
  Lombok, MapStruct, Keycloak admin client, WireMock standalone, jqwik, ...), each version as one property.
- **`platform-parent`** — the parent POM every service's root `pom.xml` inherits from. It imports
  `platform-dependencies` and centralises plugin configuration (compiler and annotation processors,
  Surefire/Failsafe, JaCoCo, the `spring-boot-maven-plugin` pin), so services stay thin.

`docs/service-template.md` is the canonical 5-module service layout (`<svc>-shared` / `-entity` / `-service` /
`-security` / `-api`), including release publishing and the Dependabot config.

## Versions

Versions are `0.<series>.<n>` (for example `0.1.3`). `.platform-series` holds the `MAJOR.MINOR` pair (currently
`0.1`), and **every push to `main` that changes more than docs publishes `<series>.<last+1>`** to this
repository's GitHub Packages Maven registry and pushes the tag `v<version>` (`publish.yml`). Pushes that only
touch `docs/**`, `*.md`, `.gitattributes`, `.gitignore` or `.dockerignore` publish nothing.

- The tags are the record of what shipped: `git ls-remote --tags https://github.com/TT-Ads/platform-bom`
  (today `v0.1.0` to `v0.1.3`). The Packages page of this repository shows the latest.
- Publishing is serialised (`concurrency: publish-platform-bom`, no cancel). After three or more quick merges
  GitHub cancels the *pending* middle run; the last run contains those commits, so no change is lost, but a
  version number is skipped. That is expected.
- While the platform is at `0.x`, treat a series bump as breaking. `1.0.0` will start real semver with
  deprecation notice.

## Breaking changes

Edit `.platform-series` (`0.1` → `0.2`) in the same PR as the breaking change, and describe it in the PR title.
The first version of a new series is **`<series>.1`**, not `.0` (the counter is last+1, and a series with no tags
starts from 0). Services get a Dependabot PR for the new series and upgrade deliberately; its CI result is the
signal.

## Consuming

A service's root `pom.xml` inherits the parent and names the registry it comes from (a POM's own `<repositories>`
resolves its parent on Maven 3.9, so `settings.xml` needs only the credential):

```xml
<parent>
  <groupId>io.github.ttads</groupId>
  <artifactId>platform-parent</artifactId>
  <version>0.1.3</version>
  <relativePath/>
</parent>

<repositories>
  <repository>
    <id>github</id>
    <url>https://maven.pkg.github.com/tt-ads/platform-bom</url>
    <snapshots><enabled>false</enabled></snapshots>
  </repository>
</repositories>
```

See `docs/service-template.md` for the full layout, CI caller and Dockerfile.

## Upgrading

Merge the Dependabot PR that bumps `platform-parent` (each service has `.github/dependabot.yml`, see the
template). Then delete any local version pin the new BOM now manages (for example jqwik or WireMock versions);
`mvn dependency:tree -Dincludes=net.jqwik,org.wiremock` shows what resolves.

## Local token setup

The Maven registry needs a token even to read public packages (anonymous requests get `401`). You need a
classic personal access token with `read:packages`:

```bash
gh auth refresh -h github.com -s read:packages        # interactive, once
export GITHUB_ACTOR=<your-login> GITHUB_TOKEN="$(gh auth token)"
```

Add the matching `github` server to `~/.m2/settings.xml` (what each service's `scripts/dev-setup.sh` writes):

```xml
<server>
  <id>github</id>
  <username>${env.GITHUB_ACTOR}</username>
  <password>${env.GITHUB_TOKEN}</password>
</server>
```

Check it: this should print `200`.

```bash
curl -s -o /dev/null -w '%{http_code}\n' -u "$GITHUB_ACTOR:$GITHUB_TOKEN" \
  https://maven.pkg.github.com/tt-ads/platform-bom/io/github/ttads/platform-parent/0.1.3/platform-parent-0.1.3.pom
```

Builds a service publishes from CI use the caller's own `GITHUB_TOKEN`; no secret is needed there.

## Publishing internals

`.github/workflows/ci.yml` (pull requests) builds and installs the reactor with a throwaway revision and publishes
nothing. `.github/workflows/publish.yml` (push to `main`) computes the version from the tags, runs
`mvn -Drevision=<version> clean deploy` to this repository's registry, then tags `v<version>`. The POMs carry
`${revision}` and `flatten-maven-plugin` bakes the resolved version into each deployed POM.

- Deploying before tagging means: if the tag push fails after a successful deploy, a re-run computes the same
  `<n>` and fails with `409`. **Do not just re-run.** Push the missing tag by hand
  (`git tag -a v<version> -m "platform-bom <version>" <sha> && git push origin v<version>`), then re-run.
- The published `platform-parent` keeps its own `<revision>0.0.0-SNAPSHOT</revision>` property. A service that
  uses `${revision}` must declare its own `<revision>` (see the template's "Release publishing"), or it would
  silently build as `0.0.0-SNAPSHOT` without `-Drevision`.
- Keep the versions `platform-parent` re-declares for Lombok, MapStruct and the binding in sync with
  `platform-dependencies`; nothing checks it yet.
