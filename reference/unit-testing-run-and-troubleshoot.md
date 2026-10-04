# Unit tests: deploying, running, diagnosing

Reference file, read on demand from `60-unit-testing.md`. Verified against a live Integration Server
12.1. Where it disagrees with `supplements/unit-testing-guide.md`, this file wins.

These steps describe *what* must happen on the server, not *which* tool does it — use whatever the
environment provides (Designer, IS Administrator, a deployment pipeline, a management API, or the
`wm-mcp-server` `unit-test` scope). The sequence does not change.

---

## Suite XML: `folder`, `name`, `filename`

- `folder` is the service's **full interface path** — the same dot-separated path as the FSL `interface`
  declaration (`{rootDir}.services` plus any folder path), *without* the service name. `name` is the
  unqualified service name. Interface `myPackage.services.orders`, `service processOrder` →
  `folder="myPackage.services.orders" name="processOrder"`. Writing only `orders` or the bare `services`
  does not resolve; the suite reports 0 tests or fails to find the service.
- `filename` must include the `tests/` prefix and resolves relative to the **package root**
  (`<package>/tests/...`) — NOT relative to the suite XML's own location, and NOT relative to the
  interface folder the service lives under.
- The suite XML itself is placed under `<package>/resources/<suiteName>/`; the input/result files it
  references live under `<package>/tests/` at the package root.

## Expecting a service exception

Use `<expected><exception class="..." message="..."/></expected>`, with no `-result.xml` file.

- `class`: `com.wm.app.b2b.client.ServiceException` for a flow `EXIT` with `signal: FAILURE` or any
  flow-level failure.
- `message`: must match the failure message **exactly**. A wrong message makes the case fail, so these
  assertions verify *which* error was raised, not merely *that* one was.

