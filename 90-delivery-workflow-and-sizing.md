# Delivery workflow, sizing and XML generation

Applies to a request that asks for a complete deliverable (service, assets, tests), not to a single
service. Complements `10-package-and-output-layout.md` and `30-factorisation.md`.

## Order of work

1. **Read the specification and the reference files**, then propose a plan **before** writing
   anything. The plan lists the services, their signatures and call counts, the assets to generate,
   the tests, and the questions that block (package name, root folder, permission to use the
   server). Factorisation is proposed there (`30-factorisation.md`): the user decides.
2. **Package and layout** (`10-package-and-output-layout.md`).
3. **Document type**, if the output is XML that must be schema-ordered (below).
4. **Services, one at a time**, bottom-up (helpers first): write, deploy, reload, test
   (`40-deployment-and-test-loop.md`). Probe any unfamiliar `pub.*` service by invoking it
   before building on it: signatures, parameter names and error messages are not
   always what the documentation says.
5. **Unit test suites** per service, then end to end (`60-unit-testing.md`, `85-unit-testing-addenda.md`).
6. **Replayable recipe** and a short README with the layout, the design choices and the traps met.

## Sizing: lines are a poor proxy

In FSL, type declarations dominate: every MAP repeats the structures it touches, so a service with a
few dozen lines of logic easily reaches 300 lines. A limit expressed in lines will be broken by
declarations, not by logic. Therefore:

- announce in the plan that the number of services will probably exceed the first estimate, and why;
- decide the split on **steps of logic** (one loop, one external call chain, one assembly), keeping a
  service under roughly 350 lines including declarations;
- a service that builds a structure and another that serialises it are two services when both are
  non-trivial;
- a helper that isolates a variable name carrying two structures is legitimate (`75-mapping-patterns-that-survive.md`),
  say so in the report;
- when the count grows beyond the plan, say it explicitly at the end and offer the merge options.

## Generating XML in document order

Element order in `pub.xml:documentToXMLString` comes from the document type given in
`documentTypeName`, not from the FSL. The document passed in `document` must contain **one** top-level
record, whose name is the XML root element.

- Generate the document type from a **sample that repeats every repeating element at least twice**
  (`wm.server.record:generateFromXMLString`: `xmlString`, `packageName`, `ifcName`, `recordName`). Check
  the dimension of the generated lists afterwards.
- Keep the XML declaration in the sample, remove the default `xmlns` from its root. Declare the default
  namespace at serialisation time with an `@xmlns` String field on the root record; declare the prefixes
  with `nsDecls` (a record, one String per prefix).
- Attributes are `` `@name` `` fields, element text is `` `*body` ``. An absent optional field simply
  produces no element.
- Keep the sample in `sources/` with a header comment saying why it is written that way.
- Compare the produced document with the reference, element by element, before recording it as an
  expected result.

## Report at the end

State plainly: what was built, what is verified and how (persisted version, number of tests, recipe),
what deviates from the plan and why, what could not be verified (for example XSD validation when no
schema is available), and the platform behaviours that future changes must respect.
