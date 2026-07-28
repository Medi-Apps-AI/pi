# Fork notes

This fork exists to make pi work with AssemblyAI's LLM Gateway
(`https://llm-gateway.assemblyai.com/v1`). We control neither pi
upstream nor the gateway, so the glue lives here. Keep the footprint
minimal so rebasing onto upstream stays cheap: no upstream doc edits,
no refactors, only the changes listed below.

## Delta vs upstream

All changes are confined to `packages/ai` and the generated
`packages/coding-agent/schemas/models.schema.json`, plus CI:

- `fix(ai): AssemblyAI LLM gateway compat`: everything the gateway
  needs from `packages/ai`, in one commit so upstream churn in
  `openai-completions.ts` stops a rebase at most once:
  - Streaming adapter: skip the empty `tool_calls` skeleton frame the
    gateway's Anthropic adapter emits before plain-text answers
    (materializing it produced a phantom empty-named tool call that
    400/500'd the follow-up request), and lift the tool-call id it
    nests under `function.id` up to the standard `tool_calls[].id`
    (an empty id 500s the follow-up). The skip guard passes through
    frames carrying only `custom` grammar-tool deltas so
    constrained-sampling input is never dropped.
  - Send `model.maxTokens` as the `max_tokens` fallback when the
    caller omits one. The gateway applies stingy per-model defaults
    when `max_tokens` is absent (Claude 1000, Kimi 2048), silently
    truncating long responses and cutting tool-call arguments off
    mid-stream.
  - New `compat.cacheControlFormat` value `"anthropic-message"`. The
    gateway only honors Anthropic `cache_control` placed on the message
    object itself and silently ignores Anthropic's native content-block
    placement, which pi's existing `"anthropic"` format emits. The new
    format marks the system message and the last conversation message;
    tool definitions need no marker because the system breakpoint
    already covers them in Anthropic's prompt ordering.
  - `detectCompat()` auto-detects the gateway by provider name
    (`assemblyai`) or baseUrl and defaults: non-standard OpenAI compat
    (no `store`, no developer role), `max_tokens` field, and
    `cacheControlFormat: "anthropic-message"` for `claude-*` model ids
    only (GPT/Gemini/Kimi cache implicitly on the gateway; they must
    not receive `cache_control`). Strict mode needs no fork change:
    since v0.87.0 upstream defaults `supportsStrictMode` to false for
    every auto-detected endpoint.
  - Schema: `packages/ai/src/providers/compat-schema.ts` accepts the
    new value so user config can still opt in explicitly for other
    message-level gateways. The `OpenAICompletionsCompat` type and
    `models.json` validation both derive from it (since v1.2.0), and
    `models.schema.json` is regenerated from it.
  - Tests: `packages/ai/test/openai-completions-cache-control-format.test.ts`.
- `ci: add fork binary release workflow`:
  `.github/workflows/release-binary.yml` and a fork guard in
  `build-binaries.yml`. Fork-only release plumbing; see
  "Making a release" below.
- `ci: add weekly upstream sync pipeline`:
  `.github/workflows/weekly-upstream-sync.yml` plus these fork notes;
  see "Weekly sync pipeline" below.

## Known gateway quirks (theirs, not ours)

- Streaming usage omits cached/written token counts, so pi's cache
  stats display reads zero even when caching works. The tell that
  caching works is a tiny per-turn input token count. Non-streaming
  responses do include `prompt_tokens_details.cached_tokens`.
- Block-level `cache_control` is silently ignored (no error), unlike
  OpenRouter and LiteLLM which accept it.
- No image/vision input: the chat-completions schema's `ContentPart`
  is text-only ("Currently only supports text content parts"), so
  gateway models must stay `input: ["text"]` in `models.json`.
- OpenAI reasoning models (e.g. `gpt-6-luna`, `gpt-6-sol`) reject
  function tools with any `reasoning_effort` other than `"none"`
  (400: "use /v1/responses or set reasoning_effort to 'none'").
  Chat completions does not support tools plus reasoning for these
  models, and the gateway offers no `/v1/responses` (404). pi always
  sends tools, so map every thinking level to `"none"` via
  `thinkingLevelMap` in `models.json`.
- Claude models reason by default, but the gateway strips
  `reasoning_effort` and `thinking` (not in `/v1/models`
  `supported_parameters`), so the depth cannot be controlled. Only
  `reasoning_tokens` is reported; no thinking text is returned. Leave
  `reasoning` unset for gateway Claude models in `models.json`.

## Rebase checklist

- Rebase onto the upstream release tag (`v<version>`), never onto
  `upstream/main`. Main accumulates unreleased commits at the same
  version number between releases; a `-medi` release must be exactly
  upstream's release plus the fork delta. Self-check after rebasing:
  `git describe --tags --match v<version> HEAD` must read
  `v<version>-3-g<sha>` (tag plus the three fork commits).
- Reapply the commits above; watch `detectCompat()` and the compat
  resolution in `packages/ai/src/api/openai-completions.ts`, which
  upstream touches often.
- After touching `packages/ai/src/providers/compat-schema.ts`, run
  `npm run generate:schemas -w packages/coding-agent` and keep the
  `models.schema.json` diff in the `fix(ai)` commit. Seen on the
  v1.2.0 rebase: upstream moved the compat interfaces out of
  `types.ts` and the TypeBox schemas out of `model-config.ts` into
  `compat-schema.ts`; both old fork edits collapsed into one literal
  union there.
- Run `npm run hydrate-model-data` in `packages/ai` before its tests.
  The model catalog under `src/providers/data/` is gitignored,
  generated output; a stale copy fails upstream's newer tests with
  `model` undefined, which is not a fork regression.
- Run `packages/ai` tests, then verify caching end to end: a repeat
  request through the gateway must bill only a few input tokens.
- Known local-only failure: `packages/coding-agent`
  `test/package-manager.test.ts` asserts an empty global skills list
  and fails on any machine with real skills in `~/.agents`. Upstream
  test-isolation gap, not a fork regression; it passes on the CI
  runner (no `~/.agents` there), so it never blocks the pipeline.
- Known local-only failure (macOS, seen on the v0.87.0 rebase):
  `packages/coding-agent` `test/interactive-tui.test.ts`, "replaces
  the renderer and restores the previous screen for resume-hint
  exits", sees the renderer invalidated twice. It fails identically
  on a pristine upstream checkout of the tag and passes in upstream's
  CI, so it is not a fork regression. Confirm on pristine upstream
  before treating any such failure as ours: `git worktree add
  ../pi-check v<version>`, symlink `node_modules` and the hydrated
  `packages/ai/src/providers/data` in, run the one test file.
- `release-binary.yml` is fork-only, so a rebase never conflicts on
  it, but it calls upstream's `scripts/build-binaries.sh`, whose
  options drift. After every rebase, diff that script between the old
  and new base (`git diff <old-tag> <new-tag> -- scripts/build-binaries.sh`)
  and re-check every flag the workflow passes. v0.87.0 removed
  `--skip-deps` (the clipboard bindings install it controlled was
  deleted) and the darwin step died with "Unknown option" only after
  the tag was pushed, forcing a re-tag.

## Making a release

Releases are GitHub Releases on this fork carrying self-contained pi
binaries, published by `.github/workflows/release-binary.yml`.
Upstream's `build-binaries.yml` pipeline (npm publish, draft staging)
is guarded to `earendil-works/pi` and skips entirely here.

1. Find the latest published upstream release and fetch its tag
   (`origin` is the fork, `upstream` is `earendil-works/pi`):
   <!-- markdownlint-disable-next-line MD013 -->
   `gh release list --repo earendil-works/pi --exclude-drafts --exclude-pre-releases --limit 1`
   then `git fetch upstream refs/tags/v0.81.1:refs/tags/v0.81.1`.
   Rebase this branch onto that tag (`git rebase v0.81.1`) and verify
   (tests green, caching verified per the checklist above).
2. Tag the branch tip. Convention: `v<version>-medi`, where
   `<version>` is the upstream release tag you rebased onto, which
   equals the `version` field in `packages/coding-agent/package.json`
   at that tag (upstream bumps it; we never do). No numeric suffix
   after `-medi`; one fork release per upstream version. Lightweight
   tag is fine: `git tag v0.81.1-medi`
3. Push the tag: `git push origin v0.81.1-medi`. The tag push
   triggers the workflow, which builds `pi-linux-x64` and
   `pi-darwin-arm64` via `scripts/build-binaries.sh` and publishes
   the release with auto-generated notes.

### Weekly sync pipeline

`.github/workflows/weekly-upstream-sync.yml` automates the full cycle
weekly (Monday 06:15 AEST) and via manual dispatch: mirror main from
upstream, look up the latest published upstream release via
`gh release list`, version gate (abort early if `<tag>-medi` already
exists), rebase onto that release tag, verify (`npm ci` / `build` /
`check` / `build:daemon` / `test`), then force-push the branch and push
the tag, which triggers `release-binary.yml`. Notes:

- `packages/env` tests need the Rust `pi-env` daemon, which
  `npm run build` does not build; the pipeline mirrors upstream
  `ci.yml` and runs `npm run build:daemon --workspace packages/env`
  before `npm test`. `ubuntu-latest` ships rustup and cargo, and
  rustup installs the toolchain pinned in
  `packages/env/daemon/rust-toolchain.toml`.

- The rebase target is the upstream release tag, not `upstream/main`.
  A plain `git rebase <tag>` is safe there because the gate only lets
  a run through when a release newer than the branch's current base
  exists, so only the fork commits are replayed. The main mirror push
  is independent of the release and takes no part in the gate.

- Git pushes authenticate with a short-lived installation token from
  the org-owned `medi-pi-sync` GitHub App (repo secrets
  `SYNC_APP_ID` and `SYNC_APP_PRIVATE_KEY`; the app has Contents and
  Workflows write on this repo only). The built-in `GITHUB_TOKEN`
  cannot be used: it can never push workflow-file changes (which
  upstream syncs routinely carry), and its tag pushes would not
  trigger the release workflow. To rotate: generate a new private
  key on the app, update the secret, delete the old key.

- Any failure pushes nothing; the run goes red and (best effort)
  opens an issue. Rebase conflicts always escalate to a manual
  session following the steps above; the pipeline never resolves
  conflicts.
- A `dry_run` dispatch input exercises every step without pushing
  or releasing.
- The repo's default branch must stay `feat/assembly-gateway`:
  scheduled workflows only run from the default branch, and main
  must remain a fast-forwardable upstream mirror.
- Weeks without a new upstream release are skipped entirely (the gate
  aborts). Several upstream releases in one week produce one `-medi`
  release, for the latest; the intermediate versions are never
  released here.

Consumer contract: downstream CI pins a tag and runs
`gh release download <tag> -p 'pi-linux-x64'`. The asset names
`pi-linux-x64` and `pi-darwin-arm64` are the API; renaming them
breaks consumers. A bare downloaded binary prints `0.0.0` for
`--version` (the real version needs the sidecar `package.json`
layout); this is expected, not a broken build.

## Fork history

The fork delta was squashed to the three commits above on
2026-07-28. Rationale: upstream churns
`packages/ai/src/api/openai-completions.ts` constantly, and with
three separate fork commits touching it a single upstream change
could stop a rebase up to three times (a 2026-07-28 rebase stopped
twice on one upstream commit). One fork commit per concern means at
most one conflict stop each, and the array-collapse fix plus its
later revert cancel out of the delta entirely.

The original commits, oldest first, with the why preserved:

1. `fix(ai): support AssemblyAI LLM gateway in openai-completions`:
   three wire-format fixes. Collapse single-text array user content
   to a plain string (the gateway then rejected array form with
   "Invalid type for 'messages[0]'"); skip the empty `tool_calls`
   skeleton frame its Anthropic adapter emits before plain-text
   answers; lift the tool-call id nested under `function.id`.
2. `fix(ai): send model maxTokens fallback in openai-completions`:
   the agent loop never passes maxTokens, and the gateway's stingy
   per-model defaults silently truncated long responses and cut
   tool-call arguments off mid-stream.
3. `ci: add fork binary release workflow`: `release-binary.yml`
   publishing `pi-linux-x64` / `pi-darwin-arm64` on `v*` tags;
   guarded upstream's `build-binaries.yml` to `earendil-works/pi`
   because its npm trusted-publishing chain cannot succeed in a fork
   and its cleanup deletes staged release assets on failure.
4. `fix(ai): auto-detect AssemblyAI gateway prompt caching compat`:
   added `"anthropic-message"` cacheControlFormat and
   `detectCompat()` gateway detection. Every turn had been
   re-billing the full prompt because the gateway silently ignores
   block-level `cache_control`.
5. `docs: document fork release flow in FORK.md`: tag convention
   `v<version>-medi` and the consumer asset contract.
6. `ci: add weekly upstream sync and release pipeline`: first
   version of `weekly-upstream-sync.yml` (cron + dispatch, version
   gate, rebase, verify, push + tag), authenticating with a deploy
   key.
7. `revert(ai): stop collapsing array text content for the gateway`:
   the gateway accepts array-form text parts since at least
   2026-07-23 (verified by curl against Claude and GPT models), so
   the collapse from commit 1 was dropped; messages go out in
   upstream's array form again.
8. `ci: push weekly sync via medi-pi-sync app token`: `GITHUB_TOKEN`
   can never push workflow-file changes (upstream syncs routinely
   carry them) and its tag pushes trigger no workflows; deploy keys
   were disabled by org policy. Switched pushes to an installation
   token from the org-owned `medi-pi-sync` GitHub App and moved the
   cron to Monday 06:15 AEST.
