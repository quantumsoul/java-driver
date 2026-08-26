# Building the docs

## Prerequisites

To build the documentation of this project, you need a UNIX-based operating system. Windows is not fully supported as it does not support symlinks.

You also need the following software installed to generate the reference documentation of the driver:

- Java JDK 11 or higher
- Maven

Once you have installed the above software, you can build and preview the documentation by following the steps outlined in the `Quickstart guide <https://sphinx-theme.scylladb.com/stable/getting-started/quickstart.html>`_.

## Custom commands

To generate the reference documentation of the driver, run the command `make javadoc`. This command generates the reference documentation using the Javadoc tool in the `_build/dirhtml/<VERSION>/api` directory.

## Using the Makefile

Most day-to-day tasks are wrapped in the top-level `Makefile` so you do not have to remember long Maven invocations. Common targets include:

- `make download-all-dependencies` pre-fetches all Maven artifacts to warm local caches.
- `make compile-all` compiles main and test sources without running tests, skipping format, clirr, and animal-sniffer checks for speed.
- `make test-unit` runs the fast unit-test suite; use `MVNCMD` to tweak the underlying Maven command if needed.
- `make test-integration-scylla` and `make test-integration-cassandra` execute CCM-backed integration suites. Export `SCYLLA_VERSION` or `CASSANDRA_VERSION` to pin specific server versions before invoking.
- `make check` executes `mvn verify -DskipTests` for static analysis, while `make fix` applies code formatters via `mvn fmt:format`.
- `make fix` executes `mvn fmt:format` to format the code.
- `make clean` removes Maven targets, shaded artifacts, and release backups to reset the tree.

### Measuring code coverage

`jacoco-maven-plugin`'s instrumentation/report executions live behind an opt-in `coverage` Maven
profile (see the root `pom.xml`), not the default build, so `test-unit`/`test-integration-*` do
*not* collect coverage data on their own. The targets below activate that profile and each write
their own `target/jacoco.exec`. What's missing without them is a combined, cross-module view (e.g.
attributing coverage `core` gets *through* the integration suite back to `core`'s own source), and,
for unit tests specifically, resilience to one module's test failure discarding every other
module's data:

- `make test-unit-coverage` runs the same modules as `make test-unit` -- one reactor-wide `mvn
  test` -- but with `-Dmaven.test.failure.ignore=true`. Without that, a test failure in `core`
  would make Maven skip every module that depends on it (`query-builder`, `mapper-runtime`, ...)
  too -- losing their coverage data along with core's, regardless of `-fae`/`-fn`, since those
  flags only rescue *independent* modules, not ones with a real dependency on the failed one.
  Because that flag also makes surefire swallow the failure (so Maven's own exit code no longer
  reflects it), the target reads pass/fail back from the surefire XML reports afterwards.
- `make test-integration-scylla` / `make test-integration-cassandra` need no coverage-specific
  variant beyond passing `MAVEN_EXTRA_ARGS=-Pcoverage` to activate the profile: `maven-failsafe-plugin`
  already separates running integration tests (`integration-test` phase, which always completes)
  from failing the build on their results (`verify` phase), so a test failure there was never able
  to lose coverage data in the first place.
- `make coverage-report` merges whatever the above collected into one cross-module report, via a
  dedicated `coverage-report` module that depends on the others and runs
  `jacoco:report-aggregate`. Run whichever of `test-unit-coverage` / `test-integration-scylla` /
  `test-integration-cassandra` you want measured first, then this (it fails fast if it finds no
  `jacoco.exec` anywhere, rather than rendering a confident-looking but empty report). It writes an
  HTML report to `coverage-report/target/site/jacoco-aggregate/index.html` (open it in a browser
  for a line-by-line view), plus `jacoco.xml`/`jacoco.csv` alongside it.
- `make clean-coverage` deletes every module's `jacoco.exec` and `target/site/jacoco/`, plus the
  aggregate report.

Note: the surefire/failsafe configs in `core` and `integration-tests` previously set `<argLine>` to
just their own JVM flags (e.g. `${mockitoopens.argline}`), which silently discarded the
`-javaagent` flag `jacoco:prepare-agent` injects into the `argLine` property -- coverage was being
collected for every *other* module, but not these two. They now combine both via Maven's
deferred-property syntax: `<argLine>@{argLine} ${mockitoopens.argline}</argLine>` (`@{...}` is
necessary rather than `${...}` because `jacoco:prepare-agent` sets `argLine` at build-execution
time, after the POM's own `${...}` references would already have been resolved). `argLine` itself
is declared, empty, as a root `pom.xml` property so that combination resolves to something even
outside the `coverage` profile, where `jacoco:prepare-agent` never runs to give it a real value.
(`distribution-tests` has no `src` of its own, so surefire never forks there either way; it was
left out of this.)

The Makefile automatically installs the shaded Guava dependency and, for integration tests, bootstraps the appropriate CCM toolchain and raises kernel `aio-max-nr` when required. If a target fails because the toolchain is missing, rerun after installing the prerequisites highlighted in the target output.
