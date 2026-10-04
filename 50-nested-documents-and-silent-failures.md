# Nested documents and silent MAPCOPY loss

Applies to Step 5 whenever the service builds a nested structure: an XML payload, a document with
repeating children, any record more than one level deep.

**Before planning the mapping, read `extended-skill/reference/document-assembly-and-silent-mapcopy.md`
in full.** It is not loaded automatically (it lives in a subfolder). It holds best practice BP-012 and
three verified failure modes, none of which produce an error.

## Why it matters

FSL assembles whole records; it never mutates fields. Writing it the other way fails **silently**: the
compiler discards the statement, `dsl_validate` still returns SUCCESS, deployment succeeds and the
service runs without error — it just does not do what the FSL says.

## The rules, in short

1. **Build each node as a complete record with a `set` literal, then `copy` whole records/lists into the
   parent.** Never `set Doc/field = "X";` — that leaves an empty IData, not the string.
2. **Values from the pipeline use `set (variable)`.** A plain `set` stores the `%` signs verbatim.
3. **One nesting level per literal.** `{a: {b: {c: ""}}}` emits the inner braces as text.
4. **Constants transit through a flat top-level String** before being referenced in a parent's literal.
5. **Never seed a record you intend to graft into** (`set D = {kid: {}}` creates a String holding `{}`).
   Declare the structure in `mapTarget`.
6. **Inside an accumulating LOOP, `drop` the appended record and every grafted sub-record**, not only
   the list — otherwise every entry holds the last iteration's values.
7. **A nested-list copy that silently does nothing may depend on the root record's name.** Only names
   starting with `Doc` were observed to work (hypothesis, not established). Rename the root to
   `DocXxx` and remap at serialisation.
8. **Element order in generated XML comes from a document type, not from the FSL.** To emit
   schema-valid XML, set `documentTypeName` for `pub.xml:documentToXMLString`. The document type must
   exist on the server; see the sibling `document-type` skill.

## Detecting a silent loss

The compiled node on disk is the only source of truth:

```
grep -o '<MAPCOPY FROM="[^"]*" TO="[^"]*"' <IS>/packages/<pkg>/ns/<interface path>/<service>/flow.xml
```

A `copy` in your FSL with no matching `MAPCOPY` line was dropped. Path suffix `;2;0` = single record,
`;2;1` = list. `node_get` exposes none of this. When the service builds nested documents and the
Integration Server package folder is reachable locally, do this check before reporting success.
