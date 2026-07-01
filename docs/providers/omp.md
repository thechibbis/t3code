# Oh My Pi

This guide is for people who want to use Oh My Pi (omp) in T3 Code.

Oh My Pi is a coding agent that runs locally and exposes an ACP (Agent Client Protocol) server over stdio. T3 Code connects to it via `omp acp`.

## Prerequisites

Install Oh My Pi and authenticate:

```bash
omp auth login
```

This stores credentials under `~/.omp`. T3 Code uses these local credentials automatically — no API keys or tokens to configure.

## I Only Use One omp Account

Use the default provider.

In Settings, your omp provider can stay like this:

```text
Display name: Oh My Pi
Binary path: omp
```

An empty `Binary path` means T3 Code searches `PATH` for `omp`.

## I Want A Custom Binary Path

If `omp` is not on your `PATH`, or you want to use a specific build:

```text
Display name: Oh My Pi
Binary path: /usr/local/bin/omp
```

## How Authentication Works

omp uses a single auth method called `"agent"` — it reads credentials already configured under `~/.omp`. There is no API key or OAuth flow to wire in T3 Code.

If `~/.omp` has no valid credentials, the first session creation will fail with an ACP error. Run `omp auth login` to fix this.

## Models

omp exposes 80+ models across `opencode-zen/*` and `umans/*` providers. T3 Code discovers these dynamically when starting an omp session — the model picker shows all available models.

The default model is `umans/umans-glm-5.2`. You can change it per thread or set a sticky default.

## Modes

omp supports two modes:

- **Default**: Standard ACP headless mode.
- **Plan**: Read-only planning mode that drafts a plan to a markdown file before any code changes.

Mode switching is handled through the standard ACP config options mechanism.

## Thinking Levels

omp supports configurable thinking levels: `off`, `auto`, `high`, `xhigh`. These appear in the traits picker when an omp session is active.

## Can I Switch Models In An Existing Thread?

Yes. omp supports in-session model switching, so you can change models without starting a new thread.

## Can I Resume An Existing omp Thread?

Yes. omp supports `session/load` and `session/resume`, so existing threads can be reopened and continued.

## Environment Variables

Use the provider's Environment variables section in Settings if you need to pass omp-specific configuration. Mark API keys or tokens as sensitive.
