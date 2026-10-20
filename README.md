# veterinary-booking

Spring Boot web app for veterinary clinic booking. Currently a skeleton: one Thymeleaf page at `/hello` and a passing context-load test.

## Requirements

| Tool | Version | Notes |
| --- | --- | --- |
| JDK | 21 (Temurin) | `maven.compiler.release=21` |
| Maven | 3.10.0 | **Not required** — use the bundled `./mvnw` wrapper |

No global Maven install is needed. The wrapper is `distributionType=only-script`, so it downloads Maven 3.10.0 into `~/.m2/wrapper/dists/` on first use. `maven-wrapper.jar` is gitignored.

Check your JDK:

```bash
java -version   # expect 21.x
```

## Setup

```bash
git clone git@github.com:wenanoshe/veterinary-booking.git
cd veterinary-booking
./mvnw -v      # first run downloads Maven + dependencies
```

## Run

```bash
./mvnw spring-boot:run
```

Then open <http://localhost:8080/hello>.

| Path | Result |
| --- | --- |
| `/hello` | 200 — renders `templates/hello.html` with `mensaje=Funciona` |
| `/h2-console` | 302 → `/h2-console/` — H2 web console (dev mode only, see below) |
| `/` | 404 — no index page yet |

Change the port:

```bash
./mvnw spring-boot:run -Dspring-boot.run.arguments=--server.port=9090
```

Package and run standalone (DevTools is excluded from the packaged jar):

```bash
./mvnw package
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

## Test

```bash
./mvnw test                                     # all tests
./mvnw test -Dtest=DemoApplicationTests        # one class
./mvnw test -Dtest=DemoApplicationTests#contextLoads   # one method
```

## Quality gate

Java has no separate typecheck step — `javac` *is* the type checker, and it runs as part of the build. Format, lint, and bug detection are separate concerns, and this project wires all three into one command:

```bash
./mvnw verify      # THE gate: compile + test + format + lint + bug scan
```

`./mvnw test` runs only the tests. The three quality plugins are bound to the `verify` phase, so use `verify` before pushing.

| Check | Tool | Run alone with | Fix with |
| --- | --- | --- | --- |
| Format | Spotless 3.10.4 (palantir-java-format 2.102.0) | `./mvnw spotless:check` | `./mvnw spotless:apply` |
| Lint | Checkstyle 10.26.1 | `./mvnw checkstyle:check` | fix by hand |
| Bug patterns | SpotBugs 4.10.5.0 | `./mvnw spotbugs:check` | fix by hand |
| Toolchain | maven-enforcer-plugin | `./mvnw enforcer:enforce` | fix by hand |

Configuration lives in:

- `config/checkstyle/checkstyle.xml` — curated ruleset
- `config/spotbugs/exclude.xml` — suppressed bug patterns
- `pom.xml` — plugin versions and Spotless steps

The enforcer requires JDK ≥ 21, Maven ≥ 3.9, and no duplicate dependency versions.

### Conventions this setup enforces

- **4 spaces, no tabs.** The Spring Initializr default was tabs; `spotless:apply` converted the codebase.
- **No wildcard imports**, sorted imports, no unused imports, trailing newline.
- **Braces on every `if`/`for`/`while`.**
- Naming conventions for types, methods, constants, and parameters.

### Notes on the ruleset

- The Checkstyle ruleset **deliberately omits formatting modules** (`Indentation`, `LineLength`, `WhitespaceAround`, …). Those belong to Spotless. Running both over the same concern means the formatter moves the code and the linter complains, forever.
- `HideUtilityClassConstructor` false-positives on Spring beans, so `DemoApplication` carries `@SuppressWarnings("checkstyle:hideutilityclassconstructor")`. Do the same for any `@Component`/`@Configuration` class that only has static members.
- `MissingJavadocType` is a **warning**, not an error. The build stays green, but public types without Javadoc get reported. Promote it to `error` once the codebase is documented.
- SpotBugs excludes `EI_EXPOSE_REP` / `EI_EXPOSE_REP2`, which fire on every Spring entity or DTO getter where returning the internal reference is the intended contract.
- The JDK 21 warnings about a dynamically loaded Java agent and Mockito self-attaching are expected and harmless.

There is no CI workflow yet — `verify` only runs locally.

## Project layout

```
src/main/java/com/example/demo/
  DemoApplication.java          @SpringBootApplication entrypoint
  HelloController.java          @GetMapping("/hello") → Thymeleaf view "hello"
