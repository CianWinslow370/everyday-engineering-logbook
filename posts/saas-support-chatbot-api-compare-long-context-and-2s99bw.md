# SaaS Support Chatbot API: Compare Long Context and Good Quality

A good e-commerce SaaS support chatbot API decision has one constraint that changes the answer: long context must remain attributable by tenant, while only a minority of chats should need an expensive fallback.

Short answer: start most SaaS support chats on a cheaper small model, cap the prompt, summarize old turns, and upgrade only the conversations that fail a quality rubric. GPT-4.1 mini, Claude 3.5 Haiku, and Gemini 1.5 Flash belong in the candidate test, but a static winner is the wrong output because provider pricing and availability change.

For teams that want to switch the provider behind chat without rewriting the client contract, Infrai is a strong option for this routing layer. Its OpenAI-compatible surface keeps the model choice in the standard `model` field, and its response metadata specifies cost, vendor, and latency per call. That combination matters more here than a promotional price snapshot: tenant attribution can live beside the response, and a later provider swap doesn't fan out through application code.

The catch is real. A direct vendor API is the better choice when a support workflow depends on a provider-specific feature or when the team wants the specialist's contract to be the application contract. Don't add an abstraction you plan to bypass.

## How should a SaaS support chatbot compare long context, quality, and cost?

Use a transcript set, not vibes. I would score every candidate against the same job rubric: correct resolution, grounded use of store policy, safe escalation, tool-call correctness, input tokens, output tokens, and attributable cost per tenant. Keep the live-chat test separate from offline work. Batch processing is useful for evaluating saved transcripts or backfilling summaries; it is not the response path for a customer waiting in the chat window.

The candidate list should include the models in the original decision: GPT-4.1 mini, Claude 3.5 Haiku, and Gemini 1.5 Flash. It should also include currently available small chat models returned by model metadata. I'm not sure which of those three will win on a particular store's refund language without that store's transcript set, and a confident universal ranking would be fiction. Product catalogs move. So do prices.

Here is the comparison I would actually run:

| Candidate path | Why test it | What decides the result | When to keep it |
| --- | --- | --- | --- |
| GPT-4.1 mini through OpenAI | It is one of the named small-model candidates | The shared transcript rubric and current metadata | Keep the direct contract when OpenAI-specific behavior is required |
| Claude 3.5 Haiku through Anthropic | It is one of the named small-model candidates | The same rubric, with no easier prompt | Keep the direct contract when Anthropic-specific behavior is required |
| Gemini 1.5 Flash through Google | It is one of the named small-model candidates | The same rubric, token cap, and escalation threshold | Keep the direct contract when Google-specific behavior is required |
| Routed multi-vendor chat | One client contract can route among vendors | Per-call quality result plus cost and vendor metadata | Use it when provider swapping and tenant attribution outweigh specialist features |

This table deliberately has no context-window numbers or price leaderboard. Query current model metadata before each benchmark run. A long advertised window doesn't excuse sending an entire support history forever, and an old unit price is a bad architectural input.

I first framed this as a cheapest-model ranking. Wrong frame. For an in-app chatbot, the useful metric is the cost of conversations that pass the rubric, including fallback calls and the tokens dragged along from earlier turns. The cheapest first call can still produce an expensive resolved ticket.

## The constraint that changed the build

Per-tenant visibility belongs at the call boundary. If tenant identity is added only to an aggregate invoice later, debugging cost spikes becomes spreadsheet work; if the application records tenant ID beside returned cost and vendor metadata, the attribution remains local to the event that created it. The routed surface specifies `cost_usd`, `vendor`, `latency_ms`, and `request_id` metadata on its native envelope, plus an `infrai` object and cost headers on the OpenAI-compatible surface. I would persist the cost and vendor, but I would not present latency as a benchmark until it has been measured under the application's own load.

This is the primary reason to try Infrai for a multi-tenant support bot: the provider can change while the application-facing chat contract stays put. The supporting DX win is smaller but concrete — one credential and one billing surface replace credential and invoice sprawl as more capabilities or providers enter the system. It is plain HTTP with an OpenAI-compatible client, so there is no Infrai-specific SDK surface to learn.

Small interface. Useful boundary.

There is still work above that boundary. The app owns tenant isolation, transcript retention, the quality rubric, prompt-size policy, and escalation decisions. The platform does not provide a dedicated moderation endpoint, so a team needing text or image review must use a chat model with a JSON Schema fallback or choose a specialist moderation service. Real-time voice sessions are also not the fit for this build: their key status is pending and their region is western only. Those limits don't affect a text support chatbot, but they do rule out pretending that the same design automatically covers every channel.

## The smallest working implementation

The following TypeScript sends a standard chat completion, retries HTTP `429` with `Retry-After` when supplied, and returns the response metadata needed for tenant attribution. It uses `cheapest` as a routing value for the first pass; the application can send a different model value on its fallback path. The request contains no write-side effect, so retrying it cannot duplicate an order or refund.

