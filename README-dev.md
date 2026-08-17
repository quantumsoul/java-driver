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

`jacoco-maven-plugin` is already wired into every module (see the root `pom.xml`), so
`test-unit`/`test-integration-*` collect coverage data as a side effect of running normally --
each module writes its own `target/jacoco.exec`. What's missing without the targets below is a
combined, cross-module view (e.g. attributing coverage `core` gets *through* the integration
suite back to `core`'s own source), and, for unit tests specifically, resilience to one module's
test failure discarding every other module's data:

- `make test-unit-coverage` runs the same modules as `make test-unit`, but as one `mvn`
  invocation per module instead of a single reactor-wide one. This matters because in a single
  `mvn test` reactor build, a test failure in `core` makes Maven skip every module that depends on
  it (`query-builder`, `mapper-runtime`, ...) too -- losing their coverage data along with core's,
  regardless of `-fae`/`-fn`, since those flags only rescue *independent* modules, not ones with
  a real dependency on the failed one.
- `make test-integration-scylla` / `make test-integration-cassandra` need no coverage-specific
  variant: `maven-failsafe-plugin` already separates running integration tests (`integration-test`
  phase, which always completes) from failing the build on their results (`verify` phase), so a
  test failure there was never able to lose coverage data in the first place.
- `make coverage-report` merges whatever the above collected into one cross-module report, via a
  dedicated `coverage-report` module that depends on the others and runs
  `jacoco:report-aggregate`. Run whichever of `test-unit-coverage` / `test-integration-scylla` /
  `test-integration-cassandra` you want measured first, then this. It prints a per-module summary
  and writes an HTML report to `coverage-report/target/site/jacoco-aggregate/index.html` (open it
  in a browser for a line-by-line view), plus `jacoco.xml`/`jacoco.csv` alongside it.
- `make clean-coverage` deletes every module's `jacoco.exec` and the generated report.

Note: the surefire/failsafe configs in `core`, `integration-tests`, and `distribution-tests`
previously set `<argLine>` to just their own JVM flags (e.g. `${mockitoopens.argline}`), which
silently discarded the `-javaagent` flag `jacoco:prepare-agent` injects into the `argLine`
property -- coverage was being collected for every *other* module, but not these three. They now
combine both via Maven's deferred-property syntax: `<argLine>@{argLine} ${mockitoopens.argline}</argLine>`
(`@{...}` is necessary rather than `${...}` because `jacoco:prepare-agent` sets `argLine` at
build-execution time, after the POM's own `${...}` references would already have been resolved).

The Makefile automatically installs the shaded Guava dependency and, for integration tests, bootstraps the appropriate CCM toolchain and raises kernel `aio-max-nr` when required. If a target fails because the toolchain is missing, rerun after installing the prerequisites highlighted in the target output.
