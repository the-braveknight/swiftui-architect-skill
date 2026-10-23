# swiftui-architect

This repository is the `swiftui-architect` agent skill. Agents read `skills/swiftui-architect/SKILL.md` and its `references/`; people read `README.md`. It ships as a Claude Code plugin (`.claude-plugin/`) and through the skills CLI, and it is public.

## What the skill is for

- It decides where code lives, who owns it, and the shape of the pieces that hold a modular SwiftUI app together: modules and their dependency direction, domain models and errors, services and the networking client, view models, the views a screen is made of, app composition, navigation and naming.
- It states shapes and rules. How to write the code inside a shape (view bodies, modifiers, layout, isolation, task groups) belongs to the SwiftUI and Swift Concurrency skills listed under Related skills in `SKILL.md`.
- Out of scope: visual design, accessibility, and anything that belongs to one app built with the skill.

## What a change must respect

- Add only what an agent would get wrong without it: a placement, an ownership rule, a shape, a gotcha. Cut explanations of what Claude already knows about Swift, SwiftUI or Xcode.
- State the rule, then a short reason when the reason guides judgment in cases the rule doesn't cover.
- Each rule lives in one reference. `SKILL.md` carries at most one line for it (a principle, a brief, a gotcha or a workflow step), other references link to it instead of restating it, and `conventions.md` holds only the names it uses.
- Examples come from the one small app in `SKILL.md`'s table, a movie browser on the TMDB API, and each idea is shown on the screen that shows it best:
  - one resource: the cast screen
  - several resources and paging: Discover
  - a thing and its paged list, and a mode: a genre's movies
  - a keyed resource: a person's credits by department
  - a nested model and a port: the movie's watchlist button
  - a presentation and a draft: Search's filter

  Reuse these screens, and add one only when none of them can show a new idea. Never bring in names, endpoints, components or screens from the apps the skill is used on; write the rule they illustrate instead.
- Code shows a shape the skill prescribes, such as a view model's state and loads, `XView`'s modifiers or a destination enum. It isn't an implementation to copy: no helper functions, error classifiers, test stubs or component internals. Elide the rest with `…`.
- Native SwiftUI and Apple's frameworks only: no navigation libraries, DI containers or invented infrastructure.
- No invented states: an empty answer is never an error, and a screen shows only the states its design has.
- Testing guidance names the tests a bug would break, never a test for what every instance of a shape repeats.
- Describe the current design only. Never mention old names, renames, migrations or history.
- One term per concept, as `references/conventions.md` names it: feature, screen, a tab's root screen, tab XScreen, pushed XScreen, resource, keyed resource, mode, nested model, place, the shared package, the Live targets, `XDestination`, `SharedDestination`, `Destination`. Names follow the Swift API Design Guidelines and that file's naming table.
- A new rule mustn't contradict an existing one. When it replaces one, change every place the old rule appears.
- Voice: plain declarative sentences in the present tense, lists over paragraphs, no "you", no all-caps emphasis, no hedging.

## How the skill is structured

- `SKILL.md` is the entry point and stays under 500 lines and about 5,000 tokens. After compaction Claude Code re-attaches only the first 5,000 tokens of an invoked skill, so the most important content comes first. Its sections are: the example app, the shape, dependency direction, who owns what, principles, the shapes in brief, gotchas, workflows, references and related skills.
- Gotchas stay in `SKILL.md`, where every invocation reads them. Each one is a mistake an agent actually makes.
- Detail lives in `references/`, one topic per file. Every reference is listed in the References table in `SKILL.md`, saying when to read it, and none is reachable only through another reference.
- A reference over 100 lines starts with a `## Contents` list of its `##` sections.
- Links between files use GitHub heading anchors; renaming a heading means updating every link to it.
- Frontmatter holds only `name` and `description`. claude.ai and the API accept only the open Agent Skills keys (`name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`) and reject an upload with any other, such as Claude Code's `when_to_use` or `paths`. `name` matches the folder. `description` is under 1,024 characters, in the third person, says what the skill does and then when to use it, key use case first.

## Making a change

1. Decide whether it belongs: architecture, ownership or a shape goes in; technique goes to the related skills; one app's detail stays out.
2. Find every place the rule or name appears: `grep -rn "<term>" README.md .claude-plugin skills`.
3. When the change alters what an agent produces, copy `skills/swiftui-architect` to a scratch folder first, as the baseline to compare against.
4. Edit the reference first, then the line in `SKILL.md`, then `conventions.md`, then `README.md` when what the skill covers changes.
5. Keep the descriptions in `SKILL.md`, `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` in step.
6. Verify, and fix until each passes:
   - `wc -l -c skills/swiftui-architect/SKILL.md` stays within the limits above, at about four characters a token
   - `grep` finds no stale name or rule, and no name from an app the skill is used on
   - every link and anchor resolves
   - `claude plugin validate .` passes; its notes that `CLAUDE.md` doesn't ship with the plugin and that there's no icon are expected
7. When the change alters what an agent produces, give two fresh agents the same two or three realistic tasks (add a feature, add a paged screen, link two features, place a shared type), one reading the baseline and one the edited skill. Read both answers, and fix whatever the edited skill still leaves them to guess.

A new reference file gets a lowercase topic name, a Contents list once it passes 100 lines, a row in the References table in `SKILL.md`, and a line in the layout in `README.md`.

## Releases and commits

- The version appears three times, in `plugin.json` and twice in `marketplace.json` (the marketplace and its plugin entry); keep them equal and change them only when releasing.
- Commit or push only when asked. A subject line names the guidance in the imperative, such as "Keep a paged list's pages when its screen comes back", not the files touched.