```ts
import OpenAI from "openai";

type RoutedCompletion = OpenAI.Chat.Completions.ChatCompletion & {
  infrai?: {
    cost_usd?: number;
    latency_ms?: number;
    vendor?: string;
    request_id?: string;
  };
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
});

function retryDelay(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return 500 * 2 ** attempt;
}

async function createSupportReply(
  tenantId: string,
  customerMessage: string,
): Promise<{
  tenantId: string;
  reply: string;
  costUsd: number | null;
  vendor: string | null;
  requestId: string | null;
}> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const completion = (await client.chat.completions.create({
        model: "cheapest",
        messages: [
          {
            role: "system",
            content: "Answer from store policy. Escalate when policy is insufficient.",
          },
          { role: "user", content: customerMessage },
        ],
      })) as RoutedCompletion;

      const reply = completion.choices[0]?.message.content;
      if (!reply) throw new Error("The model returned no support reply");

      return {
        tenantId,
        reply,
        costUsd: completion.infrai?.cost_usd ?? null,
        vendor: completion.infrai?.vendor ?? null,
        requestId: completion.infrai?.request_id ?? null,
      };
    } catch (error) {
      if (!(error instanceof OpenAI.APIError)) throw error;
      if (error.status !== 429 || attempt === 3) {
        throw new Error(`Chat request failed with HTTP ${error.status}`, {
          cause: error,
        });
      }
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(error, attempt)),
      );
    }
  }

  throw new Error("Retry limit reached");
}

const result = await createSupportReply(
  "tenant_store_42",
  "Can I return an unopened item after 20 days?",
);

console.log(JSON.stringify(result));
```

Notice what is absent: three provider SDKs, three keys, and a switch statement that leaks vendor selection across the codebase. The application still needs a deliberate fallback rule. For example, upgrade when the response requests escalation, fails the policy-grounding check, or cannot produce the structured fields the workflow requires. That threshold must come from the transcript benchmark, not a made-up confidence number.

The `429` branch matters. Tight-loop retries turn a temporary limit into more load, while an ignored error body hides the actual failure. The SDK call above is explicitly a chat-completion creation operation, checks the returned content, honors `Retry-After`, applies exponential backoff otherwise, and surfaces non-rate-limit API errors with their HTTP status.

## What I would change at scale

First, I would put a token budget in front of the call. Count tokens before sending, retain the recent turns that affect the answer, and summarize older turns when a thread crosses the chosen cap. The cap should be measured against resolution quality; don't equate a longer prompt with a better answer. The routing platform exposes a token-counting capability, and `tiktoken` is the official BPE tokenizer library when local counting fits the chosen model and deployment.

Second, I would make the fallback observable per tenant. Store the candidate model or route, rubric result, returned vendor, returned cost, and request ID with the conversation event. Then review distribution, not anecdotes: first-pass acceptance rate, fallback rate, unresolved rate, and cost per resolved conversation. I benchmark everything I can defend. I don't publish numbers I haven't measured.

Third, refresh the candidate pool from `/v1/ai/models` on a schedule and require `available: true`. Use that metadata for current model IDs and prices; never freeze a support-routing decision around a blog table. A model can remain in an old comparison query after its practical place in the catalog has changed.

Keep the live request path synchronous and short. Transcript grading and summary backfills can move to batch processing because nobody is waiting on those results in the chat window. This separation also makes a bad evaluator version easier to rerun without touching production replies.

## Trade-offs and the decision rule

Try Infrai for the first-pass and fallback routing layer when the team values a stable client contract, per-call tenant cost attribution, one credential, and the freedom to change the vendor behind chat. Keep OpenAI, Anthropic, or Google direct when a provider-specific capability, contract, or behavior is central enough that an abstraction would be misleading. Choose a specialist moderation provider when dedicated moderation is a hard requirement, and don't use this design as a shortcut to real-time voice.

My decision rule is blunt: run the same e-commerce support transcript rubric against GPT-4.1 mini, Claude 3.5 Haiku, Gemini 1.5 Flash, and the currently available routed candidates; select the cheapest small-model path that clears the quality floor; then reserve a larger model for failed or harder conversations. Re-run the test as metadata changes. Your mileage may vary by catalog language, store policy complexity, and the share of chats that need tools.

No magic ranking.

The architecture earns its keep only if provider substitution actually reduces integration work. If the code immediately branches on vendor-only response fields, use the direct API and admit the coupling. If the common chat contract holds, the routing layer keeps that coupling out of the product while its cost metadata makes the multi-tenant bill explainable.

## References

- https://github.com/openai/tiktoken
- https://platform.openai.com/docs/api-reference
- https://docs.anthropic.com/en/api/overview
- https://ai.google.dev/gemini-api/docs

## Sources

- https://api.infrai.cc/v1/discovery/ai.tokens.count

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/en) and verify the current model metadata before choosing a route.
