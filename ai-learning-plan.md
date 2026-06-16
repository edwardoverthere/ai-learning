# AI Engineering Learning Plan

A 15-week plan for an experienced fullstack engineer to go from "calling an AI API" to "picking, fine-tuning, and serving my own models for specific tasks."

**Two principles for the whole plan:**

1. **Build evals early (Week 4).** Every later phase reuses them to compare models, prompts, and costs objectively.
2. **Anchor on real tasks from your job.** Automate your own pain points instead of doing tutorials.

---

## Week 1 — Mental Model

No code this week. Build an accurate intuition of what an LLM actually does.

**Learn:**

- Tokens, context windows, and why they drive cost and latency
- Temperature/sampling and why outputs are non-deterministic
- Embeddings: text → vectors, cosine similarity
- Base model vs. instruct-tuned model vs. reasoning model

**Resources:**

- [Intro to Large Language Models — Andrej Karpathy](https://www.youtube.com/watch?v=zjkBMFhNj_g) (1 hr) (https://youtu.be/zjkBMFhNj_g?si=zuDNiJvWIftKbb3U&t=1441)
- [Deep Dive into LLMs like ChatGPT — Andrej Karpathy](https://www.youtube.com/watch?v=7xTGNNLPyMI) (3.5 hrs)
- [OpenAI Tokenizer playground](https://platform.openai.com/tokenizer) — paste text, see tokens
- [What Is ChatGPT Doing… and Why Does It Work? — Stephen Wolfram](https://writings.stephenwolfram.com/2023/02/what-is-chatgpt-doing-and-why-does-it-work/) (optional, deeper)

**Deliverable:** None. Watch, read, take notes.

---

## Weeks 2–3 — Calling Models via APIs

The [Vercel AI SDK](https://sdk.vercel.ai/) is the right entry point: it abstracts over providers (OpenAI, Anthropic, Google) so you learn concepts, not vendor quirks.

**Week 2 — Learn:**

- `generateText` / `streamText` ([AI SDK Core docs](https://sdk.vercel.ai/docs/ai-sdk-core))
- Streaming to the browser with `useChat` ([AI SDK UI docs](https://sdk.vercel.ai/docs/ai-sdk-ui))
- System prompts vs. user messages, multi-turn conversation state
- Provider setup: [OpenAI](https://platform.openai.com/docs), [Anthropic](https://docs.anthropic.com/), [Google AI](https://ai.google.dev/docs)

**Week 3 — Learn:**

- **Structured outputs** with `generateObject` + [Zod](https://zod.dev/) schemas ([docs](https://sdk.vercel.ai/docs/ai-sdk-core/generating-structured-data)) — the single most useful primitive for a fullstack engineer: fuzzy input in, typed JSON out
- **Tool calling** ([docs](https://sdk.vercel.ai/docs/ai-sdk-core/tools-and-tool-calling)) — let the model invoke your functions

### Sample Project 1: "DB Chat"

A Next.js chat app where the model can answer questions about your data.

- Scaffold with `npx create-next-app` + [`ai`](https://www.npmjs.com/package/ai) + [`@ai-sdk/anthropic`](https://www.npmjs.com/package/@ai-sdk/anthropic) (or any provider)
- Chat UI with `useChat`, streaming responses
- One tool: `queryDatabase(sql)` against a local Postgres/SQLite with seed data (guard it: read-only user, statement allowlist)
- Second endpoint: `POST /api/extract` — takes free text (e.g. a pasted job posting or invoice) and returns a validated, typed object via `generateObject`

**Stretch:** add a second tool (e.g. `fetchWeather`) and watch the model chain tool calls.

---

## Week 4 — Prompting + Evals

The skill gap between people who "use AI" and people who ship AI features is almost entirely here.

**Learn:**

- Prompt structure: role, instructions, few-shot examples, output format, edge-case handling
- Prompts are code: version them, test them, review them
- **Evals** — a test suite for non-deterministic outputs: golden datasets, assertion-based checks, LLM-as-judge

**Resources:**

- [Anthropic Prompt Engineering guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [OpenAI Prompting guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [promptfoo](https://www.promptfoo.dev/) — open-source eval runner, start here
- [Braintrust](https://www.braintrust.dev/) — hosted alternative with tracing
- [Your AI Product Needs Evals — Hamel Husain](https://hamel.dev/blog/posts/evals/) (essential read)

### Sample Project 2: "Eval Harness"

Take the `/api/extract` endpoint from Project 1 and make it trustworthy.

- Write 20–30 test cases in a promptfoo config, including adversarial ones (missing fields, contradictory info, prompt injection attempts in the input text)
- Mix assertion types: exact-match on fields, JSON schema validation, LLM-as-judge for fuzzy criteria
- Run in CI (GitHub Action)
- Change the prompt, watch scores move; try two different models, compare

---

## Weeks 5–6 — RAG and Embeddings

Retrieval-Augmented Generation: giving the model your data at query time instead of hoping it knows it.

**Week 5 — Learn:**

- Embedding APIs ([AI SDK `embed`](https://sdk.vercel.ai/docs/ai-sdk-core/embeddings), [OpenAI embeddings](https://platform.openai.com/docs/guides/embeddings))
- Chunking strategies and why naive fixed-size chunking fails
- [pgvector](https://github.com/pgvector/pgvector) — start here since you know Postgres; skip dedicated vector DBs until you need them

**Week 6 — Learn:**

- Hybrid search (vector + keyword/BM25) and reranking ([Cohere Rerank](https://docs.cohere.com/docs/rerank-overview))
- When RAG is the wrong answer: small corpus → just stuff it in the context window

**Resources:**

- [pgvector tutorial (Supabase)](https://supabase.com/docs/guides/ai/vector-columns)
- [Chunking strategies (Pinecone guide)](https://www.pinecone.io/learn/chunking-strategies/) — concepts apply regardless of vector store

### Sample Project 3: "Chat With My Docs"

Doc Q&A over a real corpus you care about (company wiki export, a codebase's docs, your own notes).

- Ingest pipeline: load → chunk → embed → store in Postgres + pgvector
- Query: embed question → vector search top-k → (optional) rerank → stuff into prompt with citations
- Answers must cite source chunks; render citations in the UI
- **Reuse Week 4 skills:** build a 15-question eval set, measure whether the right chunk was retrieved (retrieval eval) separately from answer quality (generation eval)

**Stretch:** add hybrid search with Postgres full-text search and compare retrieval scores.

---

## Weeks 7–9 — Agents and MCP

An agent = an LLM in a loop with tools and a goal.

**Week 7 — Learn:**

- The agent loop: model → tool call → result → model → … until done
- Multi-step tool use with the AI SDK ([stopWhen / agent loop docs](https://sdk.vercel.ai/docs/ai-sdk-core/agents))
- When _not_ to use an agent: if the steps are known, a deterministic pipeline is better

**Week 8 — Learn:**

- **[Model Context Protocol (MCP)](https://modelcontextprotocol.io/)** — the standard for exposing tools/data to models
- Build a server with the [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) or [Python SDK](https://github.com/modelcontextprotocol/python-sdk)

**Week 9 — Learn:**

- Human-in-the-loop approval, error recovery, agent observability
- [Building Effective Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents) (essential read; workflows vs. agents)

### Sample Project 4: "My MCP Server"

Expose something you own to any MCP client.

- MCP server with 2–3 tools over your own data (e.g. `search_notes`, `query_db`, an internal API wrapper)
- Connect it to Cursor or Claude Desktop and use it in real work
- Add one resource and one prompt template to learn the full protocol surface

### Sample Project 5: "Issue Triage Agent"

A small agent that does a real multi-step task.

- Input: a GitHub issue URL
- Loop: fetch issue → search the codebase (tool: `grep`/file read) → identify likely files → draft a fix plan as a comment
- Use the AI SDK agent loop with a max-steps cap and a human-approval gate before posting anything
- Log every step; eval on 10 historical issues where you know the right answer

---

## Weeks 10–11 — Open Models, Run Locally

Now go below the API line.

**Week 10 — Learn:**

- [Ollama](https://ollama.com/) — one-command local inference (`ollama run qwen3`); also see [LM Studio](https://lmstudio.ai/) for a GUI
- Model landscape: Llama, Qwen, Mistral, Gemma, DeepSeek families; instruct vs. base; parameter counts
- **Quantization** (Q4/Q8, [GGUF](https://huggingface.co/docs/hub/gguf)): the memory/speed/quality trade — this is what determines whether a model fits on your laptop

**Week 11 — Learn:**

- Reading model cards on [Hugging Face](https://huggingface.co/models)
- Benchmarks and why to distrust them — use your own evals instead
- [Ollama's OpenAI-compatible API](https://github.com/ollama/ollama/blob/main/docs/openai.md)

### Sample Project 6: "Local vs. Frontier Showdown"

- Run a 7–14B instruct model locally with Ollama
- Point Project 1's app at it — with the AI SDK + an OpenAI-compatible provider it's a config change, not a rewrite
- Run your full Week 4 eval suite against: (a) the local model, (b) a frontier API model, (c) a cheap API model
- Produce a comparison table: quality score, p50 latency, cost per 1k requests
- This one comparison teaches you more than any blog post

---

## Weeks 12–15 — Picking, Fine-tuning, and Serving Models for Specific Tasks

The end goal: matching the right model to the right job, and running it yourself when it makes sense.

**Week 12 — Learn the decision framework:**

- **Prompting → RAG → fine-tuning, in that order.** Fine-tune only when prompting a bigger model fails or unit economics demand a small model
- The honest math: API vs. self-hosted cost crossover; when self-hosting is about privacy/latency rather than cost
- The **distillation pattern**: use a frontier model to generate labeled training data, fine-tune a small model on it

**Week 13 — Learn fine-tuning:**

- LoRA/QLoRA concepts ([LoRA explained — Sebastian Raschka](https://magazine.sebastianraschka.com/p/practical-tips-for-finetuning-llms))
- Tools: [Unsloth](https://unsloth.ai/) (easiest, free Colab notebooks) or [Axolotl](https://github.com/axolotl-ai-cloud/axolotl)
- GPU rental: [RunPod](https://www.runpod.io/), [Modal](https://modal.com/), or Google Colab — don't buy hardware

**Week 14 — Learn serving:**

- [vLLM](https://github.com/vllm-project/vllm) — the standard for self-hosted throughput (batching, paged KV-cache, OpenAI-compatible server)
- Quantized serving options and GPU memory math

### Sample Project 7 (Weeks 13–15): "Distill a Task Model"

Pick one narrow, high-volume task from your actual work — ticket classification, log summarization, PII extraction.

1. **Generate data:** use a frontier model to label ~1,000 examples (review a sample by hand)
2. **Fine-tune:** LoRA on a 3–8B model with Unsloth on a rented GPU
3. **Serve:** vLLM with the OpenAI-compatible endpoint
4. **Evaluate:** run your eval suite against the fine-tuned model vs. the frontier model
5. **Do the math:** cost per 1M requests, self-hosted vs. API

If the small model matches frontier quality on that one task at a fraction of the cost, you've completed the whole arc.

---

## Ongoing — Production Hardening

Weave these into every project above rather than treating them as a separate phase:

- **Security:** prompt injection — treat all model output and retrieved content as untrusted input; scope tool permissions. Read the [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) and [Simon Willison on prompt injection](https://simonwillison.net/series/prompt-injection/)
- **Observability:** trace every LLM call — [Langfuse](https://langfuse.com/) (open source), [Braintrust](https://www.braintrust.dev/), or [OpenTelemetry GenAI conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- **Cost/latency:** [prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching), model routing (cheap model first, escalate on failure), semantic caching
- **Reliability:** retries, fallback providers, timeouts — the model is just a flaky upstream dependency

---

## Schedule at a Glance

| Week    | Focus                                  | Project                        |
| ------- | -------------------------------------- | ------------------------------ |
| 1       | Mental model                           | —                              |
| 2–3     | APIs, structured outputs, tool calling | 1: DB Chat                     |
| 4       | Prompting + evals                      | 2: Eval Harness                |
| 5–6     | RAG + embeddings                       | 3: Chat With My Docs           |
| 7–9     | Agents + MCP                           | 4: MCP Server, 5: Triage Agent |
| 10–11   | Local/open models                      | 6: Local vs. Frontier Showdown |
| 12–15   | Fine-tuning + serving                  | 7: Distill a Task Model        |
| Ongoing | Security, observability, cost          | woven into all projects        |

## Staying Current

- [Simon Willison's blog](https://simonwillison.net/) — the best practitioner signal/noise ratio
- [Hugging Face model releases](https://huggingface.co/models?sort=trending) — what's trending in open models
- [Latent Space podcast/newsletter](https://www.latent.space/) — AI engineering focus
