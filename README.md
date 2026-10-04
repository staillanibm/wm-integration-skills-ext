# extended-skill/ — drop-in customization folder

Put your customizations here instead of editing `SKILL.md`.

How it works:
- The skill loads **every `.md` file in this folder, in alphabetical order**,
  before generating, and applies their contents (Step 1 of `SKILL.md`).
- `custom-instructions.md` is the primary preferences file (apply text under its
  "Your instructions" heading). Add any other `.md` files for extra standards,
  conventions, or examples.
- **This `README.md` is skipped** by the skill on purpose — it is documentation,
  not an instruction.

Conflict rules:
- If two custom files disagree, the one later in alphabetical order wins.
- Custom instructions override the skill's *conventions* (naming, folders,
  formatting, which actions to offer).
- They do NOT override correctness or safety rules: service-metadata validation,
  grammar validity, 0-based FlatFile positions, the supported MAP operations
  (copy / copy-if / set / drop), and producing deployable code. If a preference
  conflicts with one of these, the skill follows the correctness rule and tells
  you why.

If this folder is empty or absent, the skill just uses its defaults.
