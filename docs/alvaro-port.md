# Alvaro port backlog

Source: `alvarohulse/proofshot` (`alvaro/main`, tip `e788c90`, 2026-08-19).
This fork tracks `safzanpirani/proofshot` (`da374cc`). Do not merge alvaro wholesale.

Alvaro is 92 commits / +16.5k lines, almost all Codex/Cursor. Keep the behaviors below. Leave the evidence ontology, replay framework, tmux capture, and Windows recording ban behind.

Compare anytime:

```bash
git fetch alvaro
git log --oneline HEAD..alvaro/main
git diff --stat HEAD...alvaro/main
```

Pinned local runtime: `agent-browser@0.27.0`. npm latest is `0.34.0`. Do not upgrade until P0-1 lands — alvaro’s quiet FFmpeg wrapper exists because 0.34.0 can deadlock while finalizing recordings.

## Do port

### P0-1 — Quiet FFmpeg wrapper / recording finalization

Why: 0-byte `session.webm`, hung `proofshot stop`, or a deadlock when agent-browser finalizes FFmpeg. This is the blocker for upgrading `agent-browser`.

Alvaro: `src/browser/ffmpeg-wrapper.ts`, `src/commands/start.ts`, `src/commands/stop.ts`
Key commits: `354b1d6`, `a14f92c`, `d658f7a`, `6ba0ead`, `e788c90`

Port the wrapper and failed-start ownership only. Do not take the Windows “recording unsupported” hard-ban unless we hit that bug here (this machine is macOS).

Acceptance: start → exec → stop on a real page produces a non-empty `session.webm` and a working `viewer.html` against `agent-browser@0.34.0`. Failed start must not close the browser before the recorder has finalized, and must not delete a usable video.

### P0-2 — Failed trim must not destroy the original video

Why: alvaro found empty/failed FFmpeg trim replacing good footage.

Alvaro: `src/commands/stop.ts`, video trim path
Key commits: `8e78372`, `ca1eae8`

Validate trimmed output (size + duration) before replacing `session.webm`. Keep the original on failure.

### P0-3 — `proofshot pr` uploads to the PR head repo

Why: current `getRepoInfo()` uses `gh repo view` (`src/utils/github.ts`). On a fork checkout that can resolve to the parent, so artifacts get pushed to someone else’s `proofshot-artifacts` branch.

Alvaro: `src/utils/github.ts`, `src/commands/pr.ts`, `src/browser/provenance.ts` (too big — take the repo-resolution piece only)
Key idea from alvaro: resolve the PR head repository, never the network parent.

Acceptance: `proofshot pr` from a fork PR writes the artifacts branch to the head repo and comments on that PR. Dry-run prints the target `owner/repo`.

### P1-1 — Viewer fit / aspect ratio

Why: this is the review artifact we actually open.

Alvaro: `src/artifacts/viewer.ts`
Key commits: `0ca9ded`, `f83b0ad`, `eb28a83`

- Use recorded viewport as `aspect-ratio` before video metadata loads
- Fit video + scrub bar in the window
- Do not stretch

Do not port the verdict/integrity chrome unless we also take P2-3.

### P1-2 — Timeline offset vs physical trim

Why: idle-frame compression / clamped trim shifts action markers off the video.

Alvaro: `src/commands/stop.ts`, viewer timestamp mapping
Key commit: `8e78372`

Keep a canonical timeline offset separate from physical trim. Markers and console/server log sync must survive a clamped trim.

### P1-3 — Isolated agent-browser session

Why: inheriting `~/.agent-browser/config.json` or `AGENT_BROWSER_*` env can attach the proof session to the wrong Chrome / CDP / profile.

Alvaro: `src/browser/isolation.ts`
Key commit: `31adcaf` (plus later isolation hardenings)

Port: isolated session name, ignore incompatible env/config keys, prefer the Chrome that `agent-browser install` downloaded over system Chrome.

Skip: hard pin to exactly `0.34.0`.

### P1-4 — Reapply viewport after record starts

Why: recording can reset viewport. Ofri’s June fix; alvaro also has it.

Alvaro / Ofri: start/record path
Ofri commit: `f1db45b` on `OfriHarlev/proofshot` (`fix: reapply viewport after recording starts`)

Small. Do after P0-1 so we are on a known recorder.

### P2-1 — Session registry (`proofshot session list|clean`)

Alvaro: `src/commands/session.ts`, `src/session/registry.ts`

Useful when agents leak sessions. Safzan already kills `--run` servers on stop; this is the leftover browser/session inventory.

### P2-2 — Redact secrets from public reviewer text

Alvaro: `src/browser/redaction-policy.ts`, `src/browser/diagnostic-redaction.ts`

Only if `proofshot pr` starts leaking tokens/URLs into GitHub comments. Do not take the interaction-category taxonomy.

### P2-3 — Hashed session manifest + withhold comment after partial upload

Safzan already skips the comment when every upload fails. Alvaro adds per-session hashes and refuses comments after a partial upload.

Take the “no comment unless every listed asset landed” rule. Skip the rest of `src/artifacts/evidence.ts`.

## Do not port

| Alvaro area | Why skip |
| --- | --- |
| `src/artifacts/evidence.ts` + verdicts `PASS/FAIL/INCOMPLETE/BLOCKED` | Replaces human review with an agent-written judge |
| `src/commands/replay.ts` + `src/replay/case.ts` | Separate product. We want start/exec/stop/pr |
| `src/environment/tmux.ts` + multi-source env capture | Kentro/Hermes proof is a Next/dev-server URL, not tmux |
| Windows recording hard-ban | We are on macOS |
| `src/browser/command-policy.ts` / provenance action categories | Overfit policy layer |
| `src/commands/lifecycle.integration.test.ts` (1.9k lines) | Bring tests with each ported behavior, not this suite |
| Node 24 runtime pin | This machine is Node 22.14.0 |
| Cherry-picking 92 commits | Independent history from safzan; rebase/cherry-pick will fight the hang/leak/log-pump work we already have |

## Other forks (not alvaro, do not merge)

- `OfriHarlev/proofshot` — viewport-after-record + command timeouts. Pedigree is good; take P1-4 instead of merging.
- `doruksahin/proofshot` — `--cdp` for Electron. Only if we need to record Hermes desktop, not a local web app.
- `RoseBludd/proofshot` — Worker-thread `exec` deadlock. Revisit if `proofshot exec` hangs on macOS.

## Suggested order

1. P0-1 + P0-2 (recording actually produces proof)
2. P1-1 + P1-2 (viewer is reviewable)
3. P0-3 (safe `proofshot pr` from forks)
4. P1-3 + P1-4
5. P2 only when we feel the pain
