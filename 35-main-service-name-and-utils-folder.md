# Main service name, and utility services in `utils`

Applies to Step 5 (internal construction) and Step 6 (output) of `SKILL.md`. Complements
`10-package-and-output-layout.md` (layout) and `30-factorisation.md` (when to extract). Where this file and
`30-factorisation.md` disagree on where an extracted service goes or on what to check first, this file wins.

## 1. Name of the main flow service

Add to the internal model: the name of the **main** flow service, i.e. the entry point the request is about.

- If the prompt gives it, use it as is.
- If it does not, **STOP and ask** before generating anything — do not derive it from the business subject, do
  not default it (`main`, `process`, `myService`...). Ask together with the other blocking questions (target
  package, folder path), in a single message.
- The name must satisfy the naming standard (`supplements/code-review.md`, rule NS6 and neighbours).
- Utility services (below) are named by their own responsibility, not after the main service.

## 2. Utility services go in a `utils` subfolder

Every flow service that is not the main service and that the main service (or another utility) calls is a
**utility service**. Utility services are created in a `utils` folder, never next to the main service.

| What | Interface | Local file |
|---|---|---|
| main service | `{rootDir}.services{.folder path}` | `{packageName}/{rootDir}/services/{folder path}/{Name}/flow.flow` |
| utility service | `{rootDir}.services.utils` | `{packageName}/{rootDir}/services/utils/{Name}/flow.flow` |

- `utils` sits directly under `services/`, so that it is shared by every main service of the package and the
  reuse check below has a single place to look.
- Create the `utils` folder on the Integration Server (parent first) and in the local mirror
  before generating the first utility service. Do not create other sub-folders for utility services.
- The interface declaration and the on-disk path must match segment for segment, as in
  `10-package-and-output-layout.md`.
- This changes *where* an extracted service goes. Whether to extract at all remains governed by
  `30-factorisation.md` (raise it, let the user decide; follow a breakdown the prompt already specifies).

## 3. Reuse existing utility services before creating new ones

Before proposing or generating any utility service, check whether one already exists that does the job. This
is part of Step 5, done **before** the factorisation proposal, so that the proposal says what is reused and
what is new.

Where to look, in this order:

1. **The target package's `utils` folder**, locally (`{packageName}/{rootDir}/services/utils/`) and on the
   server (list the content of `{rootDir}.services.utils`). The server is the source of truth: the two may differ.
2. **The `utils` folders of the other packages** on the server (list the packages, then the content of each
   package's `utils` folder), when step 1 found nothing suitable.

How to decide that a service can be reused:

- Read its signature and its documentation (`comment:` in `properties`, or read it from the server, as metadata or
  as FSL, when only the server copy exists). Match on **behaviour**, not on the name.
- Reuse it only if its inputs and outputs fit without adapting the existing service and without bending the
  caller's logic. A near match is not a match: do not modify an existing utility service to make it fit,
  and do not copy it under another name.
- When several fit, prefer the one in the target package.

What to do with the outcome:

- **Found in the target package**: call it, and say so in the plan.
- **Found in another package**: reusing it creates a dependency between packages. Do not call it silently: put
  it in the plan and let the user decide (depend on it, or create a local utility service).
- **Nothing suitable**: create the new utility service in `utils` (section 2).

Say plainly, in the plan and again in the final report, which utility services were **reused** (with their
full path) and which were **created**, so the user can see that the check was done. An empty `utils` folder
is a valid result of the check; state it.
