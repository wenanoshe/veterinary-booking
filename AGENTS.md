# AGENTS.md

## Stack (verified against `pom.xml` / `dependency:tree`)

Spring Boot **4.1.1** · Spring Framework **7.0.9** · Java **21** (`maven.compiler.release=21`) · embedded Tomcat 11 · Thymeleaf views (no REST layer yet).
Base package `com.example.demo`, artifactId `demo`, `spring.application.name=demo`.

## Commands — always `./mvnw`

`mvn` is **not installed** on this machine. The wrapper is `distributionType=only-script` (downloads Maven 3.10.0 on first use; `maven-wrapper.jar` is gitignored).

| Task | Command |
| --- | --- |
| Run (dev, port 8080) | `./mvnw spring-boot:run` |
| Run on another port | `./mvnw spring-boot:run -Dspring-boot.run.arguments=--server.port=9090` |
| **Full gate** | `./mvnw verify` → compile + tests + format + lint + bug scan |
| Test all | `./mvnw test` |
| One test class | `./mvnw test -Dtest=DemoApplicationTests` |
| One test method | `./mvnw test -Dtest=DemoApplicationTests#contextLoads` |
| Fix formatting | `./mvnw spotless:apply` |
| Package | `./mvnw package` → fat jar `target/demo-0.0.1-SNAPSHOT.jar` |
| Run packaged | `java -jar target/demo-0.0.1-SNAPSHOT.jar` |
| Clean | `./mvnw clean` |

**`./mvnw verify` is the gate, not `./mvnw test`.** `test` alone skips all three quality plugins, because they are bound to the `verify` phase.

There is still **no CI config** in this repo (no `.github/`), so `verify` only runs locally unless you wire it up.

## Boot 4 gotchas — these will bite you

1. **Jackson 3, not 2.** JSON uses `tools.jackson.databind.*`. `com.fasterxml.jackson.databind.ObjectMapper` is not on the classpath and will not compile.
2. **Starter names changed.** `spring-boot-starter-webmvc` (not `-web`, which is deprecated) and `spring-boot-starter-webmvc-test` (not `-test`). Test starters are per-module slices: `spring-boot-starter-data-jpa-test` for `@DataJpaTest`, `spring-boot-starter-jdbc-test` for `@JdbcTest`. Every `*-test` starter pulls `spring-boot-starter-test` transitively (JUnit, Mockito, AssertJ, Hamcrest, JSONAssert, Awaitility, XMLUnit, JsonPath, spring-test/MockMvc).
3. **JUnit 6** (`junit-jupiter` 6.0.3) — JUnit 5 API shape, 6.x coordinates. `junit-jupiter-params` is already present.
4. **The H2 console works only because devtools is on the classpath.** The `spring-boot-h2console` jar ships `META-INF/spring-devtools.properties` → `defaults.spring.h2.console.enabled=true`. Without devtools the console 404s unless you set `spring.h2.console.enabled=true` yourself. URL is `/h2-console` (302 → `/h2-console/`).
5. **There is no persistence layer yet.** `com.h2database:h2` is `runtime` scope but no JDBC/JPA starter exists, so Spring creates **no DataSource** and the H2 console has nothing to connect to. Adding entities requires `spring-boot-starter-data-jpa` plus an explicit `spring.datasource.url` for H2.
6. Mockito "self-attaching" / "Java agent loaded dynamically" warnings on JDK 21 are harmless noise from `mockito-core` 5.23 — not a failure.
7. DevTools is `optional` + `runtime` scope: active under `spring-boot:run` and the IDE, absent from the packaged jar. Its restart thread shows up as `restartedMain` in logs.

## Quality gates — `./mvnw verify`

There is no separate "typecheck" step in Java: **`javac` is the type checker**, and `verify` runs it. The three plugins below cover format, lint, and bug patterns.

| Plugin | Goal | Config | Fix by |
| --- | --- | --- | --- |
| `spotless-maven-plugin` 3.10.4 | Formatter (palantir-java-format) | `pom.xml` | `./mvnw spotless:apply` |
| `maven-checkstyle-plugin` 3.6.0 | Lint (style + correctness) | `config/checkstyle/checkstyle.xml` | fix by hand |
| `spotbugs-maven-plugin` 4.10.5.0 | Bug patterns (NPE, leaks) | `config/spotbugs/exclude.xml` | fix by hand |

Plus `maven-enforcer-plugin` (version inherited from the Boot parent) at `validate`: requires JDK ≥ 21, Maven ≥ 3.9, no duplicate dependency versions.

Rules that will bite you here:

- **Do not add formatting rules to the Checkstyle ruleset.** `Indentation`, `LineLength`, `WhitespaceAround` and friends are owned by Spotless. Enabling both means the formatter moves the code, Checkstyle complains, forever. The ruleset is deliberately curated to exclude them.
- **`HideUtilityClassConstructor` false-positives on Spring beans.** `DemoApplication` carries `@SuppressWarnings("checkstyle:hideutilityclassconstructor")`; do the same on any `@Component`/`@Configuration` class with only static members. The `SuppressWarningsHolder` + `SuppressWarningsFilter` pair in the ruleset is what makes that work.
- **Checkstyle modules belong under `TreeWalker`, not `Checker`.** Putting a `TreeWalker` rule directly under `Checker` fails the build with "not allowed as a child in Checker".
- **`MissingJavadocType` is a warning, not an error** — the build stays green, but public types without Javadoc are reported. Promote it to `error` once the codebase is documented.
- **SpotBugs excludes `EI_EXPOSE_REP`/`EI_EXPOSE_REP2`** because they fire on every Spring entity/DTO getter. Add to `config/spotbugs/exclude.xml` rather than silencing individual classes.

## Current state — read before extending

- `DemoApplication` — `@SpringBootApplication` entrypoint.
- `HelloController` — `@GetMapping("/hello")` → Thymeleaf view `hello`, model attribute `mensaje`.
- `src/main/resources/templates/hello.html` — the only template.
- `application.properties` — a single line: `spring.application.name=demo`.
- `/` returns **404** (no index page).
- Anything under `com.example.demo.*` is picked up by component scan automatically; the app name and jar name both derive from `demo`, so renaming the base package means updating `DemoApplication`'s package declaration, the test package, and `spring.application.name`.

## Conventions observed

- Formatting is **Spotless/palantir style: 4 spaces**, no tabs, imports sorted and grouped. The Spring Initializr default was tabs; `spotless:apply` converted the codebase on 2026-10-19. Do not hand-format against it — run `./mvnw spotless:apply`.
- Java identifiers are English; the one existing controller uses **Spanish** for user-facing copy and model attribute names (`mensaje`, "Funciona", "Hola"). Mirror that for user-facing text here, keep identifiers and comments in English.
- Commits are Conventional Commits with a scope, e.g. `core: Initial commit`.
- `.gitattributes` pins `/mvnw` to LF and `*.cmd` to CRLF — do not "normalize" line endings.
- `HELP.md` (generic Spring Initializr text) and `.idea/` are gitignored on purpose.
