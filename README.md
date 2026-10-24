# PolyChat

**One HTML file, every model.** PolyChat is a browser-based chat client that talks to Google Gemini, OpenAI, Anthropic Claude, DeepSeek and local models served by LM Studio from a single interface, with streaming responses, multimodal input, branching conversation history and LaTeX rendering.

There is no backend and no build step for users: the whole application ships as one self-contained HTML file. Open it in a browser, add your own API keys, and chat. Keys and conversations stay in the browser's `localStorage`.

## Features

**Multi-provider, one interface**
- Google Gemini (2.5 Flash Lite, 3 Flash, 3 Pro), OpenAI GPT-5 (Nano, Mini, 5.2), Anthropic Claude 4.5 (Haiku, Sonnet, Opus) and DeepSeek V3.2 (Chat, Reasoner).
- Local models through LM Studio's OpenAI-compatible server, including **several LM Studio servers at once** (for example a small model on a laptop and a larger one on a GPU workstation); models are discovered from each server's `/v1/models`.
- Switch models mid-conversation; each reply is tagged with the model that produced it. Model visibility is configurable per model.

**Streaming and generation control**
- Token-by-token streaming over server-sent events for every provider, rendered live (throttled to one repaint per 100 ms so long answers stay smooth).
- **Stop** at any time: rendering ends immediately and the partial answer is kept.
- Reasoning models that emit `<think>…</think>` blocks get a collapsible **Thinking** panel, separated from the final answer.

**Branching conversation history**
- **Regenerate** any assistant reply without losing the old one; navigate versions with `◀ 1/3 ▶`.
- **Edit and resend** any earlier user message; the previous branch, including everything that followed it, is preserved.
- Each branch keeps its own continuation, and only the active branch is sent to the model as context.

**Multimodal input**
- Images (PNG, JPG, GIF, …) by file picker, drag and drop or clipboard paste, with previews and a zoom view; images are sent in each provider's native multimodal format.
- Text and code files (Python, MATLAB, R, C/C++, Java, Go, Rust, Julia, SQL, VHDL and 40+ other languages, plus TXT/MD/JSON/CSV) are inlined into the prompt with clear file delimiters.

**Rendering**
- Markdown with fenced code blocks, language labels and one-click copy.
- LaTeX math via KaTeX for `$…$`, `$$…$$`, `\(…\)` and `\[…\]`.
- Light, dark and system themes; a minimal iOS-inspired design.

## Architecture

```
src/
├── html/template.html     page skeleton with CSS/JS placeholders
├── css/styles.css         design system, light/dark themes
└── js/
    ├── utils/
    │   ├── storage.js     localStorage persistence (settings, chats), theme manager
    │   └── markdown.js    Markdown + LaTeX renderer
    ├── api/               one adapter per provider, same contract
    │   ├── openai.js
    │   ├── claude.js
    │   ├── gemini.js
    │   ├── deepseek.js
    │   └── lmstudio.js
    └── ui/
        ├── chatHistory.js conversation store and branching model
        ├── sidebar.js     chat list (create, rename, delete)
        ├── chat.js        composer, attachments, streaming view, regenerate/edit
        ├── settings.js    API keys, model visibility, LM Studio servers
        └── main.js        bootstrap
build.py                   bundles everything into dist/AI_Chat_Client.html
```

**Provider adapters.** Every provider is an object with the same contract:
`chat(messages, model, credentials, onChunk)` streams the reply and calls `onChunk(text)` for each delta, then resolves with the full text; `validate()` checks the credentials. The UI builds one provider-neutral message list (OpenAI-style roles, with `image_url` parts for images), and each adapter translates it to its own API:

| Adapter | Translation |
|---|---|
| OpenAI, DeepSeek, LM Studio | Chat Completions format, SSE `choices[].delta.content` |
| Claude | system prompt lifted into `system`; images converted to base64 `image` blocks; SSE `content_block_delta` events |
| Gemini | roles mapped to `user`/`model`, images to `inlineData`, system prompt to `systemInstruction`; `streamGenerateContent?alt=sse` |

Adding a provider means writing one adapter file and registering it in the model list; the UI does not change.

**Streaming pipeline.** Each adapter reads the response body with a `ReadableStream` reader, buffers partial lines across network chunks, parses SSE `data:` frames and forwards text deltas. The chat view accumulates the raw text, separates `<think>` content from the answer, and re-renders Markdown/LaTeX on a 100 ms throttle.

**Branching model.** Conversations are stored as an ordered message list. When a reply is regenerated or a user message is edited, the message gains a `responseHistory` array; each entry stores that version's content, model, timestamp and its own **continuation** (the messages that followed it). Switching versions swaps in the stored continuation, so every branch stays complete, and the context sent to the model is always the active path only.

**Persistence and privacy.** Settings, API keys and chat history are stored in `localStorage` under the `ai_chat_*` keys. Nothing is sent anywhere except directly to the provider you select (or to your own proxy for Claude, see below).

**Single-file build.** `build.py` concatenates the CSS and the JavaScript modules in dependency order (utils → API adapters → UI → bootstrap) into the HTML template, producing one portable file in `dist/`. KaTeX is loaded from a CDN.

## Getting started

**Use it:** open `dist/AI_Chat_Client.html` in Chrome, Edge or Firefox, click the gear icon, enter the API keys you have, and pick a model.

**Build from source:**

```bash
python build.py        # writes dist/AI_Chat_Client.html
```

### Provider setup

| Provider | What you need |
|---|---|
| Google Gemini | API key from [Google AI Studio](https://aistudio.google.com/) |
| OpenAI | API key from the [OpenAI Platform](https://platform.openai.com/) |
| Anthropic Claude | API key **and** a CORS proxy URL (see below) |
| DeepSeek | API key from the [DeepSeek Platform](https://platform.deepseek.com/) |
| LM Studio | start LM Studio's local server, enter its URL (default `http://localhost:1234`) and click **Connect & Verify**; add more servers as needed |

**Claude and CORS.** Browsers block direct calls to the Anthropic API from a web page, so PolyChat sends Claude requests to a proxy URL you configure, which forwards them to `https://api.anthropic.com/v1/messages`. Use a proxy you control.

## Limitations

- PDFs can be attached and are referenced by name in the prompt, but their content is not extracted yet.
- API keys live in the browser's `localStorage`; use PolyChat on a machine you trust.
- Model lists are defined in the adapters and need updating when providers release new models.
