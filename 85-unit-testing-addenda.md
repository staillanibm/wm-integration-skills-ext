# Unit testing: addenda from practice

Complements `60-unit-testing.md` (which wins over the supplement). Read that file first. Some test
tooling writes a layout different from the one described there: `resources/test/setup/<suite>.xml`
and `resources/test/data/<suite>/<case>_input.xml` (`_expected.xml`), with file names relative to the
package root. In that case, keep `resources/test/` at the package root of the local mirror, beside
`sources/` and the namespace folder, so it can be copied onto the server unchanged.

## Test the persisted version

Reload the package before every run (`40-deployment-and-test-loop.md`). A suite that is green before a
reload and red after is the normal symptom of a service whose persisted node lost a copy.

## Choosing the expectation

| Output | Use |
|---|---|
| flat values, few fields | expected values (subset match) |
| values in lists / nested records | field assertions with JXPath: `/taxes[1]/rate`, `count(/taxes)` (1-based indexes) |
| a record whose keys contain `:` | a recorded snapshot, never JXPath: the colon is read as a namespace prefix |
| an exception | an expected exception with a stable substring of the message |
| a large generated document (XML) | an expected file holding the exact string |

Expected values are a **subset** match: extra variables in the actual pipeline are ignored, so leaked
variables do not fail a test. Test leaks separately if they matter (see below).

## Pipeline Hygiene Testing Pattern (MANDATORY for nominal cases)

Since `<expected>` is a subset match, a service leaving unwanted temporary variables, intermediate records, or technical invoke parameters (`inString`, `jsonString`, `pattern`, `num1`, `num2`, etc.) in the pipeline will still pass unit tests silently.

To guarantee and enforce pipeline hygiene:
1. In the primary nominal/happy-path test case, include explicit field assertions (JXPath) verifying that all temporary and input variables are `$null` / absent.
2. An equality assertion against `null` checks that the JXPath node evaluates to nothing.

Example, for a service with input `inputParam` and output `expectedOutput`:

| JXPath | Assertion |
|---|---|
| `/expectedOutput` | not empty |
| `/inputParam` | `null` (absent) |
| `/tempRecord` | `null` (absent) |
| `/inString` | `null` (absent) |
| `/jsonString` | `null` (absent) |

## Recording the expectation is a trap when an input is itself JSON or XML

The recording invokes the service through a different channel than the runner. With an input string
that looks like JSON it produced an exception instead of the result, which was then recorded as the
expectation. Rules:

- read the recorded result every time: a recorded `exception` you did not expect is a bug in the
  recording, not an expectation;
- for inputs that are serialised documents, create the case with a placeholder expectation, then write
  the `_expected.xml` file yourself from a real call that you **verified** (for XML output: against the
  reference document, element by element);
- a snapshot freezes what the service does today, bugs included: verify it before keeping it.

## Cases to write for every service

- the nominal case of the specification, with its documented values;
- each validation failure, with the exact message, **and the order** when several inputs are invalid
  (first reported wins);
- an **absent** input, an **empty** input and a **blank** input: they do not behave the same;
- the boundaries of the business rules (rounding exactly on a half, one element vs several, same
  key written differently such as `5.5` and `5.50`);
- optional fields absent (the element must be omitted, not emitted empty).

## Prove that the suite can fail

Once green: alter one expected value directly in the suite file on the server, reload, run and check
that `failureCount` is 1, restore the file, reload, run and check that it is green again. Do it once
per new suite set, and keep the original file aside before altering it.

## Linearising arithmetic services — pre-code checklist

Before writing any FSL code for a service that computes monetary or numeric values:

1. **Calculate expected values by hand** for the inputs you will test — including at least one exact
   half-up case and one multi-rate case if applicable. Write them down before generating any code.
   A discrepancy between your manual calculation and the service output is a bug; a match is
   confirmation. Never skip this step: it takes two minutes and prevents a debug cycle.
3. **Identify output format constraints** (e.g. schema requires `-?[0-9]+\.[0-9]{2}`). Note them
   before coding so the formatting chain is planned from the start, not bolted on after seeing bad output.

## A recipe that does not depend on the unit test framework

Also provide a script that replays the acceptance criteria over HTTP (`/invoke/<interface>/<service>`,
JSON body, `Accept: application/json`; a service failure answers HTTP 500 with the message in `$error`).
For XML outputs check well-formedness (`xmllint --noout`), namespaces, the order of the top-level
elements, and, when a reference document exists, structural identity with it (same elements, order,
attributes, values).
