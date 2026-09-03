# Qwen3-235B-A22B on Apple Silicon (MLX 4-bit) — The Full Lifecycle of Running 235B on Consumer Hardware

> A real-world record of running a 235B MoE with MLX on a Mac Studio (large-unified-memory Apple Silicon model): we installed it, we ran it, and **we ultimately retired it by choice**.
> The value of this book is the conclusion: being able to run it ≠ should run it. We document both the deployment method and the retirement reasons in full, so you can make the right call before you start.

## Configuration (as it was)

| Item | Value |
|---|---|
| Machine | Mac Studio (Apple Silicon, unified memory needs to be in the ≥128GB class) |
| Engine | MLX (mlx-lm) |
| Weights | Qwen3-235B-A22B-Instruct 4-bit (**124GB**) |
| Serving | Local port + launchd supervision (`RunAtLoad`+`KeepAlive`) |
| Role | Local fallback at the time (since retired) |

## Why we retired it (decided 2026-06, first-hand reasons)

1. **Insufficient stability**: A22B 4-bit was unstable under long sessions / continuous serving (response quality and service availability fluctuated). A fallback role demands the highest stability of all — a direct contradiction.
2. **124GB of weights eats the entire unified memory**: the headroom left for the OS + KV cache + other apps was too thin; the machine choked whenever it did anything else at the same time.
3. **Superseded by a better-fitting shape**: a 122B-class MoE (10B active) served stably at int4 on a dedicated inference box, with vision included — next to that, 235B-A22B on a Mac was "big but shaky," and retiring it was the rational choice.
4. A side finding: the same weights on dual GB10 with llama.cpp only reach ~11.7 tok/s (per community public benchmark methodology) (22B active is too heavy; the bandwidth arithmetic cannot clear the 50 tok/s line — see the selection matrix in this series' GPT-OSS-120B book). **This A22B activation class is not in the sweet spot on either kind of hardware.**

## If you still want to run it (the method still works)

```bash
pip install mlx-lm
# Community 4-bit conversion (124GB — check disk and memory before downloading)
mlx_lm.serve --model <235B-A22B-4bit weights> --port <port>
```
- For launchd supervision, use `KeepAlive.SuccessfulExit=false` (restart only on abnormal exit — avoids the debugging nightmare of processes resurrecting milliseconds after a kill; we learned this the hard way in the era of 8 services all on KeepAlive:true, when kill simply "didn't work").
- plists can get corrupted after a system reboot (we lived through one incident where every plist turned into a corrupted JSON stub on boot): add a config self-heal check to your supervision script.

## What this book is really about

**More parameters ≠ the right choice. The selection order should be: active-parameter bandwidth arithmetic (can it be fast) → stability soak (can it keep running) → only then talk about parameter scale.**
For the sweeter spot on Apple Silicon, see this series' Qwen3.6-27B book (much smaller, much more stable, and it has vision). 10x the active parameters of 235B did not give us 10x the value — only 10x the operational friction.

---
*RyanAI Lab · All numbers measured on our resident environment. Updated 2026-09. Issues welcome.*
