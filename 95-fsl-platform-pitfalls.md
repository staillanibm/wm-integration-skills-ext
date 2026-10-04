# FSL / Integration Server pitfalls observed in practice (IS 12.1)

Complements `40-` to `90-`. Loaded after them, so where this file disagrees with an earlier one, **this
file wins**. Everything below was observed on a live Integration Server 12.1. Items marked
*hypothesis* are not established.

## 1. `package_reload` says "reloaded" even when a service did not compile

A service whose FSL cannot be compiled by the loader is simply not created: the reload still answers
`reloaded`, the next `service_invoke` answers `UnknownServiceException` / HTTP 404, and the package
folder has a `flow.flow` but no `flow.xml` / `node.ndf`.

After **every** reload, before invoking anything:

- check that `flow.xml` and `node.ndf` exist next to the `flow.flow` (compare the `node.ndf` signature
  fields with the FSL, see `40-`);
- on a compile failure read `<IS>/logs/server.log` (location depends on the install: look for
  `server.log` under the IS directory) and search for `FSL Compilation Failed for FSL script` followed by
  `Line <n>:<col> - <message>` lines. Old failures stay in the log: read the **last** ones. A failure
  logged with the message `null` is a compiler exception, not a syntax error.

`dsl_validate` is not a substitute: it passes FSL that the loader still rejects or silently trims.

## 2. `$null` does not match the empty String

`IF (%payload% == $null)` is true for an **absent** variable only. An empty String (`""`) goes through.
A service that must reject "absent or empty" input fails late, with a platform message, instead of its
own. Pattern that covers absent, empty and blank:

1. `IF (%payload% == $null)` → fail (absent);
2. `pub.string:trim` (`inString` → `value`);
3. `IF (%trimmed% == "")` → fail with the **same** message.

(`trim` on an absent input was not tested: keep step 1 first.) Add the three cases to the unit tests.

## 3. A list `copy` can be dropped when its target has the same name as a child of its source