⚠️ **Do not copy the class name out of an HTTP error response.** Invoking the service directly over HTTP
returns `com.wm.lang.flow.FlowException` in `$errorType` (the server-side flow engine's exception). The
test framework runs as a **client** and sees the wrapper `com.wm.app.b2b.client.ServiceException`. Always
write the client class in the suite XML. A case using the server-side name fails with a mismatch even
though the service behaved as intended.

## Complete worked example

Service `addTwoNumbers`, package `myPackage` (interface `myPackage.services`), inputs `num1` and `num2`
as Strings, output `result`, failing with `num1 must be an integer` when an operand is not integral.

**`myPackage/tests/success_test1-input.xml`**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<IDataXMLCoder version="1.0">
  <record javaclass="com.wm.data.ISMemDataImpl">
    <value name="num1">10</value>
    <value name="num2">20</value>
  </record>
</IDataXMLCoder>
```

**`myPackage/tests/success_test1-result.xml`**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<IDataXMLCoder version="1.0">
  <record javaclass="com.wm.data.ISMemDataImpl">
    <value name="result">30</value>
  </record>
</IDataXMLCoder>
```

The expected result must list **exactly** the fields the service leaves in the pipeline — no more, no
fewer. If the service drops its inputs and every temporary before returning, `result` alone is correct;
if it leaks a helper variable, the comparison fails until the service or the expected file is fixed. That
is usually the service's bug, not the test's.

**`myPackage/tests/error_not_integer-input.xml`** (no `-result.xml`: the assertion is inline)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<IDataXMLCoder version="1.0">
  <record javaclass="com.wm.data.ISMemDataImpl">
    <value name="num1">abc</value>
    <value name="num2">3</value>
  </record>
</IDataXMLCoder>
```

**`myPackage/resources/addTwoNumbersTestSuite/addTwoNumbers_TestSuite.xml`**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<webMethodsTestSuite description="addTwoNumbers TestSuite" name="addTwoNumbersTestSuite">
    <webMethodsTestCase description="Success Test: 10 + 20 = 30" name="success_test1">
        <service folder="myPackage.services" name="addTwoNumbers">
            <input>
                <file filename="tests/success_test1-input.xml"/>
            </input>
            <expected>
                <file filename="tests/success_test1-result.xml"/>
            </expected>
        </service>
    </webMethodsTestCase>
    <webMethodsTestCase description="Validation Test: num1 is not numeric" name="error_not_integer">
        <service folder="myPackage.services" name="addTwoNumbers">
            <input>
                <file filename="tests/error_not_integer-input.xml"/>
            </input>
            <expected>
                <exception class="com.wm.app.b2b.client.ServiceException" message="num1 must be an integer"/>
            </expected>
        </service>
    </webMethodsTestCase>
</webMethodsTestSuite>
```

Three paths: the suite under `resources/<suiteName>/`, the data files under `tests/` at the package
root, and `filename` relative to that root — never relative to the suite.

---

## Deploying and running the suite

The suite lives on the Integration Server's own filesystem, and the engine only discovers it after the
package is reloaded.

### Step 1 — Place the files, flattening the local layout

Deploying a service does **not** deploy its tests. Copy them separately, converting the local
per-service layout into the server's flat one:

| Local (source of truth) | Server |
|---|---|
| `{service folder}/tests/{ServiceName}_TestSuite.xml` | `<package>/resources/<suiteName>/` |
| `{service folder}/tests/*-input.xml` / `*-result.xml` | `<package>/tests/` |

Every service's data files land in the *same* `<package>/tests/` directory however deeply the services
are nested locally, so **test case names must be unique across the whole package**. Two services each
with a `success_test1-input.xml` overwrite each other; prefix the file names.

#### If the files are written *by* the Integration Server, whitelist the paths first

When the IS runs on the machine you author on, copy with ordinary filesystem operations and skip the rest
of this step — no server configuration is involved.

When the files must be written **by the server itself** — a remote IS, or any flow using the `pub.file`
services — the target directories must first be allowed in `config/server.cnf`:

```
watt.server.file.canWritePaths
watt.server.file.canReadPaths
watt.server.file.canDeletePaths
```

On IS 12.1 an **empty** value means *no path is permitted*, not *all paths*. At the default, every write
fails with `[ISS.0086.9263]` and the suite never appears. Grant the two directories the framework needs:

```
watt.server.file.canWritePaths=<IS>/packages/*/tests;<IS>/packages/*/resources/*
```

- **Entries are separated by `;`** — not a comma or a colon.
- **Use absolute paths.** Relative entries resolve against the JVM working directory, not the package root.
- **The pattern is matched against the file's parent directory**, so the suite entry needs the trailing
  `/*`: the suite XML lives in `resources/<suiteName>/`, one level below `resources/`.
- **The server must be restarted** for the change to take effect — a package reload is not enough.

This is a deliberate security boundary. Widen it to the specific directories needed; never to the whole
package tree or the installation root.

### Step 2 — Reload the package (mandatory, every time)

The engine scans `resources/` when the package loads. A suite added or edited on disk is invisible until
then — and a stale suite keeps running in its place, which looks like an edit that silently had no
effect. Reload after **every** change to the suite XML or the data files.

### Step 3 — Run the suite and read the counters

| Counter | Meaning |
|---|---|
| `testCount` | Cases actually executed. **Check first** — `0` means the suite was never found, not that everything passed. |
| `failureCount` | Assertions that did not hold: output differed, or the exception's message/class did not match. The framework worked; the service or the expectation is wrong. |
| `errorCount` | The case could not be evaluated at all — missing file, unresolvable service, setup exception. Usually wiring, not a service bug. |
| `suiteCount` | Suites discovered. Confirms the suite XML was picked up. |

Fix errors before failures: a case that never ran tells you nothing about the service.

Two report formats give detail beyond the counters, both retrieved with the execution identifier: a
plain-text report (names each case with its individual result — the quickest way to confirm the cases
you expected actually ran) and a JUnit XML report for CI (same information plus verbose captured output;
can be large — extract the `<testsuite>`/`<testcase>` elements rather than reading it whole).

### Step 4 — Verify the suite actually discriminates

A suite that passes proves nothing until you have seen it fail. Once green, deliberately break one
expectation — change an expected value, or alter one character of an expected exception message — reload,
and re-run. `failureCount` must become non-zero. Restore, reload, re-run, and confirm it returns to
green. Do this once when the suite is first created; it is the only thing that distinguishes a real test
from a decorative one.

---

## Troubleshooting

### Error/validation test reports as `<error>` instead of passing
The case almost certainly uses `<expected><file .../></expected>` with an `errorMessage`/`$error` field
instead of `<expected><exception class="..." message="..."/></expected>`. Fix the suite XML; the service
may already be correct.

### Exception test fails on a class mismatch
Check `class` is the **client-side** `com.wm.app.b2b.client.ServiceException`, not a server-side class
such as `com.wm.lang.flow.FlowException` copied from an HTTP error response.

### Edits to the suite appear to have no effect
The package was not reloaded; the engine keeps serving the previously loaded suite. Reload and re-run. If
`testCount` still shows the old case count, the reload did not take.

### A test passes that should not
- Confirm the suite discriminates at all (Step 4).
- For output comparisons, check the expected file lists every field the service actually returns. Extra
  pipeline variables left behind by the service — helper flags, a called service's native outputs — will
  surface here.

### Two services' tests overwrite each other
All data files share one flat `<package>/tests/` directory on the server. Test case names must be unique
package-wide; prefix them per service.

### `[ISS.0086.9263]` when writing the files
The write was performed *by* the Integration Server (a `pub.file` service) and the target directory is
not in `watt.server.file.canWritePaths`. See the whitelist notes under Step 1: check the `;` separator,
absolute paths, the trailing `/*` on the `resources` entry, and remember the server must be restarted.
This does not arise when the files are copied with ordinary filesystem operations on a local install.

---

## The four things that most often go wrong

1. **IDataXMLCoder, not `<Values>`** — data files use `<record javaclass="com.wm.data.ISMemDataImpl">`.
2. **Two different directories** — the suite under `resources/<suiteName>/`, its data files under `tests/`
   at the package root, with `filename` relative to that root.
3. **Reload before every run** — otherwise the previously loaded suite keeps running.
4. **The client-side exception class** — `com.wm.app.b2b.client.ServiceException`, never the class seen
   in an HTTP error response.
