# Recast

A fast, minimal AI text tool for [Omarchy](https://omarchy.org) - a native shell plugin.
Inspired by [Kerlig](https://www.kerlig.com/) and Raycast AI on macOS: the simplest, most
productive way to ask an LLM something quickly, or get corrections on a piece of text - without
leaving the app you're in.

Select text anywhere and press **SUPER+I** to transform it with an LLM (via
[OpenRouter](https://openrouter.ai) or an [OpenCode Go](https://opencode.ai/docs/go/) subscription);
the answer streams into a centered panel and is copied
to your clipboard. Follow up to refine, regenerate, or insert the result back into the app you
came from. Or press **SUPER+SHIFT+I** to open it empty and just chat.

## Screenshots

**Quick chat** - press `SUPER+SHIFT+I`, ask anything; the model gets live context (the date, your location, the active app):

![Ask Recast anything](images/quick-chat.png)
![Recast's answer, with location-aware context](preview.png)

**Transform selected text** - select text in any app, press `SUPER+I`, and give an instruction:

![Selected text with an instruction](images/transform.png)
![The transformed result](images/transform-result.png)

## What it's for

Highlight text, hit `SUPER+I`, and ask - for example:

- **Deslop the writing** - strip the AI-ese and filler
- **Correct the grammar**
- **Translate**
- **Change the format** - e.g. plain text → Markdown
- **Explain** complex words or topics
- **Summarize**
- **Make it more concise**

The result is streamed in and copied to your clipboard, ready to paste - or **Insert** it straight
back into the app you came from.

## Install

```bash
omarchy plugin add https://github.com/TNEP4/recast.git --enable
```

Then add the keybindings - append [`bindings.snippet.lua`](bindings.snippet.lua) to
`~/.config/hypr/bindings.lua` and `hyprctl reload`:

```lua
local recast = os.getenv("HOME") .. "/.config/omarchy/plugins/io.github.tnep4.recast/bin/recast-launch"
o.bind("SUPER + I", "Recast selection", { launch = recast })
o.bind("SUPER + SHIFT + I", "Recast chat", { launch = recast .. " --chat" })
```

Set your API key: open Recast, press **Ctrl+,**, paste the key, Enter. Keys are stored
in the system keyring (never on disk). Two providers are supported — pick one in the top bar
(**Ctrl+P**) or use both:
- **OpenRouter** — `OPENROUTER_API_KEY` env var or the Settings key field.
- **OpenCode Go** (subscription via opencode's `/connect`) — picked up automatically from
  `~/.local/share/opencode/auth.json`, or set `OPENCODE_GO_API_KEY`, or paste it in Settings.

Dependencies (all in Omarchy's base): `curl`, `jq`, `wl-clipboard`, `libsecret` (`secret-tool`),
plus `hyprctl` and the `omarchy-shell`. Location context uses `omarchy-weather-location` if present.

## Removing it

1. Delete the Recast keybinding lines you appended to `~/.config/hypr/bindings.lua`, then `hyprctl reload`.
2. `omarchy plugin remove io.github.tnep4.recast`
3. Optional cleanup: `rm -rf ~/.config/recast` (settings) and
   `secret-tool clear service openrouter app recast` (the stored OpenRouter key) /
   `secret-tool clear service opencode-go app recast` (the stored Go key).

## Using it

- **SUPER+I** - transform the current selection. Type an instruction, `Enter` to send. A spinner
  shows until the first token, then the answer streams in and is auto-copied.
- **SUPER+SHIFT+I** - open empty for a direct chat (no selection).
- After an answer: **Copy output**, **Regenerate**, **Insert in <app>** (pastes into the window
  the selection came from). Type a follow-up to keep refining - the conversation is kept.
- The top bar has a **provider** picker (OpenRouter / OpenCode Go), a **model** picker and a
  **reasoning-effort** picker (OpenRouter only).
- **Web search** (Settings toggle): OpenRouter's `web` plugin grounds any model with fresh
  results (~$0.007/search + tokens, citations arrive as links). Also sent to OpenCode Go chat
  models on a trial basis — if the gateway rejects it, turn it off for Go.

### Dynamic context

The model is given live context so it can be more useful. Chat mode includes it automatically, and
you can drop these placeholders into your own system prompt (Settings → *System prompt*) - they're
filled in each time you send:

| Placeholder | Becomes |
|---|---|
| `{current-date-time}` | the current local date and time |
| `{location}` | your location (from Omarchy's weather setting, `omarchy-weather-location`) |
| `{currently-opened-app}` | the app you invoked Recast from |

### Keyboard

| Keys | Action |
|---|---|
| `Enter` | Send |
| `Ctrl+M` / `Ctrl+E` / `Ctrl+P` | Open the model / effort / provider picker (↑/↓ to move, `Enter` to pick) |
| `Ctrl+,` | Settings (API keys) |
| `Esc` | Close (a picker/settings first, then the panel) |

## Models & effort

Fifteen frontier models ship built in for OpenRouter (Claude, GPT, Gemini, Grok, DeepSeek, Qwen,
Kimi, Mistral, Meta). To use anything else on OpenRouter, open **Settings** (`Ctrl+,`) →
**Custom models**, paste an
[OpenRouter model path](https://openrouter.ai/models) (`org/slug`, e.g. `openai/gpt-4o`) and press
Enter. It joins the top-bar model picker immediately and is selected for you - so you can add the
newest OpenRouter models yourself without waiting for an app update. Remove one with the `✕` beside
it. Custom models are saved to `~/.config/recast/config.json`.

### OpenCode Go

Nothing is hardcoded: the model list is fetched from `https://opencode.ai/zen/go/v1/models`
(public endpoint — no key needed to browse) and cached to `config.json`; press **Refresh** in
Settings to pick up newly added models. Go routes models to three APIs automatically —
`chat/completions` (Kimi, GLM, DeepSeek, …), `messages` (MiniMax, Qwen), `responses` (Grok, Luna,
Muse Spark) — and sends the `x-opencode-session` header Go asks for. Extra ids can be added under
**Custom models** while the Go provider is active.

The effort picker maps to OpenRouter's `reasoning.effort` and is sent only when it isn't *Default*
and the model supports reasoning; it's hidden for models that don't. Model and effort persist to
`~/.config/recast/config.json`.

## How it works

Recast is a summoned Omarchy `panel` plugin (a layer-shell surface hosted by `omarchy-shell`).
The keybind runs `bin/recast-launch`, which grabs the primary selection and active window and
summons the panel over shell IPC with a JSON payload. Streaming is `curl -N` against
OpenRouter's SSE endpoint, or the OpenCode Go `chat/completions` / `messages` / `responses`
endpoints with per-model routing; keys are read via `secret-tool` (with `auth.json` fallback for
Go). See
[`docs/PLUGIN-RESEARCH.md`](docs/PLUGIN-RESEARCH.md) and [`docs/DEV.md`](docs/DEV.md).

The original standalone GTK4/Python version is preserved in [`legacy/`](legacy/).

## Coming next

- **Custom actions** - create your own (or let the AI write them for you): the prompts you reach
  for most, saved and just a few keystrokes away. Put your best prompts on a shelf.

## License

MIT - see [LICENSE](LICENSE).
