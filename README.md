# hermes-agent-gateway-restart-recovery

A Hermes Agent skill: safely relaunching the Hermes Agent gateway process from a session it serves, without it staying down.

## The problem
`hermes gateway restart` run from inside a served session is a self-interrupt: the ordering process is a child of the process being cycled and dies with it. The service half-restarts, nobody finishes bringing it back, and your agent goes silent (Orphan recovery: effect UNKNOWN).

## The fix
Order a **detached** relaunch whose own script carries the recovery (cycle, wait, ensure-up), then status-verify from the session. Copy-paste one-liners for Windows (Start-Process) and Linux/macOS (systemctl --user) are in SKILL.md.

## Install (as a Hermes skill)
```bash
git clone https://github.com/leonininder/hermes-agent-gateway-restart-recovery ~/.hermes/skills/hermes-agent-gateway-restart-recovery
```
(Locate yours with: hermes profile show default)

MIT license.
