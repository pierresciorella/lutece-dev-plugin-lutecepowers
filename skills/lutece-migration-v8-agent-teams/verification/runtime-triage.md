# Runtime triage — the failures a green build cannot see

Read at PHASE H.1, when the bench or a deployed instance fails and the build is green. `verify-migration.sh`
and the reviewer read the sources; every failure below exists only at runtime, and the compiler reports none
of them.

A symptom is not a diagnosis. Name the family, prove it with the command given, then fix at the source and
attribute — the plugin, the core, the environment. A fix applied to the wrong family silences a screen and
leaves the defect.

## Contents
- Already named elsewhere
- A jar the target core no longer ships
- FreeMarker is strict on a null the previous engine tolerated
- A SQL error naming a column is rarely the migration
- Logs land in another webapp
- An empty screen, and no error anywhere
- Reading a bench failure with this file

## Already named elsewhere

These have a single source; go there rather than diagnosing twice.

| Symptom | Where |
|---|---|
| Webapp does not start, `ValidationFailedException`, `1 changesets check sum` | `rules/sql-liquibase.md` |
| Plugin tables missing, no exception, `files not managed by liquibase are …` | `rules/sql-liquibase.md` |
| `Duplicate entry`, tables emptied after a directory or plugin rename | `rules/sql-rename.md` |
| `Unsatisfied` / `Ambiguous dependency` at deployment | `patterns/cdi-patterns.md` §5 |
| A service, a cache or a listener acting twice | reflection-instantiated class carrying a CDI scope — `patterns/cdi-patterns.md` §2, `patterns/core-8x-moves.md` |
| `$ is not defined`, DataTables / Select2 / jQuery UI dead | `rules/template-back-office.md`, `rules/template-front-office.md` |
| `RequireUpperBoundDeps`, or a `NoClassDefFoundError` an `<exclusions>` caused | `rules/dependency-convergence.md` |
| A core class that moved to another artefact | `patterns/core-8x-moves.md` |
| JPQL Hibernate accepted, EclipseLink refuses | `patterns/persistence-patterns.md` §5 |

## A jar the target core no longer ships

**Symptom.** `NoClassDefFoundError` or `ClassNotFoundException` on a third-party package
(`org.apache.commons.*`, `net.sf.*`) at the **first use of a feature**, not at startup, while the calling code
did not change.

**Cause.** The previous core shipped that jar in `WEB-INF/lib` and the target core does not, so a library that
still needs it resolves nothing at runtime. The core drops artefacts between its own patch versions too:
`net.sf.opencsv` is a dependency of `lutece-core` 8.0.1 and not of 8.0.2.

**What decides it.** Compare the assembled webapps, not the poms — the pom says what is declared, the
directory says what is deployed:

```bash
diff <(unzip -Z1 lutece-core-<old>-webapp.zip | sort) <(unzip -Z1 lutece-core-<new>-webapp.zip | sort)
diff <(ls <site>/target/<old>/WEB-INF/lib | sort) <(ls <site>/target/<new>/WEB-INF/lib | sort)
```

**Fix.** Declare the artefact explicitly where the feature runs. It is not a version conflict, so an exclusion
or a pin changes nothing: nothing removed the class, the core simply stopped carrying it.

An application deployed as several webapps hits this asymmetrically: the artefact another module declares is
on that webapp's classpath and missing from the next one, so the same feature works on one side and fails on
the other. Test the feature where it runs, not where it is convenient.

## FreeMarker is strict on a null the previous engine tolerated

**Symptom.** `InvalidReferenceException … has evaluated to null or missing`, wrapped in
`LuteceFreemarkerException`, answering 500 on a screen that rendered before.

**Cause.** The engine no longer accepts a missing variable in `${…}` or `#list`. The template is usually not
the defect: one code path that renders it does not populate the model the nominal path fills — a secondary
handler, an error branch, an AJAX entry point.

**What decides it.** The stack names the template, the line and the variable. Find who fills it:

```bash
grep -rn "MARK_<the variable>\|\"<the variable>\"" src/java
```

**Fix.** Populate the model in the controller, on that path. `!` or `![]` in the template is a last resort: it
hides a model the caller was supposed to fill, and the screen then renders empty instead of failing.

Two neighbours, different families: a `ParseException` or an unclosed directive is a **syntax** defect —
`scripts/check-template-parse.sh` finds it without a server; a macro called with an argument it does not
declare is found by `scripts/render-template.sh`. This one is **semantic**, fires only on the path that
renders, and is therefore what the bench catches and the unit tests do not.

