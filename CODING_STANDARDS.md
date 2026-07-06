# Virtual Paradox Coding Standards

This document lists the coding rules enforced by `eu.virtualparadox:parent`.
The source of truth is `parent/pom.xml` plus the Checkstyle, PMD, SpotBugs,
Forbidden APIs, license, and OWASP configuration files under
`build-config/src/main/resources`.

## Baseline Requirements

- Projects compile against Java 21: `maven.compiler.release=21`.
- Maven 3.9.6 or newer is required.
- Consumer code should live under the `eu.virtualparadox...` package prefix if
  it is expected to be checked by NullAway, because the parent configures
  NullAway for that annotated package prefix.
- The gates check both `src/main/java/**/*.java` and
  `src/test/java/**/*.java`.

## Formatting And File-Level Rules

- Use Palantir Java Format. `spotless:check` verifies it, and
  `spotless:apply` applies it.
- Unused imports are not allowed; both Spotless and Checkstyle cover this.
- Tab characters are forbidden.
- Every file must end with a newline.
- Maximum line length is 140 characters.
- The line-length rule ignores `package`, `import`, and `http(s)://` lines.
- Wildcard imports are forbidden.
- Redundant imports and unnecessary fully qualified names are forbidden.
- Only one statement is allowed per line.
- Only one variable may be declared in a single declaration.
- Control-flow statements must use braces.
- Use uppercase `L` for long literals; lowercase `l` is forbidden.

## Naming

The standard Checkstyle Java naming rules apply:

- packages: lower-case Java package names;
- types, classes, interfaces, and enums: UpperCamelCase;
- methods, members, parameters, and local variables: lowerCamelCase;
- constants: ALL_CAPS_WITH_UNDERSCORES.

## Finality And Immutability

- Method parameters must be `final`.
- Local variables must be `final` where the Checkstyle rule applies.
- Fields should be `final` when PMD `ImmutableField` determines that mutability
  is unnecessary.
- A class with only private constructors must be `final`.
- `finalize()` is forbidden.
- Every class must declare an explicit constructor; PMD `AtLeastOneConstructor`
  can also apply to test classes.

## Javadoc

- Public and protected types require Javadoc.
- Public and protected methods require Javadoc.
- Javadoc must include `@param` tags for parameters and `@return` tags for
  return values.
- Unknown Javadoc tags are not allowed.
- Methods annotated with `@Override`, `@Test`, `@BeforeEach`, `@AfterEach`,
  `@BeforeAll`, or `@AfterAll` are exempt from the Checkstyle method Javadoc
  requirement.
- The Maven Javadoc gate runs with `doclint=all`, and every Javadoc warning
  fails the build.

## Null Safety And Return Values

- `return null;` is forbidden.
- For empty results, use immutable empty collections, `Optional`, or throw an
  explicit exception.
- PMD also forbids returning `null` from collection-returning methods
  (`ReturnEmptyCollectionRatherThanNull`).
- NullAway runs at `ERROR` level for `eu.virtualparadox` packages. Nullable
  contracts must be explicit with a nullable annotation; unannotated references
  should be treated as non-null contracts.
- Do not ignore return values that Error Prone treats as required
  (`ReturnValueIgnored`, `FutureReturnValueIgnored`).

## Control Flow And Complexity

- Each method should have exactly one `return` (`OnlyOneReturn`).
- Do not use `instanceof`; prefer polymorphism or an explicit, narrow dispatch
  boundary.
- Do not use switch `yield`; extract case logic into named methods.
- Switch fall-through is allowed only when documented in a form accepted by
  Checkstyle.
- Do not compare objects by reference equality; use `equals` or design a
  different identity model.
- Method cyclomatic complexity limit: 8.
- Class cyclomatic complexity limit: 40.
- NPath complexity limit: 120.
- PMD cognitive complexity and Law of Demeter rules are enabled.
- A class may have at most 12 fields.
- A class may have at most 18 methods.
- A parameter list becomes a violation at 6 parameters.
- Coupling threshold: 10.
- Do not reassign method parameters.
- Empty `catch` blocks are forbidden.
- Do not catch generic `Exception` without a narrow reason.
- Do not throw exceptions or return from `finally` blocks.
- Preserve the original stack trace when rethrowing exceptions.
- Close closeable resources, preferably with try-with-resources.
- Use `isEmpty()` for collection emptiness checks.
- Overridden methods must have `@Override`.
- Use the diamond operator where possible.

