# Digitaler Aluhut

> *"Digitaler Aluhut"* (German) ≈ "digital tinfoil hat" — except we actually built the infrastructure. Self-hosted everything, no cloud dependencies for core functionality, AI runs locally.

## Intent

Take back control over your digital life — not by going offline, but by running your own stack. Chat with AI models that don't phone home. Message people without a corporation reading along. Deploy apps that you own, on hardware that you own.

This isn't about paranoia. It's about the observation that you *can* run all of this yourself now, and it's surprisingly affordable. The entire setup — cluster, AI inference, messaging, coding tools — runs on a single mini PC for the price of ~2 years of cloud API subscriptions.

## Architecture

```
Internet → Cloudflare Tunnel (outbound-only) → k3s → Apps
                                                 ↕
                              Local AI (llama.cpp, whisper, ComfyUI)
                                    on Strix Halo / ROCm
```

- **Zero inbound ports.** All traffic flows through an encrypted outbound-only Cloudflare Tunnel. No port forwarding, no exposed attack surface.
- **AI inference is local.** No API keys, no rate limits, no data leaving the machine. Models run on-device with full GPU acceleration.
- **Everything is IaC.** Pulumi + TypeScript, git-versioned, CI/CD via GitHub Actions. Reproducible from scratch.
- **Auth on every route.** OAuth2-Proxy (GitHub) or Authelia (local accounts) — nothing is exposed without authentication.

## Components

| Repo | Role | What's inside |
|------|------|---------------|
| [homelab](https://github.com/digitaleraluhut/homelab) | Platform layer | k3s cluster, Cloudflare Tunnel, Traefik Gateway API, cert-manager, External Secrets, Authelia, OAuth2-Proxy |
| [local-ai](https://github.com/digitaleraluhut/local-ai) | AI layer | llama.cpp (LLM), whisper.cpp (STT), ComfyUI + FLUX (image gen), ROCm, systemd services |
| [homelab-apps](https://github.com/digitaleraluhut/homelab-apps) | Application layer | LobeHub (AI chat), Matrix + WhatsApp/Signal bridges + voice transcription, OpenCode (remote coding), Aftertouch (Bose speakers) |

Each app in `homelab-apps` is a standalone Pulumi project that references the shared infrastructure from `homelab`. The `local-ai` endpoints are consumed by apps like LobeHub and the Matrix transcription bot.

## Hardware

The entire stack runs on a [Bosgame M5](https://www.bosgamepc.com/products/bosgame-m5-ai-mini-desktop-ryzen-ai-max-395) — a mini PC with AMD Ryzen AI Max 395 (Strix Halo), 128 GB unified memory. Cost: ~1500€ in early 2026. Runs 120B parameter models locally. LLM + embeddings + STT + image generation all fit in one box.

### Don't have beefy hardware?

You don't need a Strix Halo to use this. The platform and application layers (`homelab` + `homelab-apps`) work on any machine that runs k3s — even a Raspberry Pi cluster or a cheap VPS.

For AI, you have options:
- **Use remote providers.** Point LobeHub or your apps at any OpenAI-compatible API — OpenAI, Anthropic, Mistral, or European providers like Aleph Alpha or DeepInfra. You still control the application layer: what's stored, what's logged, what's shared.
- **Run small models locally.** Even a laptop GPU can run 7B–14B models usefully. You won't get 120B quality, but you get full privacy.
- **Upgrade later.** The architecture is the same regardless of where inference happens. When affordable hardware catches up (and it will), swap the endpoint URL and you're fully local.

The key insight: even with remote model providers, self-hosting the *application layer* means no one profiles your usage patterns, stores your conversations, or trains on your data. Most GenAI services don't do this today — but you're not dependent on that promise continuing.
