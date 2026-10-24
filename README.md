# PolyChat

[![License: AGPL-3.0-only](https://img.shields.io/badge/License-AGPL--3.0--only-663399.svg)](LICENSE)

PolyChat is a browser-based chat client for multiple large language model
providers. A single interface covers Google Gemini, OpenAI, Anthropic Claude,
DeepSeek and local models served through LM Studio. PolyChat supports
streaming responses, multimodal input, branching conversation history and
LaTeX rendering.

The application has no backend. It ships as one self-contained HTML file: open
it in a browser and add your own API keys. Keys and conversations remain in the
browser's `localStorage`, and requests go directly from the browser to the
selected provider.

PolyChat was implemented with an AI coding agent, Claude Code, under the
author's direction. The author defined the requirements, reviewed each change
and merged it through pull requests. The Git history records that work.

## Supported providers

| Provider | Models | Connection |
|---|---|---|
| **Google Gemini** | Gemini 2.5 Flash Lite, Gemini 3 Flash, Gemini 3 Pro | Direct, API key |
| **OpenAI** | GPT-5 Nano, GPT-5 Mini, GPT-5.2 | Direct, API key |
| **Anthropic Claude** | Claude Haiku 4.5, Sonnet 4.5, Opus 4.5 | API key and a CORS proxy you control |
| **DeepSeek** | DeepSeek V3.2 Chat, DeepSeek V3.2 Reasoner | Direct, API key |
| **LM Studio** | Any model loaded on the server, discovered from `/v1/models` | One or more local or LAN servers |

The model can be changed in the middle of a conversation. Each reply records
which model produced it, and individual models can be shown or hidden.

## What works

- Token-by-token streaming over server-sent events for every provider. Rendering is throttled to one update per 100 ms.
- Stopping generation at any time, keeping the partial response.
- A collapsible panel for `<think>…</think>` reasoning output, kept separate from the final answer.
- **Regenerate** any assistant response while keeping earlier versions, and switch between them with `◀ 1/3 ▶`.
- **Edit and resend** any earlier user message. The previous branch, including every message that followed it, is preserved.
- Image input by file picker, drag and drop or clipboard paste, with previews and a zoom view. Images are sent in each provider's native multimodal format.
- Text and source files inlined into the prompt with explicit file delimiters: Python, MATLAB, R, C/C++, Java, Go, Rust, Julia, SQL, VHDL, more than 40 other formats, and TXT, Markdown, JSON and CSV.
- Markdown rendering, with fenced code blocks, language labels and a copy button.
- Mathematics via KaTeX, using `$…$`, `$$…$$`, `\(…\)` and `\[…\]` delimiters.
- Light, dark and system themes.

## Quick start

Requirements: a current version of Chrome, Edge or Firefox. To build from
source you also need Python 3.

1. Open `dist/AI_Chat_Client.html` in the browser.
2. Open **Settings** (the gear icon).
3. Enter API keys for the providers you use.
4. For LM Studio, start its local server, enter the URL (default `http://localhost:1234`) and select **Connect & Verify**. Additional servers can be added.
5. Select a model and start a conversation.

To rebuild the single-file application from source:

```bash
python build.py        # writes dist/AI_Chat_Client.html
```

**Claude and CORS.** Browsers block direct requests from a web page to the
Anthropic API. PolyChat therefore sends Claude requests to a proxy URL that
you configure; the proxy forwards them to
`https://api.anthropic.com/v1/messages`. Use a proxy you control.

## Streaming pipeline

```text
user message + attachments
  → provider-neutral message list (OpenAI-style roles, image_url parts)
  → active-branch context selection
  → provider adapter: request translation for the target API
  → fetch with a streaming response body
  → ReadableStream reader → line buffering across network chunks
  → SSE `data:` frame parsing → text deltas via onChunk
  → <think> separation → Markdown + KaTeX render (at most every 100 ms)
  → message stored with model name and timestamp
```

## Provider adapters

Each provider implements the same interface:
- `chat(messages, model, credentials, onChunk)` streams the response, calls `onChunk(text)` for each delta, and resolves with the complete text;
- `validate()` checks the credentials.

| Adapter | Request and stream format |
|---|---|
| OpenAI, DeepSeek, LM Studio | Chat Completions; SSE `choices[].delta.content` |
| Claude | System prompt passed as `system`; images as base64 `image` blocks; SSE `content_block_delta` events |
| Gemini | Roles mapped to `user` / `model`, images to `inlineData`, system prompt to `systemInstruction`; `streamGenerateContent?alt=sse` |

To add a provider, write one adapter file and add an entry to the model list.
The UI does not change.

## Branching model

A conversation is an ordered list of messages. When a response is regenerated
or a user message is edited, that message gains a `responseHistory` array.
Each entry stores one version's content, model and timestamp, together with
its **continuation**: the messages that followed it. Switching versions swaps
in the stored continuation, so every branch stays complete, and the context
sent to the model is always the active path only.

## Privacy rules

- Settings, API keys and conversation history are stored only in the browser's `localStorage`, under keys prefixed with `ai_chat_`.
- Requests go directly from the browser to the selected provider, or to your own proxy for Claude.
- There is no PolyChat server, telemetry or analytics. The only other network request loads KaTeX from a public CDN.
- Attached files are read locally and are sent only as part of the prompt to the selected provider.

## Repository layout

- `src/html/template.html`: page skeleton with CSS and JavaScript placeholders.
- `src/css/styles.css`: styles and light/dark themes.
- `src/js/utils/`: `localStorage` persistence, theme handling, and Markdown and LaTeX rendering.
- `src/js/api/`: one adapter per provider (`openai`, `claude`, `gemini`, `deepseek`, `lmstudio`).
- `src/js/ui/`:
  - `chatHistory.js`: conversation store and branching model;
  - `chat.js`: composer, attachments, streaming view, regenerate and edit;
  - `sidebar.js`: conversation list;
  - `settings.js`: API keys, model visibility and LM Studio servers;
  - `main.js`: initialization.
- `build.py`: bundles the sources into `dist/AI_Chat_Client.html` in dependency order (utilities, adapters, UI, initialization).
- `dist/`: the prebuilt single-file application.

## Limitations

- PDF attachments are referenced by file name only; their content is not yet extracted.
- Stopping a response ends rendering, but the underlying network request is not cancelled.
- API keys are stored in `localStorage`; use PolyChat only on a trusted device.
- Model lists are defined in the adapters and must be updated when providers release new models.
- The repository does not include an automated test suite.

## License, commercial use, and brand

Original PolyChat source code is Copyright (C) 2025-2026 Xiaofan Liu and is
licensed under **AGPL-3.0-only**. Anyone who operates a modified version over a
network must provide the corresponding source, as the license requires.
Organizations that need proprietary terms can request a separate commercial
license.

The PolyChat name, logos, slogans and distinctive brand identity are excluded
from the software license and remain all rights reserved. Model provider names
are trademarks of their respective owners; PolyChat is not affiliated with or
endorsed by any of them. Third-party components keep their own licenses.

See [`LICENSE`](LICENSE), [`LICENSING.md`](LICENSING.md),
[`TRADEMARKS.md`](TRADEMARKS.md) and [`NOTICE`](NOTICE) for the complete
licensing boundaries.