src/main/resources/
  application.properties        spring.application.name=demo
  templates/hello.html          the only template
src/test/java/com/example/demo/
  DemoApplicationTests.java     @SpringBootTest contextLoads
config/
  checkstyle/checkstyle.xml     lint ruleset
  spotbugs/exclude.xml          suppressed bug patterns
```

## Dependencies

Declared in `pom.xml`, inherited from `spring-boot-starter-parent:4.1.1`.

| Dependency | Scope | Why |
| --- | --- | --- |
| `spring-boot-starter-webmvc` | compile | Spring MVC + embedded Tomcat 11 |
| `spring-boot-starter-thymeleaf` | compile | Server-side view templates |
| `spring-boot-starter-jackson` | compile (transitive) | JSON — **Jackson 3** (`tools.jackson.databind.*`) |
| `spring-boot-h2console` | compile | H2 web console at `/h2-console` |
| `com.h2database:h2` | runtime | In-memory database driver |
| `spring-boot-devtools` | runtime, optional | Live restart; dev classpath only |
| `spring-boot-starter-webmvc-test` | test | JUnit 6, Mockito, AssertJ, MockMvc, JSONAssert, Awaitility |

Resolved versions: Spring Framework 7.0.9, Tomcat 11.0.24, Jackson 3.1.5, H2 2.4.240, JUnit Jupiter 6.0.3, Mockito 5.23.0, AssertJ 3.27.7.

## Notes and gotchas

- **Spring Boot 4 renamed starters.** Use `spring-boot-starter-webmvc`, not the deprecated `spring-boot-starter-web`. Test starters are per-module slices (`spring-boot-starter-data-jpa-test`, `spring-boot-starter-jdbc-test`, …) and each pulls `spring-boot-starter-test` transitively.
- **Jackson 3.** The `com.fasterxml.jackson.databind` package is not on the classpath; import `tools.jackson.databind.*`.
- **The H2 console is enabled by DevTools, not by config.** The `spring-boot-h2console` jar ships `META-INF/spring-devtools.properties` with `defaults.spring.h2.console.enabled=true`. If DevTools is ever removed, set `spring.h2.console.enabled=true` explicitly or the console returns 404.
- **There is no persistence layer yet.** H2 is on the classpath, but without a JDBC/JPA starter Spring creates no `DataSource`, so the H2 console has nothing to connect to. Adding entities means adding `spring-boot-starter-data-jpa` and an explicit `spring.datasource.url`.
- **Line endings** are pinned by `.gitattributes`: `/mvnw` is LF, `*.cmd` is CRLF.
- `HELP.md` and `.idea/` are gitignored. `HELP.md` is generic Spring Initializr boilerplate, not project documentation.

## Next steps

- [ ] Pick the persistence stack (`spring-boot-starter-data-jpa` + H2/PostgreSQL) and configure a `DataSource`.
- [ ] Add an index page so `/` resolves instead of 404.
- [ ] Model the domain (owners, pets, appointments) and add the booking flow.
- [ ] Add a CI pipeline that runs `./mvnw verify` — there is none today.

## References

- [Spring Boot 4.1.1 reference](https://docs.spring.io/spring-boot/4.1.1/reference/)
- [Spring Boot Maven plugin guide](https://docs.spring.io/spring-boot/4.1.1/maven-plugin)
- [Thymeleaf](https://docs.spring.io/spring-boot/4.1.1/reference/web/servlet.html#web.servlet.spring-mvc.template-engines)
