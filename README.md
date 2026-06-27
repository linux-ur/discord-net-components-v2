discord-net-components-v2


A Claude Skill that teaches Claude (in Claude.ai, Claude Code, or via the API) everything it needs to write and debug Discord bot UIs using Discord.Net's Message Components V2 (ComponentBuilderV2) in C#.


Built from:


Discord.Net's official Components V2 guides — Intro, Advanced, Interaction
Discord's own Component Reference
Discord.Net's v3.18 breaking changes notes


What's inside


discord-net-components-v2/
├── SKILL.md                       — entry point: mental model, nesting cheat sheet, quick start
└── references/
    ├── component-types.md         — full field-by-field spec for every component type
    ├── builder-guide.md           — ComponentBuilderV2 fluent API + a complete worked example
    ├── interactions.md            — catching clicks/selects/modals, FindComponentById, UpdateAsync
    └── troubleshooting.md         — the ComponentsV2 flag, v3.18 breaking changes, common errors



SKILL.md is intentionally short — it carries the decision-making info you need on every task (what nests in what, the builder pattern, the quick-start snippet). The references/ files are loaded on demand for exhaustive detail, full code samples, and error-message lookups.


Installing this skill


Claude.ai / Claude apps: zip the discord-net-components-v2/ folder (or download the release .skill file, if you've packaged one) and upload it under Settings → Capabilities → Skills.


Claude Code: drop the folder into your skills directory, e.g.:


cp -r discord-net-components-v2 ~/.claude/skills/



API: include the skill's contents as part of your tool/skill configuration per the Agent Skills docs.


Once installed, just talk to Claude about Discord.Net Components V2 — buttons, action rows, containers, sections, modals, the ComponentsV2 flag, etc. — and it will consult this skill automatically.


Scope & accuracy notes


This skill targets the Discord.Net guide snapshot at version 3.20.1 (the version referenced in the source docs at the time this skill was written). Discord.Net evolves quickly — if you hit a mismatch (a method that doesn't exist, a new fluent helper that does), trust your installed package's IntelliSense/source over this skill, and consider opening a PR here.
Discord's API reference also documents newer modal-only component types (Label, File Upload, Radio Group, Checkbox Group, Checkbox — types 18, 19, 21–23) that were not yet present in Discord.Net's own documented component table as of this writing. They're noted in references/component-types.md as "API-level, library support unconfirmed."


Contributing


PRs welcome — especially to confirm/correct the "inferred" fluent builder methods (WithSection, WithThumbnail, WithFile, WithSeparator, WithContainer) once you've checked them against a live Discord.Net install, or to add coverage for the newer modal component types once Discord.Net ships them.


License


MIT — see LICENSE.