`copy `order/items` -> items;` in a MAP whose `mapSource` is `record order { recordList items }` and
whose `mapTarget` is a top-level `recordList items` compiled **without** the copy: no error, `items`
stayed empty, and the callee failed later with `Missing Parameter`. Renaming the target (and the
callee's parameter with it) fixed it. *Hypothesis*: the compiler resolves the target name against the
source's own child.

Rule: never give a copy target the name of a child of the source record. Always run the `copy`-vs-
`MAPCOPY` comparison of `40-` after writing a MAP that moves a list or a record.

## 4. Corrections to `50-` / `reference/document-assembly-and-silent-mapcopy.md`

- **Copying a String into a field of a record declared in `mapTarget` works.** Many such copies
  compiled and ran correctly, to depth 4, with names containing `:`, `@` and `*`:
  `copy x -> `Party/ns:Address/ns:City`;`
  `copy currency -> `Total/ns:Amount/@currency`;`
  The statement "a copy targeting an individual field of a record is discarded" is true of `set` on a
  path (Case A), not of `copy` into a declared record. Prefer this form: it needs no literal, no
  nesting-level splitting and no flat constants other than the String constants themselves.
- **Whole-record and whole-list grafts work** (`copy Party -> supplierParty`,
  `copy toList -> `Total/ns:Line``, `copy Total -> `document/Root/ns:Total``).
- **Case B (root name must start with `Doc`) was not reproduced.** Roots with unrelated names, and a
  list copied into a nested list, all worked. Do not rename roots pre-emptively; rename only if the
  `MAPCOPY` comparison shows a loss.
- **Case C was not reproduced either** where each appended record is built flat and dropped after
  `appendToDocumentList` (distinct entries every time). Keep the drop of the appended record.
- A path with two multi-segment sides needs backticks on **both** sides
  (`copy `a/b` -> `c/d`;`); the failure is a hard compile error (`mismatched input '/b -> c/'`).
- To serialise to XML, build one top-level record named `document` containing the XML root record
  (`document/Root/...`), put the default namespace in an `@xmlns` String of the root, the prefixes in an
  `nsDecls` record, set `documentTypeName` and `encode = "true"`, and call
  `pub.xml:documentToXMLString` with no mapping. Element order, omission of absent optional elements and
  escaping of `& < > "` all came out right.

## 5. Patterns verified and worth reusing

- **Group and sum by key:** prepare `documents` and `groupCriteria`
  (`set groupCriteria = [{"key": "rate", "compareStringsAs": "numeric"}];`) in a MAP, call
  `pub.document:groupDocuments` with no mapping, then `LOOP` over `/documentGroups` with an inner
  `LOOP` over `/documentGroups/documents`. `numeric` merges `5.5` and `5.50` and sorts ascending by
  value (`2.10`, `5.50`, `20.00`), not as text.
- **Rounding granularity matters:** rounding a percentage per element or once on a cumulated base gives
  different cents (0.10 + 0.10 at 5.5 %: 0.01 on the base, 0.02 per element). Do what the specification
  says, and test it.
- **Flat-String INVOKE mapping is reliable** (a callee's String parameters in `input`, its String
  result in `output`); it saves a preparation MAP per call.
- **Read the log, not `node_get`:** `node_get` on a service with a large signature returns tens of
  thousands of characters. Use the compiled files and the log instead.
- **Repetitive declarations may be generated by a script** (the same structure appears in several
  `mapSource` / `mapTarget` blocks); the generated FSL file is the deliverable and is then edited
  directly.

## 6. `pub.math:roundNumber` strips trailing zeros — always reformat

`roundNumber("180.00", 2)` → `"180.0"`, not `"180.00"`. The service rounds numerically but renders with
the minimum digits needed. This breaks any schema or comparison that requires exactly two decimal places.

**Mandatory pattern after every `roundNumber` call:**

```fsl
INVOKE pub.string:numericFormat {
    input { ... copy rounded -> num; set `pattern` = "0.00"; }
    output { ... copy `value` -> formatted; }
}
INVOKE pub.string:replace {
    input { ... copy formatted -> inString; set searchString = ","; set replaceString = "."; }
    output { ... copy `value` -> finalAmount; }
}
```

The `replace` is not optional: `numericFormat` follows the server locale and may produce `"180,00"` with
a comma on a French-locale server. The two calls together are always correct.

Apply this to **every** monetary output field — line amounts, VAT amounts, and totals.

## 7. `pub.math:divideObjects` loses precision when the divisor is large — multiply first

`divideObjects(5.5, 100)` with BigDecimal operands returns `0.06` (rounds `0.055` to 0 decimal places
before the result scale is set). **Never divide a rate by 100 to get a factor, then multiply.**

Correct order for VAT: multiply `lineAmount × vatRate` first, then divide by 100:

```
lineAmount × vatRate = 180.00 × 5.50 = 990.00   (exact)
990.00 / 100         = 9.90                       (exact, no rounding loss)
```

Wrong order:
```
vatRate / 100 = 5.50 / 100 = 0.06  ← rounded to 0 dp, precision lost
lineAmount × 0.06 = 180.00 × 0.06 = 10.80  ← wrong
```

This applies to any percentage computation in FSL.

## 8. Unit tests are part of the delivery when the specification asks for testability

This replaces "never generate them unasked" of `60-unit-testing.md` in that case. Deliver one suite per
service, without waiting to be asked, when the specification says the service must be testable /
replayable, lists test cases, or gives acceptance criteria. Otherwise keep the old rule and offer them.

How, with `test_suite_create` (it writes `resources/test/setup/<suite>.xml` and
`resources/test/data/<suite>/`; the layout described in `supplements/unit-testing-guide.md` is not the
one produced):

- one suite per service, case names **prefixed per service** (they must be unique in the package);
- `expected` for flat values, `expected_fields` (JXPath, 1-based, `count(...)`) for lists and nested
  values **without `:` in key names**, `expected` as a nested JSON subset for records whose keys contain
  `:` or `@`, `expected_exception` with a stable substring of the message for failures (the client
  exception class is filled in);
- `record: true` is fine when the **input** is a record; it also worked for a JSON-String input, but
  **read the recorded file back every time** (a recorded exception, or a wrong document, would become the
  expectation). Compare a recorded XML with the reference document element by element;
- write the boundary cases the rules name: exact half, a value `double` gets wrong, negative, zero,
  absent / empty / blank input, same key written differently, one vs several elements, optional fields
  absent (element omitted), special characters;
- reload before every run, read `testCount` first, then `errorCount`, then `failureCount`;
- prove the suite can fail: break one expectation of each kind (pipeline value, field path, exception
  message), reload, run (`failureCount` must equal the number broken), restore from a backup, reload,
  run green. Keep the suites in the local mirror under `<package>/resources/test/`.

## 7. Sizing

`90-` suggests keeping a service under roughly 350 lines. It is a guideline, not a limit: a service
that builds a nested document legitimately reaches ~450 lines because the same structure is declared in
every `mapSource` / `mapTarget`. Split on steps of logic, never to meet a line count, and report the
sizes at the end.
