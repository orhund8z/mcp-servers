# MCP Server Review Checklist

A pre-flight checkup list for building (or reviewing) an MCP server with AI
assistance — combines the interview-prep basics with the full build/verify
discipline already validated in this repo (see `.claude/skills/mcp-builder/SKILL.md`).

## 1. Concepts to have cold

- **Three primitives**: **tools** (model-invoked actions), **resources**
  (app-pulled read-only context), **prompts** (user-invoked templates). Know
  when to use which.
- **Transport**: stdio (client owns the process, zero network surface —
  default for local/private) vs HTTP/SSE (remote access). Be able to justify
  the choice.
- **stdout is the JSON-RPC channel — logs must go to stderr.** The single
  most common first bug in a new server.
- **Schema validation doubles as the tool signature** the model sees (Zod /
  pydantic) — not just runtime safety.

## 2. Working with AI to build it (what's actually being evaluated)

- [ ] Prompt scoped and explicit: name, read-only vs mutating, backend,
      exact tools/inputs/outputs — not "build me an MCP server."
- [ ] **Read every line the AI generates before accepting** — don't rubber-stamp.
- [ ] Confirm scope before writing code (one focused round of clarifying
      questions if the ask is ambiguous), rather than guessing and building
      the wrong shape.
- [ ] One responsibility per tool — no opaque do-everything call.
- [ ] Trim/reshape external API responses; don't hand the model a raw
      upstream payload.

## 3. Implementation non-negotiables

- [ ] Logs → `stderr` (structured: timestamp, level, tool, correlation id);
      never bare strings; never log secrets/PII verbatim.
- [ ] Inputs validated with Zod, every field `.describe()`d; cross-field
      rules (e.g. `end` after `start`) checked explicitly in the handler.
- [ ] Structured errors (`isError: true`), not thrown exceptions.
- [ ] Every external HTTP call is timeout-bounded (`AbortController`, ~8s).
- [ ] `SIGTERM`/`SIGINT` handled for clean shutdown.
- [ ] Startup config/auth validated in `main()`, fails fast with one clear
      message — never a silent fallback.
- [ ] Mutating tools: conflict-check first (refuse by default, explicit
      `force` to override) + optional `idempotencyKey` so retries don't
      duplicate. Test the **key-omitted** path, not just the happy path —
      that's exactly where the real bug in this repo hid
      (`undefined !== undefined` silently filtering out a real conflict).
- [ ] Tool `annotations` set honestly (`readOnlyHint`, `destructiveHint`,
      `idempotentHint`, `openWorldHint`) — this is what lets the *client*
      decide whether to gate a call behind human confirmation.

## 4. Build and verify (do not skip, even under time pressure)

1. [ ] `npm run build` — clean, no `tsc` errors.
2. [ ] Automated test suite (Vitest) covering: happy path, required-field-
       omitted validation error, conflict path with the optional field
       genuinely omitted, `force`/override, idempotent retry, retry/backoff
       behavior on `429`/`5xx` (and no retry on other `4xx`).
3. [ ] One real external-API call end-to-end (raw JSON-RPC or Inspector) —
       don't just trust the mocked test suite.
4. [ ] **MCP Inspector** (`npm run inspect:<name>`) — call each tool, read
       each resource, render each prompt; check the stderr log pane.
5. [ ] Only after 1–4 pass, call it done. If something can't be exercised
       (e.g. OAuth needing your own credentials), say so explicitly.

## 5. Docs

- [ ] `src/<name>/README.md`: what it is → tool/resource/prompt table →
      how to use (build, Inspector, `claude mcp add` snippet, a raw-JSON-RPC
      sanity check with expected output) → design notes ("why") → layout.
- [ ] **Example prompts section** — plain-language phrasings a user would
      actually type, not just the raw tool call shape.
- [ ] Root `README.md` updated: servers table row + `## Layout` entry.
- [ ] Version bumped on any breaking `inputSchema` change.

## 6. Registering and demoing live

```bash
claude mcp add <name> --scope user -- node /absolute/path/to/dist/<name>/server.js
claude mcp list          # shows it + connection check
claude mcp get <name>    # resolved config
```

Then in a session: `/mcp` lists the connected server and its tools/resources/
prompts. Ask in plain language — no special syntax needed — and use
**Ctrl+O** to expand a collapsed tool call and show exactly what ran.
