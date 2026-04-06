# Codex CLI

This fork publishes Codex CLI to npm as `@akramkhan/codex`.

It includes the `/loop` command so the same prompt can run again automatically after the
previous run completes.

## Install

```bash
npm install -g @akramkhan/codex
```

## Run

```bash
codex
```

## Loop prompts

Use `/loop <count> <prompt>` to repeat a prompt a fixed number of times.

```text
/loop 5 review and continue
/loop 100 hi
```

For effectively continuous runs, use a very large count:

```text
/loop 10000000 review and continue
```

Large loop counts are queued lazily, so Codex does not dump a huge pending list into the
UI up front.