Two instances of the family are already written down as incidents: `url_banner`
(`teammates/config-migrator.md`) and `pageContainer` (`teammates/test-migrator.md`). Both are the same
shape — a model the rendering path does not fill.

## A SQL error naming a column is rarely the migration

**Symptom.** `column "x" does not exist` (or an unknown table) raised by one screen, while the core upgrade ran.

**Cause**, in order of likelihood: the plugin's own upgrade chain never ran on that database; or the branch
carries a functional change — a column, a view, a trigger — shipped by a later application version and
unrelated to the platform.

**What decides it.** Read the stack: the failing DAO names the plugin, which already rules out the core. Then
ask the sources, then the database:

```bash
grep -rn "<column>" src/sql
mysql <db> -e "SELECT ID, FILENAME, EXECTYPE FROM DATABASECHANGELOG WHERE FILENAME LIKE '%<plugin>%' ORDER BY ORDEREXECUTED"
```

- No row at all for the plugin → the SQL directory name does not match the plugin `<name>`: `rules/sql-rename.md`.
- Rows stop before the script that adds the column → the upgrade script is missing, or has no Liquibase
  header: `rules/sql-liquibase.md`.
- The object appears nowhere in the sources → functional debt of the application, not the migration. Report it
  as a prerequisite to replay on every environment; never absorb it into the migration.

## Logs land in another webapp

**Symptom.** `WEB-INF/logs/` of the webapp under investigation stays empty while the application serves
requests; `application.log` has size 0.

**Cause.** Several webapps in one JVM share the log4j2 context. The core configuration resolves
`luteceLogDirectory` through `${web:rootDir}`, which the **first initialised** webapp wins: every webapp of
that JVM then writes into that one directory.

**What decides it.** Look for the file that grows, not the one expected:

```bash
find <tomcat|liberty>/ -name 'application.log' -printf '%TY-%Tm-%Td %TH:%TM %10s %p\n' | sort
```

**Fix.** Pin the directory in the profile's own `WEB-INF/conf/log.properties` (log4j2) with a portable lookup
such as `${sys:catalina.base}/logs`. Overriding `override/plugins/log.properties` has no effect: that is the
log4j1 file, which the core no longer reads.

Different from the backend diversion of `rules/dependency-convergence.md` (`jboss-logmanager` taking priority
over Log4j2): here the backend is right and the destination is wrong. The bench never shows it — one
application per container — which is exactly why it costs hours on a shared environment.

## An empty screen, and no error anywhere

**Symptom.** A screen renders cleanly but a list, a tree or a reference is empty or partial. No exception, in
any log. The same call sometimes returns an empty response, then a correct one.

**Cause.** The screen feeds on an internal HTTP call whose client times out; the service catches and returns an
empty list. An empty response is the **client** giving up, not a server failure — which is why nothing is
logged server-side.

**What decides it.** Time the same endpoint on both versions, plus a light control endpoint to prove the chain:

```bash
curl -s -m 60 -o /dev/null -w "HTTP %{http_code} | %{time_total}s | %{size_download}B\n" <url>
```

| Reading | Verdict |
|---|---|
| Same payload, very different time | performance, not migration |
| Same time, different body | functional regression — go into the service |
| Same 404 on both | the endpoint never existed; out of the migration's scope |

**Classic root cause.** A tree or a reference list built by recursion, one query per node. The cost is
N × latency: invisible where the application and the database share a host, tens of seconds from a workstation
against a remote database, and the client gives up. Qualify it honestly — a pre-existing inefficiency amplified
by topology is not a regression introduced by the migration, and it is not nothing either. Report it; do not
fix it inside the migration.

To unblock a workstation without touching the application, raise the client timeouts **in the local profile
only** (`src/conf/local/WEB-INF/conf/override/plugins/httpaccess.properties`):

```properties
httpAccess.connectionTimeout=10000
httpAccess.socketTimeout=60000
```

Properties are read at startup: restart. Templates reload hot, properties do not.

## Reading a bench failure with this file

`python3 tools/causes.py` prints the server exception logged in the failed step's window. Match it to a family
above, fix at the source, and attribute the finding (PHASE H.2). Three readings that save a wrong fix:

- an exception naming nothing of the artefact is the platform's noise in the same window — `causes.py` marks it
  `[hors périmètre]`, and the run's own attribution rule (`lutece-e2e`, `reference/scope.md`) already keeps it
  from reddening the bench;
- a failure with **no** exception at all is not a harness bug by default: the empty-screen family above logs
  nothing by construction, and a jar the core stopped shipping fails under a package the artefact's fingerprint
  does not name;
- a screen that fails **with** its parameters is a defect of the plugin, not a bench gap
  (`lutece-e2e`, `reference/traps.md`).
