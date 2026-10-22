# swiftui-architect

This repository is the `swiftui-architect` agent skill. Agents read `skills/swiftui-architect/SKILL.md` and its `references/`; people read `README.md`. It ships as a Claude Code plugin (`.claude-plugin/`) and through the skills CLI, and it is public.

## What the skill is for

- It decides where code lives, who owns it, and the shape of the pieces that hold a modular SwiftUI app together: modules and their dependency direction, domain models and errors, services and the networking client, view models, the views a screen is made of, app composition, navigation and naming.
- It states shapes and rules. How to write the code inside a shape (view bodies, modifiers, layout, isolation, task groups) belongs to the SwiftUI and Swift Concurrency skills listed under Related skills in `SKILL.md`.
- Out of scope: visual design, accessibility, and anything that belongs to one app built with the skill.

## What a change must respect

- Add only what an agent would get wrong without it: a placement, an ownership rule, a shape, a gotcha. Cut explanations of what Claude already knows about Swift, SwiftUI or Xcode.
- State the rule, then a short reason when the reason guides judgment in cases the rule doesn't cover.
- IMPORTANT: Examples use the skill's own running example, a movie browser on the TMDB API. Never bring in names, endpoints, hosts, components, helpers or screens from the apps the skill is used on; write the rule they illustrate instead.
- Code shows a shape the skill prescribes, such as a view model's state and loads, `XView`'s modifiers or a destination enum. It isn't an implementation to copy: no helper functions, error classifiers, test stubs or component internals. Elide the rest with `…`.
- Native SwiftUI and Apple's frameworks only: no navigation libraries, DI containers or invented infrastructure.
- No invented states: an empty answer is never an error, and a screen shows only the states its design has.
- Describe the current design only. Never mention old names, renames, migrations or history.
- One term per concept, as `references/conventions.md` names it: feature, screen, resource, keyed resource, mode, nested model, place, `XDestination`, `SharedDestination`, `Destination`. Names follow the Swift API Design Guidelines and that file's naming table.
- A new rule mustn't contradict an existing one. When it replaces one, change every place the old rule appears.
- Voice: plain declarative sentences in the present tense, lists over paragraphs, no "you", no all-caps emphasis, no hedging.

## How the skill is structured

- `SKILL.md` is the entry point and stays short: under 500 lines and about 5,000 tokens. The most important content comes first, because Claude Code keeps only the first 5,000 tokens of an invoked skill after compaction. Its sections are: the shape, dependency direction, who owns what, principles, the shapes in brief, gotchas, workflows, references and related skills.
- Detail lives in `references/`, one topic per file. Every reference is listed in the References table in `SKILL.md`, saying when to read it, and none is reachable only through another reference.
- A reference over 100 lines starts with a `## Contents` list of its `##` sections.
- Links between files use GitHub heading anchors; renaming a heading means updating every link to it.
- Frontmatter: `name` matches the folder. `description` is under 1,024 characters, written in the third person, says what the skill does and then when to use it, key use case first.

## Making a change

1. Decide whether it belongs: architecture, ownership or a shape goes in; technique goes to the related skills; one app's detail stays out.
2. Find every place the rule or name appears: `grep -rn "<term>" README.md .claude-plugin skills`. A rule usually lives in one reference, a line in `SKILL.md` (the shapes in brief, gotchas or a workflow) and a row in `conventions.md`.
3. Edit the reference first, then `SKILL.md`, then `conventions.md`, then `README.md` when what the skill covers changes.
4. Keep the descriptions in `SKILL.md`, `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` in step.
5. Verify, and fix until each passes:
   - `wc -l -c skills/swiftui-architect/SKILL.md` stays within the limits above
   - `grep` finds no stale name or rule
   - every link and anchor resolves
   - `claude plugin validate .` passes
6. When a change alters what an agent produces, try it in a fresh session on a realistic task (add a feature, add a paged screen, link two features) and compare with the skill turned off.

A new reference file gets a lowercase topic name, a Contents list once it passes 100 lines, a row in the References table in `SKILL.md`, and a line in the layout in `README.md`.

## Releases and commits

- The version appears three times, in `plugin.json` and twice in `marketplace.json` (the marketplace and its plugin entry); keep them equal and change them only when releasing.
- Commit or push only when asked. A subject line names the guidance in the imperative, such as "Keep a paged list's pages when its screen comes back", not the files touched.
