---
name: hermes-agent-gateway-restart-recovery
description: On Hermes gateway restarts; detached restart + recovery.
version: 1.0.0
author: Leon (via Hermes Agent)
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [hermes-agent, gateway, restart, recovery, self-interrupt, windows, systemd]
    category: software-development
---

# Hermes Agent — safe gateway restart + recovery

## When to Use
- You (or your agent) are about to run `hermes gateway restart` / `gateway stop` from a
  session or automation that the same gateway serves.
- A restart left the gateway down, or a tool result said
  `[Orphan recovery: ... effect is UNKNOWN]`.
- You need a copy-paste, self-healing restart command for cron/CI or a chat agent.

Restarting the Hermes Agent **gateway** from a session the same gateway serves is a
self-interrupt: the command that orders the restart is a child of the process being
restarted, so it dies mid-flight. The restart half-completes, nothing is left to finish
bringing the gateway back, and the agent goes silent with an
`[Orphan recovery: ... effect is UNKNOWN]` tool result.

## Golden rule
Never run a bare in-session restart and assume it completes.
**Order a detached restart whose own script guarantees recovery** (restart → wait →
status → start-if-stopped), then verify from the session.

## Windows (gateway runs as a user process; login-item is only a logon safety net)
```powershell
powershell -NoProfile -Command "Start-Process -WindowStyle Hidden -FilePath powershell -ArgumentList '-Command','hermes gateway restart; Start-Sleep 15; hermes gateway start'"
```
`Start-Process` detaches from the dying session chain, so the follow-up `start` survives
the self-interrupt. Recovery typically ~7-30s.

## Linux/macOS (systemd user service: let the service manager restart it instead)
```bash
systemctl --user restart hermes-gateway   # the manager, not your process tree, does the restart
```
No self-interrupt risk when the unit is managed externally (Restart= settings re-elevate
it anyway). Only use the detached pattern above for a gateway running as a bare process.

## Mandatory verification (the step that was skipped in the original incident)
1. `hermes gateway status` — expect `✓ Gateway process running (PID: ...)`;
   on systemd: `systemctl --user is-active hermes-gateway`.
2. A round-trip ping on a connected channel (or check gateway.log for the startup line).
3. Treat any `Orphan recovery` result as "restart likely happened": re-check status
   BEFORE re-running side-effecting commands, and never re-run the restart blindly.

## Failure inventory (each with its avoidance)
- **Bare `hermes gateway restart` in-session** → gateway may stay down; the ordering
  child died with it. Avoid: detached launcher that itself contains `start`.
- **Ending the turn right after ordering a restart** → nobody watches the recovery.
  Avoid: status-verify before ending the turn.
- **Concluding "restart failed" from the orphaned tool result alone** → it usually
  means the opposite (it happened). Avoid: status read as the arbiter.
- **Relying on OS autostart (Windows login item / systemd Restart=)** → those only cover
  logoff/crash, not a mid-session restart you initiated. Avoid: in-band ensure step.
- **Double-poller hazard after forced recovery** (two gateways, one bot token) → check
  `gateway status` shows exactly one PID before telling the user it's back.

## One-liner (Windows, drop-in replacement for the naive command)
```powershell
powershell -NoProfile -Command "Start-Process -WindowStyle Hidden -FilePath powershell -ArgumentList '-Command','hermes gateway restart; Start-Sleep 15; hermes gateway start'"
```
