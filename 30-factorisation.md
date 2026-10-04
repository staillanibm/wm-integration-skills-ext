# Detecting factorisation opportunities

Applies to Step 5. Add this to the internal model: whether part of the logic should become a separate
reusable flow service instead of being repeated inline. Detecting it is your responsibility — the user
describes *what* they need, not the service breakdown. Deciding is theirs.

## Signals worth raising

- **The same sequence of steps recurs.** Two or three repetitions of an identical INVOKE chain (e.g.
  compute → format → substitute, applied to every monetary amount). Every repetition is a place the
  next change must be made again.
- **A self-contained transformation with a narrow signature.** One or two scalars in, one value out, no
  dependency on the surrounding pipeline — and unit-testable on its own.
- **A step whose failure deserves its own error message.** Isolating it makes the failure attributable.
- **A rule likely to be reused by a sibling service.** Formatting, validation and code-mapping rules are
  rarely needed only once in a package.

## Signals to leave alone

- Logic used exactly once with no prospect of reuse.
- A sequence only superficially similar — same shape, different intent. Merging couples two rules that
  will diverge.
- Anything needing more than a handful of inputs, or that would have to reach back into the caller's
  pipeline. A helper with eight parameters is not a simplification.

## How to raise it

Before generating, state plainly: what you would extract, why (which signal), what its signature would
be, and how many call sites it would have. Then let the user decide. Proceed on their answer; if they
decline, generate inline without further comment.

**Never split a service into sub-services silently.** An unapproved factorisation changes the namespace
the user will maintain and adds assets to their package. When the prompt already specifies the
breakdown, follow it and do not propose an alternative unless it is technically impossible.

This is consistent with ZERO-INFERENCE (`operational-rules-and-code-construction-patterns.md` §10):
that rule governs what you *emit* unasked. Raising a factorisation in Step 5, before any output exists,
is how an extraction becomes explicitly requested. Spotting the pattern is analysis; generating it
unapproved would be inference.

If the user approves, generate each service as its own complete FSL file under its own
`{ServiceName}/flow.flow` path (see `10-package-and-output-layout.md`), with a signature that stands on
its own — it will be invoked by name, not inlined.
