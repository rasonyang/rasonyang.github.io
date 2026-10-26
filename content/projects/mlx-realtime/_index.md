---
title: "mlx-realtime"
description: "The OpenAI Realtime protocol outside; VAD → ASR → LLM → TTS on one Apple Silicon Mac inside."
---

Local realtime voice service for Apple Silicon. It speaks a subset of the
OpenAI Realtime (GA) WebSocket protocol and runs every model on the Mac:
Silero VAD and Smart Turn, then Qwen3-ASR, Qwen3.6 and Qwen3-TTS on MLX.

Repository: [rasonyang/mlx-realtime](https://github.com/rasonyang/mlx-realtime)
