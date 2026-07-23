# Fork notes

This fork exists to make pi work with AssemblyAI's LLM Gateway
(`https://llm-gateway.assemblyai.com/v1`). We control neither pi
upstream nor the gateway, so the glue lives here. Keep the footprint
minimal so rebasing onto upstream stays cheap: no upstream doc edits,
no refactors, only the changes listed below.

## Delta vs upstream

All changes are confined to `packages/ai` and
`packages/coding-agent/src/core/model-config.ts`, plus CI:

- `ci: add fork binary release workflow` (57db4278):
  `.github/workflows/release-binary.yml` and a hook in
  `build-binaries.yml`. Fork-only release plumbing.
- `fix(ai): support AssemblyAI LLM gateway in openai-completions`
  (260633b2): request shaping the gateway requires.
- `fix(ai): send model maxTokens fallback in openai-completions`
  (cd2e78e9): gateway rejects requests without a max tokens value.
- Prompt caching (this change):
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
