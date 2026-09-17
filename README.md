# Gadiel Paiz
 
Software developer working mainly with Flutter/Dart and TypeScript/Firebase.
Outside of work, I'm building Setward, a macOS command-line tool written in Swift 6.
 
## Setward
 
### The problem
 
My MacBook Air gets wiped every few months to recover storage. Each time,
rebuilding my development environment meant remembering which Homebrew
formulae, casks, and taps I had installed, and reinstalling them by hand.
Setward captures that state into named profiles and restores it on demand.
 
### How it works
 
Setward records Homebrew state (formulae, casks, taps) as structured JSON
profiles. To restore a profile, it renders a temporary Brewfile and hands it
to `brew bundle`. Profiles are plain files: they can be inspected, copied, or
kept under version control without Setward installed.
 
### Technical decisions
 
- **Delegate installation to Homebrew.** Setward does not reimplement package
  installation. `brew bundle` already handles dependencies, retries, and
  architecture detection; rebuilding that would add months of work and a new
  class of bugs.
- **JSON as the source of truth, not Brewfile.** A Brewfile is Ruby code, not
  data: reading one safely means evaluating it. JSON can be read and written
  programmatically, validated, and extended with metadata. Brewfiles are
  generated only as an execution format.
- **A reusable core, separate from the CLI.** All domain logic lives in a
  library that knows nothing about terminals, so a planned SwiftUI app can
  consume it unchanged. The executable is a one-line entry point, which keeps
  all command logic unit-testable.
- **Swift 6 with complete strict concurrency.** Data races are compile-time
  errors. Homebrew queries run concurrently, and `brew bundle` output is
  streamed line by line as it arrives.
- **Nothing is destroyed without asking.** Removing an entry, or capturing
  over a profile that already exists, asks for confirmation and supports a
  dry run. With no terminal on standard input, the command fails and points
  at `--yes` instead of guessing an answer, so it can't hang in a script.
- **Errors are not masked.** Removing a package that isn't in a profile is an
  error, not a silent no-op. Creating a profile that already exists fails
  atomically at the filesystem level, with no window between the existence
  check and the write. And a capture writes nothing unless every Homebrew
  query succeeded: a transient failure used to be recorded as an empty list,
  which could overwrite a good profile with a degraded one while reporting
  success.
- **Tests from the first milestone.** Unit, integration, and end-to-end tests
  (Swift Testing) are written alongside each milestone, not at the end. The
  suite currently has 255 tests across 35 suites.
- **Decisions are written down.** Every scope change and design decision is
  recorded in a decision log, with the alternatives considered and why they
  were rejected.
### A bug worth mentioning
 
The `edit` command opens a profile in the user's `$EDITOR`. During
development, terminal editors such as nano stopped the moment they opened.
The process table showed the editor in a stopped state and in a different
process group from the shell: Foundation's `Process` was launching the child
outside the terminal's foreground process group, so the kernel sent it
`SIGTTIN` as soon as it tried to read input.
 
The fix was to spawn the editor with `posix_spawn` in the parent's process
group, the same approach `git` uses. Editor commands with arguments (such as
`code --wait`) run through `sh -c`, with the file path passed as a positional
argument rather than interpolated into the command string, so a path can
never be interpreted as shell code.
 
Automated tests can't reproduce this class of bug without a pseudo-terminal,
so `edit` is verified manually in a real terminal before every merge.
 
### Example
 
```console
$ setward doctor
✓ Homebrew is installed: Homebrew 6.0.22-67-g29b882c Homebrew/homebrew-core (git revision 67891133f7a; last commit 2026-08-09)
 
$ setward create demo
✓ Profile 'demo' created.
 
$ setward add demo jq
✓ Formula 'jq' added to profile 'demo'.
 
$ setward add demo visual-studio-code --cask
✓ Cask 'visual-studio-code' added to profile 'demo'.
 
$ setward show demo
Profile: demo
Taps (0):
Packages (2):
  • [formula] jq
  • [cask] visual-studio-code
 
$ setward install demo --dry-run
brew "jq"
cask "visual-studio-code"
 
$ setward remove demo jq
Remove Formula 'jq' from profile 'demo'? [y/N] y
✓ Formula 'jq' removed from profile 'demo'.
```
 
`install --dry-run` prints the Brewfile that would be passed to
`brew bundle`, without running it.
 
### Status
 
- **v0.2.0** — released, with a prebuilt Apple Silicon binary. Commands:
  `init`, `doctor`, `dump`, `list`, `show`, `install`, `create`, `add`,
  `remove`, `edit`.
- **Next** — continuous integration, a man page, shell completions, and
  colored output.
- **Planned** — profile diff and sync, package search, Brewfile
  import/export, macOS 11 support, and later a SwiftUI app built on the same
  core.
Requires macOS 13 or later. I use it on my own machine; the prebuilt binary
is Apple Silicon only.
 
Built with Swift 6, Swift Package Manager, swift-argument-parser, and
Swift Testing.
 
### Source code
 
The repository is private. I can grant read access on request during a
hiring process: <gadiel.paiz@gmail.com>