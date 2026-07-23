# Fork notes

This fork exists to make pi work with AssemblyAI's LLM Gateway
(`https://llm-gateway.assemblyai.com/v1`). We control neither pi
upstream nor the gateway, so the glue lives here. Keep the footprint
minimal so rebasing onto upstream stays cheap: no upstream doc edits,
no refactors, only the changes listed below.

## Delta vs upstream

All changes are confined to `packages/ai` and
`packages/coding-agent/src/core/model-config.ts`, plus CI:

- `ci: add fork binary release workflow`:
  `.github/workflows/release-binary.yml` and a fork guard in
  `build-binaries.yml`. Fork-only release plumbing; see
  "Making a release" below.
- `fix(ai): support AssemblyAI LLM gateway in openai-completions`:
  request shaping the gateway requires.
- `fix(ai): send model maxTokens fallback in openai-completions`:
  gateway rejects requests without a max tokens value.
- `fix(ai): auto-detect AssemblyAI gateway prompt caching compat`:
  - New `compat.cacheControlFormat` value `"anthropic-message"`. The
    gateway only honors Anthropic `cache_control` placed on the message
    object itself and silently ignores Anthropic's native content-block
    placement, which pi's existing `"anthropic"` format emits. The new
    format marks the system message and the last conversation message;
    tool definitions need no marker because the system breakpoint
    already covers them in Anthropic's prompt ordering.
  - `detectCompat()` auto-detects the gateway by provider name
    (`assemblyai`) or baseUrl and defaults: non-standard OpenAI compat
    (no `store`, no developer role), `max_tokens` field, no strict
    mode, and `cacheControlFormat: "anthropic-message"` for `claude-*`
    model ids only (GPT/Gemini/Kimi cache implicitly on the gateway;
    they must not receive `cache_control`).
  - Schema: `model-config.ts` accepts the new value so user config can
    still opt in explicitly for other message-level gateways.
  - Tests: `packages/ai/test/openai-completions-cache-control-format.test.ts`.

## Known gateway quirks (theirs, not ours)

- Streaming usage omits cached/written token counts, so pi's cache
  stats display reads zero even when caching works. The tell that
  caching works is a tiny per-turn input token count. Non-streaming
  responses do include `prompt_tokens_details.cached_tokens`.
- Block-level `cache_control` is silently ignored (no error), unlike
  OpenRouter and LiteLLM which accept it.

## Rebase checklist

- Reapply the commits above; watch `detectCompat()` and the compat
  resolution in `packages/ai/src/api/openai-completions.ts`, which
  upstream touches often.
- Run `packages/ai` tests, then verify caching end to end: a repeat
  request through the gateway must bill only a few input tokens.

## Making a release

Releases are GitHub Releases on this fork carrying self-contained pi
binaries, published by `.github/workflows/release-binary.yml`.
Upstream's `build-binaries.yml` pipeline (npm publish, draft staging)
is guarded to `earendil-works/pi` and skips entirely here.

1. Update main first: on main, `git pull` (local main tracks
   `upstream/main`, i.e. `earendil-works/pi`; `origin` is the fork).
   Rebase this branch on top and verify (tests green, caching
   verified per the checklist above).
2. Tag the branch tip. Convention: `v<version>-medi`, where
   `<version>` is the current `version` field in
   `packages/coding-agent/package.json` (upstream bumps it; we never
   do). No numeric suffix after `-medi`; one fork release per
   upstream version. Lightweight tag is fine:
   `git tag v0.81.1-medi`
3. Push the tag: `git push origin v0.81.1-medi`. The tag push
   triggers the workflow, which builds `pi-linux-x64` and
   `pi-darwin-arm64` via `scripts/build-binaries.sh` and publishes
   the release with auto-generated notes.

Consumer contract: downstream CI pins a tag and runs
`gh release download <tag> -p 'pi-linux-x64'`. The asset names
`pi-linux-x64` and `pi-darwin-arm64` are the API; renaming them
breaks consumers. A bare downloaded binary prints `0.0.0` for
`--version` (the real version needs the sidecar `package.json`
layout); this is expected, not a broken build.
