# Fork changelog — shashank-mugiwara/rpiv-todo

Fork of [`@juicesharp/rpiv-todo`](https://www.npmjs.com/package/@juicesharp/rpiv-todo).
Upstream monorepo: `juicesharp/rpiv-mono`, `packages/rpiv-todo`.

## v2.2.0-fork.1 — checkpoint

Verbatim checkpoint of published npm `@juicesharp/rpiv-todo@2.2.0` (latest at
fork time, published 2026-07-29). No behavioural change.

### Why fork

Audit over 163 local Pi sessions (34,375 assistant turns) found the tool's own
prompt copy drives turn inflation, not token volume:

| metric | value |
| --- | --- |
| assistant turns calling **only** `todo` | 1964 / 34375 (5.71%) |
| turns where `todo` was batched with real work | 129 (15:1 ratio) |
| cacheRead tokens on todo-only turns | 184.6M |
| billed spend on turns that produced no work | $164.59 (7.1%) |
| longest run of consecutive todo-only turns | 9 |
| create : update calls | 971 : 1569 (~1.6 status flips per task) |

Root cause: `todo.ts` `DEFAULT_PROMPT_GUIDELINES[1]` — *"Mark it completed
IMMEDIATELY when done — never batch completions"* — mandates a dedicated
round trip per status flip.

LLM-visible token cost is small (~790 tokens/session, worst case 4k). The
`details` payload persisted on every call (4.37MB across those sessions, 8.5x
the LLM-visible bytes) is **not** context cost — `session-manager.js:863`
documents details as "not sent to LLM" and `pi-ai` `convertToolResult()` builds
the provider `tool_result` block from `msg.content` alone. It is a session-file
size / replay-parse cost only, and it is not safely removable: `state/replay.ts`
needs `tasks`+`nextId` from the last record and `view/format.ts` needs `params`
to re-render historic transcript rows.

Baseline harness: `harness/evals/todo-turn-inflation.mjs` in the harness repo.

### Planned

- Re-author `DEFAULT_PROMPT_GUIDELINES` so status flips ride along with the
  next real tool call instead of occupying their own turn.
