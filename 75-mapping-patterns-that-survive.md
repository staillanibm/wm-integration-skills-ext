# Mapping patterns that survive persistence

Applies to Step 5 (construction) and Step 6 (output). Complements
`50-nested-documents-and-silent-failures.md`. All rules below were observed on Integration Server 12.1
and fail **silently**; they are only visible after `package_reload` (see
`40-deployment-and-test-loop.md`).

## 1. Prepare INVOKE inputs in a MAP, then INVOKE without mapping

Typed record or record-list copies inside an INVOKE `input`/`output` mapping are not reliable: the
compiler sees the callee's own (often empty) signature and drops what it cannot type. Instead:

```fsl
MAP {                                   // prepare under the callee's parameter names
    mapSource { recordList sourceRows { String id; String rate; String price; }; }
    mapTarget { recordList documents  { String id; String rate; String price; }; }
    copy sourceRows -> documents;
}
INVOKE pub.document:groupDocuments {    // no mapping: the pipeline is passed by name
    comment: "..."
}
```

Flat `String` mappings inside an INVOKE are fine. Declared outputs are only needed for flat values you
rename.

## 2. Read nested paths in a MAP, into flat variables

Inside a `LOOP`, a path such as `parent/list/field` read directly in an INVOKE input mapping yielded
nothing. Read it in a MAP with the full path in backticks, then use the flat variables:

```fsl
MAP {
    mapSource { record order { recordList items { String id; String quantity; }; }; }
    mapTarget { String itemId; String itemQuantity; }
    copy `order/items/id` -> itemId;
    copy `order/items/quantity` -> itemQuantity;
}
```

The same goes for a field of an in-flow record (`copy order/orderNumber -> x` inside an INVOKE
mapping was dropped; inside a MAP it was kept).

## 3. Build record lists with appendToDocumentList, not with a LOOP outputArray

`outputArray` collects a record as a **list of one-element lists** (`[[{...}],[{...}]]`). `String[]`
outputs are fine. For records:

- build the item under the name `fromItem`, the accumulator is `toList`;
- `INVOKE pub.list:appendToDocumentList` **with no mapping**;
- after the loop, `copy toList -> yourList` in a MAP, then `drop toList`.

The parameters are `toList`, `fromList`, `fromItem` (not `fromDocument`: that name is silently ignored).
Declaring `toList`/`fromItem` in an INVOKE mapping with empty structures lost the second element: keep
the INVOKE mapping-free.

## 4. One name, one structure, and clean variables

- **A variable name must keep one structure within a service.** Declaring `documents` as
  `{id, rate, price}` and later as `{key, total, share}` made the compiler drop the second copy.
  Isolate the second use in its own small service.
- **A variable dropped then recreated by a mapping-free INVOKE has no known type** for the compiler:
  a later `copy` from it is dropped. Do not `drop` it between two uses; drop it after the last one.
- **Start from clean accumulators.** The pipeline is shared with the caller: a `toList` left by the
  caller is appended to; a record such as `ns:Party` left by a previous call is written into, which
  modifies the record another variable still references (two parties ending up identical). Every
  service that accumulates or builds begins with `drop toList` / `drop <record it builds>`, and a
  service that calls an accumulating service drops its own accumulator first.
- **Do not `drop` a top-level list whose nested copies you still loop on.** After
  `pub.document:groupDocuments`, dropping the top-level `documents` empties the `documents` inside every
  `documentGroups` entry.

## 5. Syntax that compiles, and syntax that does not

| Need | Write |
|---|---|
| list of records | `recordList items { String a; };` (no `record[]`, no `items[]`) |
| name with `:`, `@`, `*` | backticks: `` String `ns:ID`; ``, `` `@currency` ``, `` `*body` `` |
| path with 3+ segments | whole path in backticks: `` copy `a/b/c` -> x; `` |
| path on both sides of a `copy` | backticks on the path(s): `` copy `a/b` -> `c/d`; `` (two plain paths do not parse) |
| attribute / text of an element | record with `` `@attr` `` and `` `*body` `` fields |
| constant | flat top-level `String`, set in its **own** MAP, then copied |

## 6. `set` and `copy` in one mapping block

The `set` is applied **after** the `copy` into the same target: `set inString = ""; copy x -> inString;`
always leaves `""`. To default a value that a copy may leave unset (copy of an absent field leaves the
target absent), seed it in a **separate MAP before**.

## 7. Absent inputs

`copy` of an absent field leaves the target unset, and most `pub.*` services then fail with
`key[...] does not exist in the pipeline`. Seed the target with an empty String in a preceding MAP
whenever the source may be absent (an absent payload must be reported like an empty one).
