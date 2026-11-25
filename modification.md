# Local modifications

This fork carries a small patch set on top of `deepseek-ai/deepseek-harness`. Everything here is
maintained as **one commit per feature on a patch branch**, rebased onto upstream tags — never as
edits to upstream tags themselves.

| Field | Value |
| --- | --- |
| Patch branch | `local/terminal-handback` |
| Base ref | `local/base` (records the tag the patch currently sits on) |
| First base | `dsh-v0.2.2-rc.5` (`9c2670d3bc`) |
| Upstream | `github-kji245:deepseek-ai/deepseek-harness.git` (remote `upstream`) |
| This fork | `github-kji245:KineticJetIce245/deepseek-harness.git` (remote `origin`) |

## 1. Return a taken-over agent terminal to its agent (`terminal/returnControl`)

**Why.** Upstream makes terminal input control one-way: the first browser write marked `typed`
flips `WebTerminalInfo.control` from `agent` to `user`, and after that the agent's writes, signals
and closes fail with `AgentTerminalError` code `user-control` (`packages/api/terminal-controller/src/terminal.ts`).
The person could take a shell away from the agent but never give it back without closing the
terminal and losing the process, its scrollback and its foreground job.

**What changed.**

| File | Change |
| --- | --- |
| `packages/api/terminal-controller/src/terminal.ts` | `BrowserTerminal.returnControl()`: allowed only for a running `origin: 'agent'` terminal; flips `control` back to `agent` and broadcasts state. Person-owned terminals refuse with `unavailable`, non-running ones with `not-running`. |
| `packages/api/terminal-controller/src/index.ts` | `@Remote returnControl(agent, id)` on `TerminalController`, mapping `AgentTerminalError` to `terminal/unavailable` / `terminal/control-unavailable`. |
| `packages/api/terminal-controller/src/client/model.ts` | `TerminalView.returnControl()` calls the endpoint and patches `info.control`. |
| `packages/client/ui-sidebar-terminal/src/client/TerminalTitle.tsx` | Returns a compact "Return / 交还" button for a taken-over running agent terminal; click stops propagation so it does not switch or drag the tab. |
| `packages/client/ui-sidebar-terminal/src/client/TerminalTitle.module.css` | Button styling on alias tokens. |
| `packages/client/ui-sidebar-terminal/src/client/locales.ts` | `handBack` + `handBackHint` in all 8 dictionaries; the agent badge hint now says control can be handed back. |
| `packages/experimental/terminal-panel/src/session.ts` | `TAKEN_OVER` refusal tells the model that the person can return control. |
| `packages/experimental/terminal-panel/src/index.ts` | `GUIDANCE` describes the Return button. |
| `packages/experimental/liveshare/src/endpoint-classification.ts` | New endpoint classified `owner-only`. |

**Semantics preserved.** The person still takes control by typing: the existing `typed` path flips
it back to `user`. The new call is the only way it ever returns to `agent`, and only for terminals
the agent itself opened. Reads never required control and still do not.

**Tests added.**

- `packages/api/terminal-controller/tests/terminal.spec.ts`: return after a take-over, next typed
  input takes it back; a person-owned terminal refuses; an exited agent terminal refuses.
- `packages/api/terminal-controller/tests/controller.spec.ts`: the Remote method refuses a
  person-created terminal.
- `packages/client/ui-sidebar-terminal/tests/terminal-body.client.spec.tsx`: the button appears
  only for a taken-over agent terminal, clicking calls `returnControl`, and a person-owned terminal
  has no button.
- Test doubles updated for the new Remote member (`model.client.spec.ts`, `recovery.client.spec.ts`).

**Verification performed.**
`pnpm exec vitest run packages/api/terminal-controller packages/client/ui-sidebar-terminal packages/experimental/terminal-panel` → 382 passed; repo
`pnpm run typecheck` → 5/5 gates; end-to-end: the person typed in an agent terminal, clicked
Return, and the agent's `terminal_send` executed in the same pty.

## Maintenance

```bash
git fetch upstream --tags
git switch local/terminal-handback
git rebase --onto <new tag> local/base
git branch -f local/base <new tag>
git push --force-with-lease origin local/terminal-handback
```

After a rebase, rebuild and restart on every machine:

```bash
pnpm install && pnpm run build:lib && pnpm run build:web
```

Windows self-update helpers live outside this repo in the local script library
(`restart-dsh.ps1`, `dsh-self-update.ps1`).

## Things deliberately not done

- No config switch, no per-session policy: the person decides, one terminal at a time.
- No agent-initiated reclaim tool. Asking the model to request control back would need an approval
  prompt, and under the `danger-full-access` preset approvals are disabled, so it could never fire.
