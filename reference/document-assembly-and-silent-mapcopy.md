# Document assembly (BP-012) and silent MAPCOPY loss

Reference file, read on demand from `50-nested-documents-and-silent-failures.md`. Verified against a live
Integration Server 12.1.

---

## BP-012 — Building a structured document: assemble whole records, never field by field

- **Context**: producing a nested document (an order, a shipment notice, any XML payload with elements,
  attributes and repeating children) from flat pipeline values.
- **The rule**: FSL assembles structures; it does not mutate them. Build each node as a **complete
  record** with a `set` literal, then **copy whole records and whole lists** into their parent. A `copy`
  targeting an individual field of a record is discarded at compile time (Case A below), so
  field-by-field construction silently produces nothing.
- **Values come from the pipeline via `set (variable)`**: the `(variable)` modifier makes IS evaluate
  `%…%` at runtime. A plain `set` stores the percent signs verbatim.

### Correct pattern

```fsl
MAP {
    comment: "Leaf nodes, each built complete. @-prefixed keys become XML attributes and *body the element text."
    mapSource {
        String currency;
        String amountValue;
    }
    mapTarget {
        record amt;
    }
    set (variable) amt = {`@currency`: "%currency%", `*body`: "%amountValue%"};
}
MAP {
    comment: "Graft the finished node into its parent."
    mapSource {
        record amt;
    }
    mapTarget {
        record DocTotals {
            record `ns:Amount`;
        };
    }
    copy amt -> DocTotals/`ns:Amount`;
}
```
→ `<ns:Amount currency="EUR">100.00</ns:Amount>`

### Rules that follow

1. **One nesting level per literal.** `set D = {a: {b: {c: ""}}};` emits the inner braces as literal text
   into the output. Build each level separately and graft upwards.
2. **Constants must transit through a flat variable.** A literal cannot be written straight into a nested
   field, so hardcoded values (a `Scheme/ID` of `"STD"`, a type code) are `set` into a top-level
   String first, then referenced in the parent's literal.
3. **Never seed a record you intend to graft into.** `set D = {kid: {}};` creates a **String field holding
   the characters `{}`**, and the compiler then refuses to copy a record or list onto a String. Declare
   the structure in a `mapTarget` instead; that is what materialises it.
4. **Inside an accumulating LOOP, drop the record after appending it** — see Case C, or every entry ends
   up holding the last iteration's values.

### Element order is set by the document type, not by the FSL

The order in which elements appear in the generated XML follows the IData, and the IData order cannot be
controlled from the pipeline: values placed by a `set` literal land **after** grafted records regardless
of block order. Verified — no arrangement of MAP blocks changes it.

To emit a schema-valid document, pass a document type to the serialiser:

```fsl
set documentTypeName = "myPackage.docTypes:MyDocType";
```

`pub.xml:documentToXMLString` then orders the output to match that type. The document type has to exist
on the Integration Server; FSL cannot create one. Generating it from a representative XML sample is the
quickest route (`wm.server.record:generateFromXMLString`) — **keep the XML declaration** (without it the
parser rejects the sample as not well-formed) and **remove the default namespace** (otherwise the root
element is typed `prefix0:Root` and will not match the record you serialise). See the sibling
`document-type` skill.

---

## ACTIVE — Silent MAPCOPY loss: three pipeline behaviours that fail without any error

- **Status**: Open. Three distinct cases where a `copy` statement is **discarded at compile time**.
  `dsl_validate` returns SUCCESS, deployment succeeds, the service runs and returns no error — it simply
  does not do what the FSL says.
- **Category**: Transpiler problem (statements silently dropped).
- **How to detect**: the generated node on disk is the only source of truth. Read
  `<IS>/packages/<pkg>/ns/<interface path>/<service>/flow.xml` and grep `MAPCOPY`:
  ```
  grep -o '<MAPCOPY FROM="[^"]*" TO="[^"]*"' flow.xml
  ```
  A `copy` from your FSL with no matching `MAPCOPY` line was rejected. The path suffix `;2;0` vs `;2;1`
  also reveals how the compiler typed each variable — `0` is a single record, `1` is a list.
  `node_get` exposes none of this.

### Case A — `set` on a path inside a record writes nothing

`set Doc/field = "X";` leaves `Doc = {"field": {}}` — the field exists but holds an empty IData, not the
string. It fails at **one** level of nesting, with plain names, no backticks involved, so it is not an
escaping or depth problem.

- ❌ **FORBIDDEN**: `set Doc/field = "X";`
- ✅ **MANDATORY**: build the record whole, then graft it — see BP-012 above.

`set` is only valid on a **top-level** variable (`set v = "X";`) or to create a whole record from a
literal (`set Doc = {field: "X"};`).

### Case B — the target record's NAME decides whether a list copy compiles

Two files identical except for one identifier, both deploying successfully:

```fsl
copy src -> Doc/`ns:Line`;   // → MAPCOPY compiled, works
copy src -> Zed/`ns:Line`;   // → no MAPCOPY at all, target stays null
```

Observed: `Doc`, `Docu`, `Docum`, `Documen`, `Document` all work. `Zed`, `I`, `In`, `Xnvoic`, a
business noun, its truncation and the same noun prefixed by an acronym all fail. It is neither name
length nor any particular word — only names beginning with `Doc` were found to work. `Document`/`document` are real lexer tokens
(`DOCUMENT_DATA`, `DOCUMENT_TYPE` in `AssetLexer.g4`), so the compiler may only resolve a nested-list
copy target when the root identifier prefix-matches a document keyword — **this is a hypothesis, not
established**.

- **Action**: when a `recordList` nested under a `record` is a copy target and the copy silently does
  nothing, **rename the root record to start with `Doc`** (e.g. `Order` → `DocOrder`). Remember the
  record name becomes the XML root element name, so remap it at serialisation:
  `copy DocOrder -> \`document\`/Order;`

### Case C — grafted sub-records are stored by reference

Inside a LOOP whose result is accumulated, a record containing a **grafted sub-record** is stored by
reference. Every accumulated entry then points at the same object, and all of them show the **last**
iteration's values.

```fsl
LOOP {                                   // src = [{v:"A"},{v:"B"}]
    set (variable) leaf = {val: "%cur%"};
    copy leaf -> item/inner;
    INVOKE pub.list:appendToDocumentList { … }
}
```
→ `[{"inner":{"val":"B"}}, {"inner":{"val":"B"}}]` — both entries hold `"B"`.

A record built **flat** in a single literal does not suffer this: the same loop yields the correct
`[{"val":"A"},{"val":"B"}]`.

- **Action**: in the accumulating INVOKE's `output {}` block, `drop` **the accumulated record itself**
  plus every intermediate used to build it — not just `toList`:
  ```fsl
  output {
      mapSource { recordList toList; }
      mapTarget { recordList acc; }
      copy toList -> acc;
      drop toList;
      drop fromItem;
      drop item;        // ← the record that was appended
      drop leaf;        // ← every sub-record grafted into it
  }
  ```
  Dropping the intermediates alone is **not** enough; the appended record must go too, so the next
  iteration builds a fresh one.