## Logging, Error Handling, And Forbidden APIs

- `System.out` and `System.err` are forbidden; use an SLF4J logger.
- `printStackTrace()` is forbidden.
- Do not catch an exception only to print its stack trace.
- Host JVM termination is forbidden:
  - `System.exit(int)`;
  - `Runtime.exit(int)`;
  - `Runtime.halt(int)`.
- Direct process execution through `Runtime.exec(...)` is forbidden. Any process
  execution capability must sit behind an approved and auditable process
  boundary.
- `Thread.sleep(...)` is forbidden. Tests must also use explicit time
  abstractions or deterministic synchronization.

## Spring And Lombok Rules

- `@Autowired` is forbidden. Use constructor injection.
- These Lombok annotations are forbidden:
  - `@Setter`;
  - `@Data`;
  - `@Value`;
  - `@SneakyThrows`.
- The parent configures Lombok as an annotation processor, but the annotations
  above are policy violations.

## Tests

- Unit test file names: `*Test.java` or `*Tests.java`.
- Integration test file names: `*IT.java` or `*IntegrationTest.java`.
- Surefire and Failsafe do not retry flaky tests and do not stop after the
  first failure.
- Test code goes through the same Checkstyle, PMD, SpotBugs, and Forbidden APIs
  gates as production code.
- `Thread.sleep(...)` is forbidden in tests too.
- The coverage gate means relevant branches must be tested, not only the happy
  path.

## Coverage

- JaCoCo minimum: 90% branch coverage at bundle level.
- JaCoCo minimum: 90% line coverage at bundle level.
- Unit and integration coverage reports are generated, then the parent merges
  execution data files.
- Pure data carriers are excluded from the coverage gate:
  - `**/*Dto.class`;
  - `**/*DTO.class`;
  - `**/dto/**`;
  - `**/*Entity.class`;
  - `**/entity/**`;
  - `**/*Record.class`;
  - `**/model/**`;
  - `**/*Request.class`;
  - `**/*Response.class`.

## Dependency And License Rules

- The parent defines a dependency license allowlist and blacklist.
- When the license gate actually runs, it fails on missing dependency license
  metadata.
- The current Maven goal is `aggregate-add-third-party`; the plugin may skip
  jar packaging and run on aggregate/root POMs.
- Allowed licenses:
  - Apache License 2.0 variants;
  - BSD / BSD-2-Clause / BSD 2-Clause / BSD 3-Clause;
  - Eclipse Public License 2.0 variants;
  - MIT variants;
  - PostgreSQL License.
- Explicit blacklist:
  - GNU General Public License;
  - GNU Affero General Public License;
  - Server Side Public License.
- The security scan runs only with the `security` profile. There, OWASP
  Dependency-Check treats every unsuppressed CVE finding as a build failure
  (`failBuildOnCVSS=0`, `junitFailOnCVSS=0`).

## Suppression Rules

Suppressions are for justified, reviewable exceptions. They are not a way to
turn off gates.

- Checkstyle: project-local suppression file:
  `config/checkstyle-suppressions.xml`.
- In multi-module projects, set `vp.suppressions.dir` to the reactor root so
  every module reads the same suppression file.
- PMD: use only targeted `@SuppressWarnings("PMD.RuleName")` or `// NOPMD`.
- SpotBugs: the shared base filter only excludes generated sources. For a
  project-local filter, override `vp.spotbugs.excludeFilterFile`, but preserve
  the generated-sources exclusion.
- OWASP: project-local suppression file:
  `config/owasp-suppressions.xml`; each CVE suppression should include a
  justification.
- Generated sources are excluded from Checkstyle and SpotBugs by default.

## Recommended Local Routine

```bash
mvn spotless:apply
mvn verify
```

After dependency or security-sensitive changes:

```bash
mvn -Psecurity verify
```

To generate an SBOM:

```bash
mvn -Psbom verify
```
