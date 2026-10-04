# Package awareness and output layout

Applies to Step 5 (internal construction) and Step 6 (output) of `SKILL.md`. This file replaces the
default output path `output/{ServiceName}/flow.flow` and the matching line in the Output Contract.

Requires the sibling `package` skill (`../../package/SKILL.md`).

## Target package (Step 5)

Add these items to the internal model:

- **Target package name.** If the prompt does not give one, STOP and ask — do not guess or default it
  (no "Default", no "MyPackage").
- **Local layout check.** The package has its standard layout in the workspace when a
  `{packageName}/{rootDir}/` folder exists with `apis/`, `docTypes/`, `services/`, `triggers/` beneath
  it. `{rootDir}` is the package's root namespace folder: it defaults to the package name but may be
  different (see the `package` skill).
- **If the layout is absent, invoke the `package` skill first** to create the package (on the
  Integration Server and as a local mirror). Do not start FSL generation until it reports success, and
  keep whatever `{rootDir}` it actually used.
- **Folder path within `services/`** (optional, may be empty). If the prompt does not give one, ask or
  default to empty — never invent folder names.

## FSL `interface` declaration

Always `{rootDir}.services` followed by the optional folder path, dot-separated:

| rootDir | folder path | interface |
|---|---|---|
| `myPackage` | — | `interface myPackage.services` |
| `myPackage` | `orders` | `interface myPackage.services.orders` |
| `acmeOrders` | `orders` | `interface acmeOrders.services.orders` |

## Output path (Step 6 and Output Contract)

```
{packageName}/{rootDir}/services/{optional folder path}/{ServiceName}/flow.flow
```

That is `{packageName}/` followed by the interface path as nested folders, one directory per
dot-separated segment. `{ServiceName}` is the unqualified service name. Create any missing folder.

- `myPackage`, rootDir `myPackage`, no folder, `myService` → `myPackage/myPackage/services/myService/flow.flow`
- `myPackage`, rootDir `acmeOrders`, folder `orders`, `myService` → `myPackage/acmeOrders/services/orders/myService/flow.flow`

**The interface declaration and the on-disk path MUST match segment for segment**, like a Java
`package` statement and its directory. Never let them diverge.

Companion files written by supplements (documentation `.md`, diagram `.mmd`, `tests/`) go in the same
service folder as `flow.flow`, not somewhere else. Exception: on the *server*, test files use a
different layout — see `60-unit-testing.md`.
