# PolyChat

A single-file browser client for chatting with multiple LLM providers.

PolyChat provides one interface for Google Gemini, OpenAI, Anthropic Claude, DeepSeek and local models served through LM Studio. It supports streaming responses, multimodal input, branching conversation history and LaTeX rendering. The application has no backend. It ships as one self-contained HTML file, and API keys and conversations stay in the browser's `localStorage`.

## Overview

| | |
|---|---|
| Providers | Google Gemini, OpenAI, Anthropic Claude, DeepSeek, LM Studio (OpenAI-compatible, multiple servers) |
| Frontend | Vanilla JavaScript, HTML and CSS; KaTeX for mathematics |
| Build | `build.py` bundles the sources into a single HTML file |
| Storage | Browser `localStorage` only |

## Features

### Providers and models
- Gemini 2.5 Flash Lite, Gemini 3 Flash and 3 Pro; GPT-5 Nano, GPT-5 Mini and GPT-5.2; Claude Haiku, Sonnet and Opus 4.5; DeepSeek V3.2 (Chat and Reasoner).
- Local models through one or more LM Studio servers. Available models are discovered from each server's `/v1/models` endpoint.
- The model can be changed mid-conversation. Each reply records which model produced it, and individual models can be shown or hidden in the model list.

### Streaming
- Responses stream token by token over server-sent events for every provider. Rendering is throttled to one update per 100 ms, so long responses stay smooth.
- Generation can be stopped at any time, and the partial response is kept.
- `<think>…</think>` output from reasoning models is shown in a collapsible panel, separate from the final answer.

### Branching conversation history
- **Regenerate** any assistant response while keeping the earlier versions, and switch between them with `◀ 1/3 ▶`.
- **Edit and resend** any earlier user message. The previous branch, including all messages that followed it, is preserved.
- Only the active branch is sent to the model as context.

### Attachments
- Images can be added with the file picker, drag and drop or clipboard paste. They are previewed with a zoom view and sent in each provider's native multimodal format.
- Text and source files (Python, MATLAB, R, C/C++, Java, Go, Rust, Julia, SQL, VHDL and more than 40 other formats, plus TXT, Markdown, JSON and CSV) are inlined into the prompt with explicit file delimiters.

### Rendering
- Markdown, including fenced code blocks with language labels and a copy button.
- Mathematics via KaTeX, with `$…$`, `$$…$$`, `\(…\)` and `\[…\]` delimiters.
- Light, dark and system themes.

## Architecture

```
src/
├── html/template.html     page skeleton with CSS/JS placeholders
├── css/styles.css         styles and light/dark themes
└── js/
    ├── utils/
    │   ├── storage.js     localStorage persistence (settings, chats) and theme handling
    │   └── markdown.js    Markdown and LaTeX rendering
    ├── api/               one adapter per provider, sharing a common interface
    │   ├── openai.js
    │   ├── claude.js
    │   ├── gemini.js
    │   ├── deepseek.js
    │   └── lmstudio.js
    └── ui/
        ├── chatHistory.js conversation store and branching model
        ├── sidebar.js     conversation list
        ├── chat.js        composer, attachments, streaming view, regenerate and edit
        ├── settings.js    API keys, model visibility, LM Studio servers
        └── main.js        initialization
build.py                   bundles the sources into dist/AI_Chat_Client.html
```

### Provider adapters

Each provider implements the same interface:
- `chat(messages, model, credentials, onChunk)` streams the response, calls `onChunk(text)` for each delta, and resolves with the complete text.
- `validate()` checks the credentials.

The UI builds one provider-neutral message list, using OpenAI-style roles and `image_url` parts for images. Each adapter converts that list to its provider's API:

| Adapter | Request and stream format |
|---|---|
| OpenAI, DeepSeek, LM Studio | Chat Completions; SSE `choices[].delta.content` |
| Claude | System prompt passed as `system`; images as base64 `image` blocks; SSE `content_block_delta` events |
| Gemini | Roles mapped to `user` / `model`, images to `inlineData`, system prompt to `systemInstruction`; `streamGenerateContent?alt=sse` |

Adding a provider requires one adapter file and an entry in the model list; the UI does not change.

### Streaming pipeline

Each adapter reads the response body with a `ReadableStream` reader. It buffers partial lines across network chunks, parses SSE `data:` frames and forwards the text deltas. The chat view accumulates the raw text, separates `<think>` content from the answer, and re-renders Markdown and LaTeX at most once every 100 ms.

### Branching model

A conversation is an ordered list of messages. When a response is regenerated or a user message is edited, that message gains a `responseHistory` array. Each entry stores one version's content, model and timestamp, together with its **continuation**: the messages that followed it. Switching versions swaps in the stored continuation, so every branch stays complete. The context sent to the model is always the active path.

### Build

`build.py` concatenates the CSS and the JavaScript modules into the HTML template in dependency order (utilities, provider adapters, UI, initialization). The result is a single portable file in `dist/`. KaTeX is loaded from a CDN.

## Requirements

- A current version of Chrome, Edge or Firefox.
- An API key for each hosted provider you use, or a running LM Studio server.
- Python 3, to build from source.

## Getting started

Open `dist/AI_Chat_Client.html` in the browser, open **Settings** (the gear icon), enter your API keys and select a model.

To build from source:

```bash
python build.py        # writes dist/AI_Chat_Client.html
```

### Provider setup

| Provider | Requirement |
|---|---|
| Google Gemini | API key from [Google AI Studio](https://aistudio.google.com/) |
| OpenAI | API key from the [OpenAI Platform](https://platform.openai.com/) |
| Anthropic Claude | API key and a CORS proxy URL (see below) |
| DeepSeek | API key from the [DeepSeek Platform](https://platform.deepseek.com/) |
| LM Studio | Start the LM Studio server, enter its URL (default `http://localhost:1234`) and select **Connect & Verify**. Additional servers can be added. |

**Claude and CORS.** Browsers block direct requests from a web page to the Anthropic API. PolyChat therefore sends Claude requests to a proxy URL that you configure, and the proxy forwards them to `https://api.anthropic.com/v1/messages`. Use a proxy you control.

## Privacy

Settings, API keys and conversation history are stored only in the browser's `localStorage`, under keys prefixed with `ai_chat_`. Requests go directly from the browser to the selected provider, or to your own proxy for Claude. There is no PolyChat server.

## Limitations

- PDF attachments are referenced by file name only; their content is not yet extracted.
- Stopping a response ends rendering, but the underlying network request is not cancelled.
- API keys are stored in `localStorage`; use PolyChat only on a trusted device.
- Model lists are defined in the adapters and must be updated when providers release new models.

## Development

The implementation was developed with an AI coding agent (Claude Code). The author defined the requirements, reviewed each change and merged it through pull requests.
