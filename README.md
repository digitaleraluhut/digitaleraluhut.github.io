# Digitaler Aluhut

> *"Digitaler Aluhut"* (German) ≈ "digital tinfoil hat" — except we actually built the infrastructure. Self-hosted everything, no cloud dependencies for core functionality, AI runs locally.

## Repositories

| Repo | Purpose | Key tech |
|------|---------|----------|
| [homelab](https://github.com/digitaleraluhut/homelab) | Kubernetes cluster, networking, auth, secrets | Pulumi, TypeScript, k3s, Cloudflare Tunnel, Authelia |
| [local-ai](https://github.com/digitaleraluhut/local-ai) | Local LLM/STT/image generation on AMD Strix Halo | llama.cpp, whisper.cpp, ComfyUI, ROCm, distrobox |
| [homelab-apps](https://github.com/digitaleraluhut/homelab-apps) | Apps deployed on the cluster | LobeHub, Matrix, OpenCode router, Aftertouch |

## Architecture

```
Internet → Cloudflare Tunnel (outbound-only) → k3s → Apps
                                                 ↕
                              Local AI (llama.cpp, whisper, ComfyUI)
                                    on Strix Halo / ROCm
```

- Zero inbound ports. All traffic through encrypted tunnel.
- AI inference is local — no API keys, no rate limits, no data leaving the machine.
- Everything is IaC (Pulumi + TypeScript), git-versioned, CI/CD via GitHub Actions.

## Replicating this

1. **Cluster**: Start with [homelab](https://github.com/digitaleraluhut/homelab). One `pulumi up` gives you k3s + Cloudflare Tunnel + auth + secrets.
2. **AI**: Set up [local-ai](https://github.com/digitaleraluhut/local-ai) on any machine with an AMD GPU. `rocm-llama qwen3-coder-30b` and you're running.
3. **Apps**: Deploy from [homelab-apps](https://github.com/digitaleraluhut/homelab-apps). Each app is a Pulumi project consuming shared infra components.

## Hardware

AMD Strix Halo with 128 GB unified memory. Runs 120B parameter models locally. The entire AI stack (LLM + embeddings + STT + image gen) fits in one box.
