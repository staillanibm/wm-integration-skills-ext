# Unit tests: authoring corrections, deploying, running

Applies to `supplements/unit-testing-guide.md`. Where this file and that supplement disagree, **this
file wins** — the points below were verified against a live Integration Server 12.1, and the
supplement's text on them is wrong or missing.

**For any task that runs, deploys, re-runs or fixes a test suite — or diagnoses a suite reporting 0
tests, failing, or ignoring its edits — read
`extended-skill/reference/unit-testing-run-and-troubleshoot.md` in full first.** It is not loaded
automatically (it lives in a subfolder).

## Supplement trigger is wider

Load `supplements/unit-testing-guide.md` not only to **generate** tests but also to **run**, **execute**,
**deploy**, **re-run** or **fix** an existing suite. Those tasks generate no FSL: point 2 of "How to
apply a supplement" does not apply, and you must not fall back into Step 5 → Step 6.

Offering tests after producing a service is fine ("Would you like unit tests for this service?"); act on
the answer. Never generate them unasked.

## Before writing a single test case — mandatory pre-work

1. **Determine expected values by invoking the service**, not by predicting them. Run the service once
   with each happy-path input, read the actual output, and use those values as expectations. For error
   cases, run the service with the bad input and copy the exact message from `$error` in the response.
2. **If any output field contains XML, HTML, or any string with `<` characters — use `record: true`
   exclusively.** Never write a `*-result.xml` file by hand for such a field: the `IDataXMLCoder` reader
   will parse the unescaped content as XML structure and the comparison will fail with tags stripped.
   `record: true` stores the value in the correct native binary format.
3. **Verify every recording** — read the `recordings` section of `test_suite_create`'s response. A
   recorded `exception` you did not expect means the recording failed; fix the input before keeping it
   as the expectation.

## Corrections to the supplement

1. **`filename` is relative to the package root**, not to the service folder and not to the suite file.
   `<file filename="tests/x-input.xml"/>` resolves to `<package>/tests/x-input.xml`.
2. **Server layout is flat and does NOT mirror the local one.** Suite XML →
   `<package>/resources/<suiteName>/`. All `*-input.xml` / `*-result.xml` → `<package>/tests/`. Locally
   they sit under the service folder (see `10-package-and-output-layout.md`); flatten when copying.
3. **`folder` is the full interface path**, without the service name: `folder="myPackage.services.orders"
   name="processOrder"`. Not `orders`, not bare `services` — those report 0 tests.
4. **Test case names are unique package-wide**, since every data file lands in one `tests/` directory.
   Prefix file names per service when a package holds several tested services.
5. **Exception class is the client-side one:** `com.wm.app.b2b.client.ServiceException`. Never copy
   `com.wm.lang.flow.FlowException` from an HTTP error dump. Message match is exact.
6. **Deploying the service does not deploy its tests.** Copy them, then reload the package.
7. **Reload the package after every change** to the suite or its data files; otherwise the previously
   loaded suite keeps running and edits appear ignored.
8. **Read the counters in order:** `testCount` first (0 = suite never found, not "all passed"), then
   `errorCount` (case could not be evaluated: wiring), then `failureCount` (assertion did not hold).
9. **Prove the suite discriminates:** once green, break one expectation, reload, re-run — `failureCount`
   must become non-zero — then restore and confirm green. Do it once per new suite.
10. **Determine expected values by invoking the service once** when its real output (failure message
    wording, which fields survive in the pipeline) is not obvious. Do not predict them.

## Tooling

With `wm-mcp-server` (scope `unit-test`): `package_reload` (mandatory before every run), `test_run`,
`test_check_status`, `test_text_report`, `test_junit_report`, and `mock_*` for mocking. The scope
deliberately has no namespace mutation, so copying suite files onto the server is done outside MCP
(filesystem copy for a local IS). If the files must be written by the server itself, the paths must first
be whitelisted — see the reference file.
