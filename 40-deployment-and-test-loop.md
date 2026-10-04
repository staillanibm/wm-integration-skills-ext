# Deployment and the deploy, reload, test loop (added as Step 7)

Applies when the task includes deploying the service to the Integration Server, not just writing the
local `.flow` file. Runs after Step 6. Complements `50-nested-documents-and-silent-failures.md`.

A service is not done when it is deployed, nor when its first invocation succeeds: it is done when it
has been tested **after a package reload**.

## Why

On Integration Server 12.1, the node that `fsl_deploy` leaves **in memory** can differ from the node
that is **persisted** to `flow.xml`. Both the deployment and the first invocations succeed; the
difference only appears after `package_reload` (or a server restart). Observed differences, none of
which raises an error:

- a `copy` silently absent from the compiled flow;
- a list that keeps one element instead of all;
- a record shared between two steps.

## Preferred deployment: copy the `.flow` into the package, then reload

When the Integration Server packages directory is reachable from the workspace (local IS), deploy by
**copying the local `flow.flow` to `<IS>/packages/<package>/ns/<interface path>/<service>/flow.flow`
and reloading the package**. The server recompiles the service from that file and rewrites `node.ndf`,
`flow.xml` and `.fsl.digest`. Compared with `fsl_deploy`:

- the **signature is kept** (`fsl_deploy` persists an empty `sig_in`/`sig_out`: after a reload the
  service still runs but has no declared input/output; patching `node.ndf` does not help, the reload
  rewrites it);
- an existing service is **overwritten**, with no `node_delete` and no warning step needed: the file that
  runs is the file in the workspace;
- a new service only needs its folder with the single `flow.flow`;
- the file that runs is not a copy pasted into a tool call (no drift, and no multi-thousand-token echo
  per deployment).

## Fallback: `fsl_deploy`, when the packages directory is not reachable

Use it only then (remote server, no filesystem access), and accept that the signature must be restored
another way (Designer); say so in the report.

`fsl_deploy` is **create-only**: deploying a service whose node already exists fails outright, whatever
its own description claims. Before deploying, check whether a node exists at the target interface path
and service name (`node_get`):

- **Not present** → deploy directly.
- **Present** → warn the user that the deployed service will be overwritten and that any change made
  directly on the server (e.g. in Designer) since the last deployment from this source will be lost.
  Then delete the existing node (`node_delete`) and redeploy from the current `.flow` source. The warning
  is informational, not blocking, unless the user's own instructions for the task say otherwise.

## The loop (for every service, every time it changes)

1. Deploy: copy the `flow.flow` into the package folder (preferred), or the `fsl_deploy` fallback above
   (`dsl_validate` first, then `fsl_deploy`).
2. `package_reload`.
3. Check that the service really compiled, then check the persisted node (below).
4. Run the service or its test suite. Only now read the result.

Never batch several services before reloading and testing: a defect in one hides behind another.

## Tooling

The procedure does not depend on the tool. When the `wm-mcp-server` MCP server is connected (scopes
`fsl-develop` and `unit-test`), the operations map to:

| Operation | MCP tool |
|---|---|
| Copy the `.flow` into the package | filesystem copy (not an MCP tool) |
| Validate FSL without writing | `dsl_validate` |
| Check whether a node exists | `node_get` / `node_list` |
| Delete an existing node (fallback only) | `node_delete` |
| Deploy FSL (fallback only) | `fsl_deploy` |
| Read a deployed service back as FSL | `fsl_extract` |
| Create a missing folder / package | `folder_create` / `package_create` |
| Reload a package | `package_reload` |
| Invoke a service | `service_invoke` |


## Check the persisted node

- `package_reload` can answer "reloaded" even when a service did not compile: see
  `95-fsl-platform-pitfalls.md` §1 (check that `flow.xml` and `node.ndf` exist next to the `flow.flow`,
  read `server.log`).
- Check the signature fields in `node.ndf` against the FSL.
- Compare the FSL with the compiled flow. The persisted flow is at
  `<IS>/packages/<package>/ns/<interface path>/<service>/flow.xml`. Every FSL `copy a -> b;` must appear
  as `<MAPCOPY ... FROM="/a;..." TO="/b;..."`. A count is a good first check, the list of pairs is the
  real one:

```python
import re
norm = lambda p: re.sub(r';\d;\d', '', p.strip('`')).strip('/')
fsl = re.findall(r'^\s*copy\s+(.+?)\s*->\s*(.+?);', open('flow.flow').read(), re.M)
xml = [(norm(a), norm(b)) for a, b in
       re.findall(r'<MAPCOPY NAME="MapCopy" FROM="([^"]*)" TO="([^"]*)"', open('flow.xml').read())]
lost = [(a, b) for a, b in fsl if (a.strip('`'), b.strip('`')) not in xml]   # must be empty
```

A lost `copy` means the compiler could not resolve a type: see `75-mapping-patterns-that-survive.md`.

A successful deployment means the node was written, not that the service behaves as intended: confirm by
invoking it, and see `50-nested-documents-and-silent-failures.md` for failures that neither validation
nor deployment report.

## Things that look like bugs and are not

- A service deployed with `fsl_deploy` has an **empty signature** in `node_get`, `fsl_extract` and
  `node.ndf` after a reload, and `service x ( )` in the server's `flow.flow`. It still runs (the
  pipeline is dynamic) but exposes no declared interface: a defect of that deployment path, avoided by
  copying the `.flow` file. Check `node.ndf` for `field_name` entries after every deployment.
- `fsl_extract` returns a semantically equivalent text, not the source: do not diff it against the
  `.flow` file.

## Cost of a deployment

`fsl_deploy` echoes the whole source and the whole compiled signature: several thousand tokens per call,
doubled by the text you have to send. Keep each service small, deploy once per change, and debug with
small probe services (`zzProbe`, deleted afterwards) rather than by redeploying the real one.

## Diagnosing without a debugger

- `service_invoke` returns the **whole pipeline** at the end of the service, including leaked variables:
  this is the cheapest way to see what a step left behind.
- To see the pipeline at the point of a failure, wrap the body of a probe copy in
  `TRY { ... } CATCH { MAP { set failed = "yes"; } }`: the variables written before the failure stay
  visible in the response.
- Keep the probe as a separate service and delete it when done.

**A probe service is the last resort, not the first.** Before deploying one:

1. Re-read the FSL of the failing service and calculate the expected intermediate values by hand.
2. Check that the correct IS services are being called with the right parameter names (see the service
   catalog — output names such as `value`, `num`, `string` are frequently confused).
3. Only if step 1 and 2 do not identify the cause, deploy a minimal probe that isolates the one
   suspicious step.

Deploying a probe to diagnose a bug that is readable in the FSL wastes deployment tokens and pollutes
the package. Delete the probe immediately after diagnosis.
