
# OpenClaw-managed Browser on DGX Spark (Ubuntu arm64) with vLLM-served Qwen3-80B (A3B)

This is the exact, working setup pattern for **DGX Spark (arm64)** where:
- The LLM is served locally via **vLLM** (OpenAI-compatible API)
- Web access is through the **OpenClaw-managed browser** (CDP on port 18800)
- You avoid the Chrome-extension relay “re-click ON after navigation” workflow

References:
- vLLM tool calling flags: `--enable-auto-tool-choice`, `--tool-call-parser …` :contentReference[oaicite:0]{index=0}
- OpenClaw browser profiles/ports (`openclaw` profile, CDP port range 18800–18899, `chrome` profile uses 18792): :contentReference[oaicite:1]{index=1}
- OpenClaw chrome extension relay setup (token/port): :contentReference[oaicite:2]{index=2}
- OpenClaw profile routing/default profile workaround discussions: :contentReference[oaicite:3]{index=3}

---

## 0) Assumptions
- You can run OpenClaw gateway locally on the DGX (loopback is fine).
- You have Chromium installed (on Ubuntu Noble this is typically Snap Chromium).
- Your OpenClaw gateway token auth is enabled (mode `token`).

---

## 1) Start vLLM (OpenAI-compatible) for Qwen3 tool calling

### 1.1 Launch vLLM
Use an OpenAI-compatible server and enable tool calling:

```bash
vllm serve /home/aim_lab/data/vllm-spark/qwen80b-local \
  --host 127.0.0.1 --port 8000 \
  --served-model-name "Qwen3-80B" \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --trust-remote-code \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
````

Notes:

* `--enable-auto-tool-choice` + `--tool-call-parser` are the documented vLLM switches for automatic function/tool calling. ([vLLM][1])
* Qwen’s own function-calling docs show Hermes-style tool use with vLLM for Qwen3. ([Qwen][2])

### 1.2 Verify the vLLM endpoint

```bash
curl -s http://127.0.0.1:8000/v1/models | head
```

---

## 2) Point OpenClaw at vLLM (so the agent uses your local model)

Make sure OpenClaw is configured to use the vLLM OpenAI-compatible base URL and the correct model id shown by `/v1/models`.
(Exact OpenClaw config keys can vary by version; the key requirement is: OpenClaw’s “primary model” must target your vLLM endpoint.)

After updating OpenClaw config:

```bash
openclaw gateway restart
openclaw status
```

---

## 3) Configure OpenClaw-managed browser (Snap-friendly, attach-only)

OpenClaw’s browser system uses **profiles**; the managed one is named `openclaw`. The `chrome` profile is the extension relay on port 18792 (not what we want here). ([OpenClaw][3])

### 3.1 Clean up “legacy browser override” (important)

If your `~/.openclaw/openclaw.json` previously had a top-level:

```json
"browser": { "cdpUrl": "http://127.0.0.1:18800", ... }
```

remove that override and keep only:

```json
"browser": { "enabled": true }
```

Why:

* A legacy/global `browser.cdpUrl` can hijack routing so the agent/CLI doesn’t behave consistently across profiles. This is commonly implicated when the agent ignores `profile="openclaw"` and routes to `chrome`/extension instead. ([GitHub][4])

### 3.2 Set OpenClaw’s default browser profile to managed browser

```bash
openclaw config set browser.defaultProfile openclaw
openclaw gateway restart
```

This prevents the agent from trying the extension relay by default. ([Answer Overflow][5])

---

## 4) Start headless Chromium with CDP on port 18800 (DGX/arm64)

### 4.1 Create a Snap-writable profile directory

On Ubuntu Noble, Chromium is often Snap-confined; use a Snap-writable user data dir:

```bash
pkill -f chromium || true
mkdir -p "$HOME/snap/chromium/common/openclaw-user-data"
chmod 700 "$HOME/snap/chromium/common/openclaw-user-data"
```

### 4.2 Launch Chromium headless with CDP

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-address=127.0.0.1 \
  --remote-debugging-port=18800 \
  --user-data-dir="$HOME/snap/chromium/common/openclaw-user-data" \
  about:blank &
```

