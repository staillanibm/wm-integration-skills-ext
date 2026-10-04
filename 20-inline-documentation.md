# Inline documentation (`comment:`)

Applies to Step 6. Overrides the examples in `references/core-syntax-specifications-and-operational-rules.md`
that show `comment:` inside a signature field's property block.

## Mandatory on every generation — not on request

Every generated FSL file documents itself with `comment:` properties, written in the same pass as the
logic, and not conditional on the user asking for documentation. (The opt-in `.md` companion file of
`supplements/service-documentation.md` is a separate concern.)

Add a `comment:` at minimum:

- in the service's own `properties {}` block: a concise statement of its business purpose;
- on any flow step that encodes non-obvious logic: business-rule branches (`IF` / `SWITCH` / `BRANCH`
  conditions that are not self-evident), error-handling steps (`TRY` / `CATCH`, `EXIT` with a failure
  signal — explain *why* it fails, not just that it does), and any step whose purpose is not clear from
  the surrounding FSL.

Do NOT comment steps that are self-explanatory from their structure (a plain `INVOKE` of an obviously
named service with a trivial 1:1 mapping). Comments must carry information the code does not.

Plan these comments during Step 5, alongside the logic, not afterwards.

## Logical sequence segmentation (`SEQUENCE`)

To maintain clean and readable Flow Services in Software AG Designer, structure complex multi-step services into logical `SEQUENCE` blocks with descriptive `comment:` properties:

```fsl
SEQUENCE {
    comment: "=== 1. Input Validation and Parsing ==="
    // Validation & parsing logic
}
SEQUENCE {
    comment: "=== 2. Business Processing & Transformations ==="
    // Loop / mapping logic
}
SEQUENCE {
    comment: "=== 3. Serialization and Final Cleanup ==="
    // Serialization & MAP drops
}
```
This groups execution steps into clear functional phases when inspecting the service tree in Designer.

## Never inside a signature field's property block

```fsl
String interestRate {
    allowNull: false;
    required: true;
    // comment: "..." is NOT allowed here
};
```

The grammar (`grammar/Document.g4`, `simpleFieldPropertyEntry`) declares `comment` valid there, but the
FSL compiler deployed on the target Integration Server rejects it. This is a known grammar/compiler
divergence, not a mistake in the grammar file. Omitting an optional property keeps the FSL grammatically
valid, so this does not conflict with the grammar-validity rule.

Field-level documentation goes in the `.md` companion file (Inputs/Outputs tables of
`supplements/service-documentation.md`), never as an inline property on a signature field. Ignore any
example in the references that shows `comment:` on a field.
