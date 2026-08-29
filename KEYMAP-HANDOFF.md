# Handoff: gummi keymap rework

## What this is
A review of gummi's TUI key bindings found seven conflicts. The scheme and the
full findings are written up here — **read it first**:

  https://claude.ai/code/artifact/8ee17a3b-933b-4281-9184-9c2e04374bf4

Repo `/project`, branch `tui-interaction-rework`. Start from `git log -6` — a
tab shell, a hosted-agent tab, a workspace MCP endpoint, agent-CLI detection
and a batch of race/shuffle fixes all landed today, in that order.

## The scheme, in one paragraph
A key means the same thing everywhere it appears. Three tiers: **global**
(`ctrl+c`, `alt+1/2/3`, `?`) answered before any surface sees the key;
**grammar** (`j/k` `↑↓` `pgup/pgdn` `enter` `esc` `tab`) bound identically by
every surface that has the concept; **verbs** where case carries weight —
lowercase reversible and list-scoped, UPPERCASE creates, approves or destroys.
That last convention already exists in `diffview.go` (`x` resolves an
annotation, `D` deletes one); it simply isn't followed elsewhere.

## Work, worst first
1. **`x` deletes a card on the board.** Everywhere else `x` is the harmless one
   (resolve, dismiss, drop, remove). Four surfaces train the reflex, the board
   punishes it. Move board delete to `D`, matching diffview's existing
   convention. Highest severity because it is destructive.
2. **Nothing switches tabs from inside a view.** `Shell.handleKey` hands the
   whole keyboard to `chat` / `spec` / `diff` / `ingest` / `bugIngest` / `deps`
   before any tab key is considered, so from those surfaces neither `tab` nor
   `alt+1` nor `?` does anything — you must `esc` out first. `alt+1/2/3` are
   also handled in two separate places, neither above the early returns. Hoist
   `alt+1/2/3` and `?` to the top of `handleKey`.
3. **`tab` means five things** — read⇄annotate (spec, diff), next tab (board,
   inbox), form field nav, filter⇄list (bug import), and a keystroke for the
   hosted CLI. Forms are fine (the Overlay answers them first, and tab-between-
   fields is universal) and the hosted CLI is a deliberate exception (it owns
   the keyboard). Resolve spec/diff and the bug import.
4. **The agent tab is outside the `tab` cycle** (`nextTab` in `tabs.go` skips
   it deliberately). This fixes itself once 2 lands: with `alt+N` global,
   landing there is no longer a trap, so `tab` can cycle all three.
5. **Open surfaces outrank the tab they belong to.** `mainView` and `draw` in
   `shell.go` check `m.chat` / `m.spec` / `m.diff` *before* `m.tab`, so with a
   chat open, switching to the inbox still renders the chat. Nobody has hit it
   because switching tabs from a chat is currently impossible — fixing 2
   exposes it. Scope those surfaces to the board tab: hidden while elsewhere,
   restored on return. A chat holds an unsent input buffer, so discarding on
   switch would be its own bug.
6. **Four letters carry unrelated verbs** — `o` (one-liner / repo / own answer),
   `c` (comment / clean up / rename), `r` (rebase / rename), `u` (envelope /
   top up). Lowest priority, most churn. Do them letter by letter as each
   surface is touched, not as one sweep that invalidates all muscle memory.
7. **`q` quits on the board, goes back in views.** Leave it. `esc` is the
   documented way back, and a `q` that quit from inside a spec would be worse.

## The one open decision
Freeing `tab` for cycling leaves spec/diff's read⇄annotate toggle homeless:
- `m` for mode — best mnemonic, free in both views, collides with the board's
  `m` for merge (exactly the class of thing this rework reduces).
- `shift+tab` — free in both, but conventionally means "backwards".
- Keep `tab` in spec/diff — no churn, but `tab` still means two things.
Ask the user before picking.

## Ground rules
- Build `go build ./...`, test `go test ./...`. There is **no `make`**.
- A C compiler is installed, so `go test -race ./...` works and the repo is
  currently clean under it — keep it that way. `go test -shuffle=on ./...` is
  also clean; don't reintroduce shared-state leaks between tests.
- `gofmt -w` what you touch; `gofmt -l internal cmd` must be empty.
- Goldens: `go test ./internal/ui/... -update`. Many are full-screen renders,
  so a status-bar or tab-bar change touches a lot of them — that is expected.
- **The README key table is test-enforced** (`TestReadmeBoardKeysCoverBindings`)
  — change a binding, update `README.md`, or the build fails.
- Match the house comment style: substantial comments explaining *why*, not
  what. `internal/ui/tabs.go`, `internal/ui/agenttab.go` and
  `internal/engine/mcpsock.go` are recent good models.
- Every binding cited above was read out of the live handlers and key tables;
  verify rather than trusting this document if something looks off.

## A warning
Changing bindings breaks muscle memory, and this branch has already moved a
lot of the board. Prefer landing 1 and 2 (safety and reachability) as their own
commit, then 3–5, then leave 6 for later. Do not do all seven in one sweep.