Expected:

* You will see `DevTools listening on ws://127.0.0.1:18800/...`
* You may also see noisy logs (MoTTY X11 warnings, `DEPRECATED_ENDPOINT`); those are usually harmless for CDP automation.

### 4.3 Verify CDP endpoint is live

```bash
curl -s http://127.0.0.1:18800/json/version | head
```

OpenClaw’s managed browser profile typically uses CDP ports in the 18800–18899 range; 18800 is the common default. ([OpenClaw][3])

---

## 5) Start the OpenClaw `openclaw` browser profile

```bash
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw status
```

You want `running: true`.

---

## 6) CLI verification (no agent yet)

### 6.1 Navigate

```bash
openclaw browser --browser-profile openclaw navigate https://www.monash.edu/
```

### 6.2 Snapshot

```bash
openclaw browser --browser-profile openclaw snapshot
```

If the snapshot shows the page structure/links, your managed browser wiring is correct.

---

## 7) Agent test (DingTalk / chat) — minimal output to avoid context overflow

Because your vLLM server is configured for **32k max context**, browsing can overflow if the agent dumps large page content. Keep outputs tiny.

Send this to your agent:

> Use browser profile **openclaw**. Navigate to `https://example.com`.
> Reply with **only** the page title (max 10 words). Do not paste page text/HTML.

Expected: `Example Domain`.

If you previously saw the agent complain:

> “No tab is attached to the OpenClaw Chrome extension…”

that is exactly what setting `browser.defaultProfile=openclaw` avoids (it stops routing to the extension relay profile). ([Answer Overflow][5])

---

## 8) Common pitfalls (and the fixes that worked here)

### Pitfall A: Agent tries to use Chrome extension relay (`chrome` profile)

Symptoms:

* Agent says “no tab is attached to the OpenClaw Chrome extension…”
  Fix:
* Set the default profile to managed browser:

  ```bash
  openclaw config set browser.defaultProfile openclaw
  openclaw gateway restart
  ```

This aligns with known routing issues and their workarounds. ([GitHub][4])

### Pitfall B: Snap Chromium refuses your profile dir (SingletonLock permission errors)

Fix:

* Use Snap-writable user data directory under:
  `~/snap/chromium/common/...`

### Pitfall C: vLLM tool calls fail in streaming

If you see tool calls not parsed correctly in streaming with the Hermes parser, test non-streaming for tool-heavy turns (there are open reports of streaming parsing issues). ([GitHub][6])

---

## 9) Optional: Keep a “stable” browsing habit to prevent context overflow

When testing browsing with an agent:

* Ask for **title only** or **one short field**
* Avoid “summarize the whole page”
* Start fresh sessions if a thread grows too large

---

## Appendix: Extension relay vs Managed browser (quick reference)

* **Managed browser (`openclaw` profile):** CDP on 18800+; no extension clicking; best for automation stability. ([OpenClaw][3])
* **Extension relay (`chrome` profile):** targets `http://127.0.0.1:18792`; requires extension Options token/port and attaching tabs; can detach after navigation. ([OpenClaw][7])

```
[1]: https://docs.vllm.ai/en/latest/features/tool_calling/?utm_source=chatgpt.com "Tool Calling - vLLM"
[2]: https://qwen.readthedocs.io/en/latest/framework/function_call.html?utm_source=chatgpt.com "Function Calling - Qwen - Read the Docs"
[3]: https://docs.openclaw.ai/tools/browser?utm_source=chatgpt.com "Browser (OpenClaw-managed)"
[4]: https://github.com/openclaw/openclaw/issues/4841?utm_source=chatgpt.com "Browser tool ignores profile parameter, routes all requests ..."
[5]: https://www.answeroverflow.com/m/1471644396834390097?utm_source=chatgpt.com "how can i let openclaw automatically actually browse ..."
[6]: https://github.com/vllm-project/vllm/issues/31871?utm_source=chatgpt.com "Streaming mode with --tool-call-parser hermes returns raw ..."
[7]: https://docs.openclaw.ai/tools/chrome-extension?utm_source=chatgpt.com "Chrome Extension"
