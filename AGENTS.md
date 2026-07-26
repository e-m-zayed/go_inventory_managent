# Project
- inventory managenent app.
- refrence:
    - https://github.com/inventree/InvenTree
    - it should be used only as a refrence for features and structure
    - the language and the stack are diffrent.
- InvenTree was just one refrence you should search for more refrences.
- the domain is inventory and warehouse management.
- we want our app to be thorough and cover the most widespread use cases regaredless of their scale. -> ease of use is a priority

# Stack
- golang -> we are using the standard library for the backend 
- we will minimize the use of external libraries as much as possible.
- htmx: https://github.com/donseba/go-htmx
- daisyui: https://daisyui.com/htmx-component-library/?lang=en
- we are using Ent as an orm: https://github.com/ent/ent , https://entgo.io/docs/tutorial-setup/.
- barcode + QR generation and scanning: https://github.com/makiuchi-d/gozxing (pure-Go ZXing port; chosen by the user as the approved external package for both encoding and decoding across QR and 1D formats).


# Localization

- The system will be multi language -> all languages are first class.
- we are starting with English and Arabic as a beginning and adding more later
- we are using i18n: https://github.com/nicksnyder/go-i18n .
- each language localization should be in toml file.
- for a start the app will be only english -> we will add more later.
- the ui logic should be bi-directional from the start -> neither RTL and LTR should be harcoded.



# building

- all the run, build, test and similar processes should be handled by magefile build system.
- after every phase the agent should do a clean build.
- we are doing a local binary no docker nor complicated containers.

# git discipline
- every minor change should be tracked with git closely.
- one logical change per commit. if a change spans multiple files but is one decision, that is one commit. if two unrelated edits land together, split them.
- commit messages follow Mitchell Hashimoto's style, verified against his actual commits in hashicorp/go-plugin and similar repos. the rules:

  **subject line**
  - lowercase or sentence-case, never all caps.
  - imperative or declarative mood, present tense ("add SyncStdio", "interrupts should not be eaten in test mode", "Serve owns opts.Listener").
  - no conventional-commit prefixes: no `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`.
  - no scope suffixes like `barcode:` or `todo:`. the subject describes the change, not the area.
  - ≤72 characters, hard limit.
  - short is good. "add SyncStdio", "Note on exit", "fix typo in server error" are real Mitchellh subjects. prefer 3-6 words when the change is small.
  - no trailing period.

  **body**
  - optional. omit it entirely when the subject makes the change self-evident (most small commits).
  - when present, 1-2 short prose paragraphs explaining *why* the change exists or *what* behavior shifts. never enumerate files touched, never enumerate tasks completed, never reference task IDs.
  - wrapped at ~72 columns manually.
  - no bullets, no trailers (no `Co-Authored-By`, no `BREAKING CHANGE`, no `Refs:`) unless the user explicitly asks.
  - tone is direct and technical. example from go-plugin, commit ffa11a1:
    ```
    Add a test mode

    This is a special configuration on the plugin server (the plugin side)
    that enables running the plugin in-process and eases many of the
    behaviors around that.

    The desire to do this is for more easily using `go test` (and similar)
    with plugins. This mechanism allows the plugin serving to start up
    in-process, to grab a ReattachConfig, and to be able to send that
    ReattachConfig to some other plugin client for connection.
    ```

  **what the body never contains**
  - file paths and per-file summaries ("AGENTS.md: ... SPEC.md: ... tasks/plan.md: ...").
  - task IDs ("Task 08.6", "P0.3").
  - verification evidence ("mage test exit 0", command output snippets). that belongs in the PR description or a comment, not the commit.
  - meta-commentary about the commit itself ("this is a follow-up commit", "one-line change").
  - the word "note:" as a trailer.

  **enforcement**
  - before `git commit`, re-read the staged diff and ask: would Mitchellh write this subject? is the body necessary? if the body just lists files or tasks, delete it and rely on the subject alone.
  - if the change is too big to describe in one short subject, the commit is too big. split it.




# Pitfalls
- Never suggest other librairies and packages that are not part of the stl outside the ones we are using.
- Don't botch up the bi-directional ui and the loclaization.
- Don't do a change without a thorough check of the code and doing a web research.
