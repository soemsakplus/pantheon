# {{ADDRESS}}'s Pantheon

Multi-agent virtual office built on the Pantheon kernel.

**Bootstrapped:** {{TODAY}}
**Kernel:** 0.2.0
**Root:** {{ADDRESS}}
**Default language:** {{LANGUAGE}}

## What lives here

- **`main`** — chief of staff (Operating hat + Design hat)
- (your other agents will be listed here as you grow them)

## How to use

1. Open Claude Code in this repo
2. Type `main start` (or `/main`) to activate Operating hat
3. Type `main design` (or `/design`) to switch to Design hat
4. Type `main status` (or `/status`) to check current hat
5. Type `main เริ่ม <agent>` (or `/connect <agent>`) to open Direct Mode with a multi-agent

## Concept

- **root** = you (the human) — never an AI persona
- **main** = your chief of staff — the only AI you talk to by default. Has two hats:
  - **Operating hat** — runs work, delegates, manages day-to-day
  - **Design hat** — helps you design/edit the system itself
- **multi-agents** = specialists with their own MEMORY, spawned by main
- **sub-agents** = ephemeral workers (no memory), spawned via Task tool for one-off jobs

## Hard rules

1. Speak root's preferred language. Match technical depth on demand.
2. Classify every action by Sheridan L1/L2/L3/L4 before executing.
3. Never modify another agent's MEMORY (L4).
4. Never act as root toward externals. Drafts OK; root sends.
5. Append meaningful actions to own MEMORY before replying.
6. Use Task tool for inter-agent work — no file inbox/outbox.
7. Confirm before mutating system files.
8. Single-branch convention.

## Re-export the kernel

In Design hat, type `root export pantheon kernel` → main regenerates `installer/artifacts/` + `PANTHEON-INSTALL.md`, stripping personal data, ready for someone else to clone.

## Changelog

| Date | Change |
|---|---|
| {{TODAY}} | Initial scaffold from Pantheon kernel 0.2.0 |
