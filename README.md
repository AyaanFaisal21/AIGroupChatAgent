## What Sage is

Sage is an AI agent that lives inside an iMessage group chat as a persistent participant. Unlike a typical chatbot or assistant, Sage is not invoked per message — it sits silently in the chat, building up long-term memory of what each member has said over time, and chooses to speak only when the group is stuck, drifting, or contradicting something previously agreed. The governing design principle is restraint: Sage aims to speak less than 1 in 10 messages in an active chat. Interventions are always grounded in retrievable context from the chat's real history rather than hallucinated facts.

Sage also responds directly when a user addresses it with `@sage` (explicit mention) or when a message leads with `hey sage` / `sage,` (indirect mention). A lightweight "vibe" system lets the group tell Sage what tone to take ("keep it casual," "professional only"), and Sage also auto-infers a vibe from recent message history after it has observed enough traffic.

Sage was built for the Photon "Agents in iMessage" track at HackPrinceton Spring 2026.

## Tech stack

- **Language:** TypeScript (Node.js runtime, executed via `tsx` in development)
- **Messaging integration:** `spectrum-ts` (Photon's Spectrum SDK), using the iMessage provider in cloud/remote mode — this is what gives Sage a live connection to an iMessage line and an async iterator of incoming messages per "space" (a chat, either 1:1 or group)
- **LLM:** Google Gemini (`@google/generative-ai`) — the `gemini-2.5-flash` family, with automatic fallback to `gemini-2.5-flash-lite` and `gemini-2.0-flash`
- **Embeddings:** Voyage AI (`voyage-3` model) for turning text into dense vectors
- **Vector memory:** an in-process TypeScript store with hand-rolled cosine similarity (Chroma is a dependency in the repo and is used in other branches/entry points, but the production path in `sage-bot.ts` uses the in-memory store defined in `memory.ts`)
- **Config:** `dotenv` for environment variables (`PHOTON_PROJECT_ID`, `PHOTON_PROJECT_SECRET`, `SAGE_PHONE`, `GEMINI_API_KEY`, `VOYAGE_API_KEY`)

## How the pieces fit together

The main entry point is `src/sage-bot.ts`. At startup it:

1. Reads Photon credentials from `.env` and constructs a `Spectrum` instance with the iMessage provider.
2. Awaits messages via `for await (const [space, message] of app.messages)` — the SDK yields each incoming iMessage alongside the chat (space) it came from.

For every incoming message, the loop does the following:

- **Normalize the sender.** If the sender's phone number matches `SAGE_PHONE`, the message was sent *by Sage itself* and is labeled "Sage" in memory; otherwise the sender's raw ID is used. This is how Sage distinguishes its own outgoing messages from everyone else's, and is the foundation of the infinite-loop guard.
- **Append to the rolling short-term buffer.** Each chat has a `HISTORY_LIMIT = 50`-message FIFO buffer holding the most recent messages with `{speaker, content, timestamp}`. This is lossless, in-memory, and immediately available to any prompt.
- **Ingest into long-term memory asynchronously.** `memory.ts` accumulates messages in a per-chat buffer and, once `CHUNK_SIZE = 6` messages have arrived, summarizes them with Gemini into 2–3 sentences, embeds the summary with Voyage-3, and stores `{id, embedding, document, speakers, startTime, endTime}` in that chat's chunk list. The chunk list is capped at `MAX_CHUNKS = 50` per chat; when full, the oldest chunk is evicted.
- **Short-circuit on self-messages.** If the message is from Sage, the loop continues without any reaction logic — this is the non-negotiable guard against self-echo loops.
- **Introduce Sage** the first time it sees a new group chat (a single one-time intro message per `spaceId`).
- **Reset a silence timer.** If nobody speaks for `SILENCE_DELAY_MS = 20_000` (20 seconds) after the last message, Sage runs a `checkSilence()` pass that asks Gemini to classify the silence as `CONFLICT` (tension, contradiction, unresolved decision) or `RESOLVED` (natural wind-down). If `CONFLICT`, Sage proactively sends a de-escalating message.

Then, branching on the content of the message:

- **Direct mention (`@sage ...`) or indirect mention (`hey sage ...`):** route to `handleMention()`. Sage retrieves `LONG_TERM_TOP_K = 4` relevant chunks, assembles a full prompt with NOW, VIBE, GROUP MEMBERS, RECENT MESSAGES, LONG-TERM MEMORY, and the user's ask, and calls Gemini to generate a 1–4 sentence plain-text reply.
- **Vibe command (`@sage vibe: casual and joke-heavy`):** extract the directive using a dedicated Gemini prompt and lock it in for that chat.
- **Vibe query (`@sage vibe?`):** report the current vibe.
- **Any other message (no mention):** route to `handleAutonomous()`. This is the restraint path — Sage does *not* reply by default. It calls `classify()` in `agent.ts`, which retrieves `TOP_K = 3` relevant chunks from long-term memory and asks Gemini (in JSON mode) whether the current message is contradicting a past decision, relitigating a settled topic, or surfacing unresolved confusion. Only if classify returns `intervene: true` does `respond()` run and send a message. A `COOLDOWN_MS = 3 * 60_000` (3-minute) per-chat cooldown prevents Sage from speaking too frequently even when classify says yes.

Separately, after enough ambient messages accumulate (`VIBE_INFER_MIN_MESSAGES = 8`, refreshed every `VIBE_INFER_REFRESH_EVERY = 20`), Sage samples the last ~30 non-Sage messages and asks Gemini to infer a vibe directive from how the humans actually talk — vocabulary, punctuation, formality, humor. This becomes the default tone unless an explicit vibe command overrides it.
