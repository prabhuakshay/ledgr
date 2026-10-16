# Quick Add: model and output format on OpenRouter

Research for [#11](https://github.com/prabhuakshay/ledgr/issues/11). Checked 2026-10-16. Prices and capabilities change often, so re-check the [models API](https://openrouter.ai/api/v1/models) before relying on the numbers.

## Answer

- **Output format: JSON schema `response_format` with `strict: true`**, as Velora does now. Do not switch to tool calling.
- **Model: keep `openai/gpt-6-luna`** (Velora's default) at `reasoning.effort: "low"`. Fallback: `anthropic/claude-haiku-5.5`, which costs the same.
- **Cost:** about **$0.0004 (≈ ₹0.03) per text Quick Add** and **$0.0007–0.002 (≈ ₹0.06–0.17) per receipt photo**. At 300 Quick Adds a month that is about $0.11.
- **Receipt photos:** the same call works. Add an `image_url` content part to the user message. The prompt, schema and validation stay the same. Resize the photo before sending it.
- **Accuracy on short Indian-English money notes:** no first-party source measures this. Check it with a small eval of our own (see the end of this note).

## What Velora does today

Source: `~/development/velora/apps/quick_add/openrouter.py`, `drafting.py`, `validation.py`, `.env.example`.

- It makes one `POST /api/v1/chat/completions` request with `response_format: {type: "json_schema", json_schema: {name: "draft", strict: true, schema}}`, `temperature: 0`, `max_tokens: 2000`, `reasoning: {effort: OPENROUTER_REASONING_EFFORT}` and `usage: {include: true}`.
- The defaults are `OPENROUTER_MODEL=openai/gpt-6-luna` and `OPENROUTER_REASONING_EFFORT=low`.
- The schema is strict, so every field is required and optional fields are nullable. The fields are `date`, `party_id`, `new_party_name`, `description` and `splits[]` (`from_account_id`, `to_account_id`, `amount` as a string).
- The prompt is the instructions (~1,400 chars) plus a JSON list of every visible Account (`id`, `name`, `kind`) and Party (`id`, `name`), then the user's text.
- `validation.reply_errors` re-checks every rule on the server. An invalid reply gets one corrective retry, with the errors fed back to the model. Each request is logged as an `AICall` with tokens and USD cost taken from the reply's `usage`.

## Structured output vs tool calling

| | `response_format: json_schema` | Tool calling |
|---|---|---|
| Guarantees an answer in the shape | Yes, where the endpoint supports strict mode | Only with forced `tool_choice` |
| Works with reasoning | Yes | Forcing a tool skips thinking (Haiku 5.5). Opus/Sonnet 5.5 reject a forced tool with a 400 |
| Routing on OpenRouter | `response_format` is a soft routing preference. Requests go only to the model's endpoints that support it | Same (`tools`) |

The Claude 5.5 migration guide says outright: "if you were forcing a tool call to get JSON back, use structured outputs instead". Quick Add calls no real tools; it only needs one JSON object. Tool calling adds nothing here.

Caveats from the structured outputs doc:
- Support is per endpoint, not per model.
- With `strict: true`, "some [providers] guarantee schema-conforming output, while others … treat it as a strong hint". **Keep the server-side validation.**
- The optional `response-healing` plugin repairs malformed JSON on non-streaming `json_schema` requests. It may be worth enabling, but it does not replace validation.

Small fixes for ledgr's client:
- `openai/gpt-6-luna` does not list `temperature` in `supported_parameters`, so Velora's `temperature: 0` is silently ignored. If ledgr ever sets `provider.require_parameters: true`, that unsupported parameter would block routing. Drop `temperature`.
- `require_parameters` is not needed for `response_format`, since it is already a soft preference.
- Reasoning tokens count toward `max_tokens`. Velora's 2000 is enough at `low`.

## Candidates

Data from `GET https://openrouter.ai/api/v1/models` and `/models/{slug}/endpoints`. Prices are in USD per 1M tokens, at the standard tier and list price. "SO" means `structured_outputs` is in `supported_parameters`.

| Model | Input | Output | SO | Image input | Reasoning | Notes |
|---|---|---|---|---|---|---|
| `openai/gpt-6-luna` (Velora default) | 0.10 | 0.50 | Yes, on every endpoint (OpenAI, Azure, Bedrock) | Yes | optional, efforts none to max, default medium | `openai/flex` tier is half price (0.05/0.25) but lower priority |
| `anthropic/claude-haiku-5.5` | 0.10 | 0.50 | Yes on Anthropic, Vertex and Azure. **No** on Bedrock and Claude-on-AWS | Yes | optional, can be turned off | Released 2026-10-07. The price goes up above 100K prompt tokens (not relevant here) |
| `qwen/qwen3.8-flash` | 0.15 | 0.47 | Yes (Alibaba only) | Yes | optional | One provider, so no fallback |
| `deepseek/deepseek-v4.1-flash` | 0.036 | 0.684 | Mixed. The first-party DeepSeek endpoint has **no** SO | Yes (model-level) | optional | The cheapest endpoints are fp4-quantized; prices range from 0.036 to 0.6 input across providers |
| `google/gemini-3.1-flash-lite` | 0.25 | 1.50 | Yes | Yes (image priced as input) | optional | Flex tier 0.125/0.75 |
| `google/gemini-3.5-flash-lite` | 0.30 | 2.50 | Yes | Yes | **mandatory** (minimum `minimal`) | |

Larger models (`openai/gpt-6-sol`, `anthropic/claude-sonnet-5.5` at 2/10) cost about 20x more and gain nothing on a single-line extraction task.

OpenRouter's public endpoints API returned `latency_last_30m: null` for all of these models, so there is no first-party latency figure. The main latency lever under our control is reasoning effort: `low`, or `none` if it holds up in the eval.

## Estimated cost per call

Assumptions, based on Velora's prompt with roughly 40 Accounts and 60 Parties:
- **Text call:** ~2,200 prompt tokens (instructions ~350, Account/Party JSON ~1,500, schema ~300, the note ~30) and ~300 completion tokens (JSON ~120 plus low-effort reasoning).
- **Receipt call:** the same, plus the image tokens below, with ~400 completion tokens.
- **Rupee figures:** assume ₹88/USD.
- **Corrective retries:** a retry roughly doubles the cost of that Quick Add.

Image token rules, from first-party provider docs:
- **OpenAI** (Images and vision guide): images are tokenized as 32×32 patches × 1.2 for the `gpt-5.6-luna` family. `detail: high` caps an image at 2048 px and 2,500 patches, which is ≤3,000 tokens. **`auto` behaves like `original`**, so an unresized 12 MP phone photo (3024×4032) costs ~14,400 tokens. `gpt-6-luna` is not listed in that guide yet; these numbers assume it follows the `gpt-5.6-luna` rules.
- **Anthropic** (Vision docs): Claude 4.7 and later models (which includes Haiku 5.5) use the high-resolution tier, capped at 2576 px and **4,784 tokens** per image.
- **Gemini 3** (media resolution docs): 1,120 tokens per image by default, 280 at `low` and 560 at `medium`.

| Model | Text Quick Add | Receipt photo |
|---|---|---|
| `openai/gpt-6-luna` | $0.00037 (₹0.03) | $0.00072 (₹0.06) resized to ≤2048 px; $0.0019 (₹0.17) unresized 12 MP |
| `anthropic/claude-haiku-5.5` | $0.00037 (₹0.03) | ≤ $0.0009 (₹0.08) |
| `qwen/qwen3.8-flash` | $0.00047 (₹0.04) | unknown (image token rule not documented in the sources checked) |
| `deepseek/deepseek-v4.1-flash` | $0.00028 (₹0.02) at the cheapest endpoint | unknown |
| `google/gemini-3.1-flash-lite` | $0.0010 (₹0.09) | $0.0014 (₹0.12) |
| `google/gemini-3.5-flash-lite` | $0.0014 (₹0.12) | $0.0020 (₹0.18) |

At these numbers, cost does not decide the model. Accuracy and reliability of structured output do.

## Receipt photos

OpenRouter takes images as `{"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}}` parts in a multi-part `content` array. It accepts PNG, JPEG, WebP and GIF, and recommends putting the text before the image. Velora's `complete(messages, schema)` already passes messages through, so a photo only changes the user message.

To make this work in ledgr:
- Change the `messages` type to allow content parts.
- Resize the photo on the server before sending it (long edge ≈1600–2000 px). This caps the cost and avoids OpenAI's `auto` = original behaviour.
- Add a line to the prompt about reading the total, the merchant (as the Party) and the date from a receipt.

The receipt-specific cost is about ₹0.06–0.17 per photo. The real cost is product scope: handling uploads, storing or discarding the image, and privacy (a photo carries more personal data than a typed line).

## Recommendation

1. Use `openai/gpt-6-luna` with `reasoning.effort: "low"` and a strict `json_schema` `response_format`. Drop `temperature`.
2. Pass `models: ["openai/gpt-6-luna", "anthropic/claude-haiku-5.5"]` (OpenRouter model fallbacks) for availability. Both cost the same, both take images, and SO routing skips Haiku's Bedrock endpoints.
3. Keep Velora's validation, the single corrective retry, and per-call `usage` logging.
4. Before locking the model in, run a ~30-note eval on real Velora Quick Adds (Indian-English, "paid 850 to Ramesh via GPay yesterday" style) against gpt-6-luna, haiku-5.5 and one of qwen3.8-flash or deepseek-v4.1-flash, at efforts `none` and `low`. Score validity on the first try and correct Accounts, amount and date. No first-party source answers the accuracy question.

## Sources

- OpenRouter models API: https://openrouter.ai/api/v1/models, and per-model `/api/v1/models/{slug}/endpoints` (fetched 2026-10-16)
- OpenRouter structured outputs: https://openrouter.ai/docs/guides/features/structured-outputs
- OpenRouter tool calling: https://openrouter.ai/docs/guides/features/tool-calling
- OpenRouter provider routing (`require_parameters`, soft preference for `response_format`): https://openrouter.ai/docs/guides/routing/provider-selection
- OpenRouter model fallbacks (`models`): https://openrouter.ai/docs/guides/routing/model-fallbacks
- OpenRouter image inputs: https://openrouter.ai/docs/guides/overview/multimodal/image-understanding
- OpenRouter service tiers (flex/priority): https://openrouter.ai/docs/guides/features/service-tiers
- OpenRouter response healing: https://openrouter.ai/docs/guides/features/plugins/response-healing
- OpenRouter Claude 5.5 migration guide: https://openrouter.ai/docs/cookbook/evaluate-and-optimize/model-migrations/claude-5-5
- OpenAI images and vision (patch tokenization, detail levels): https://platform.openai.com/docs/guides/images-vision
- Anthropic vision (resolution tiers and token cost): https://platform.claude.com/docs/en/build-with-claude/vision
- Gemini media resolution (image token counts): https://ai.google.dev/gemini-api/docs/media-resolution
- Velora: `apps/quick_add/openrouter.py`, `drafting.py`, `validation.py`, `.env.example`, `config/settings.py`
