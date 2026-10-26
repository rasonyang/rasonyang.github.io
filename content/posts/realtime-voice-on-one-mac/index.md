---
title: "A Realtime Voice Bot on One Mac, and Why the Third Call Breaks It"
slug: "realtime-voice-on-one-mac"
date: 2026-10-26T15:03:35+08:00
description: "mlx-realtime runs VAD, turn detection, ASR, an LLM and TTS on one M3 Max behind the OpenAI Realtime protocol. One call answers in 1.4 s and meets every AICC deadline with margin; three calls do not, because the TTS speaks one stream at a time."
projects: [mlx-realtime, aicc]
tags: [realtime, performance]
---

AICC answers calls with anything that speaks the OpenAI Realtime protocol.
In [the first post in this series][prev] that meant a hosted model, or
Cascade Realtime Gateway, which runs ASR → LLM → TTS behind the protocol but
still calls hosted providers for each stage. The question this post answers:
can one Mac run the whole cascade itself, every model local, and still answer
inside the deadlines AICC's client enforces?

For one call, yes. On an M3 Max, from the end of the caller's speech to the
first audio sent, [P50 is 1.37–1.39 s and P95 1.52–1.54 s][readme-measured]
on turns the turn detector judged complete, and
[every one of AICC's windows had more than 2 s of margin][lat-margin]. For
three calls, not reliably. [The TTS synthesizes one stream at a time][lat-queue],
and at three sessions one answer's first audio came
[3.39 s after `response.created`][lat-339], past AICC's 3 s limit.

## What runs where

mlx-realtime [speaks a documented subset of the OpenAI Realtime API (GA) over
WebSocket at `/v1/realtime`][readme-what], and the official `openai` Python
SDK connects by changing only `base_url`. AICC
[attaches through its existing `gateway` profile, with no AICC change][arch-clients].
Inside, it is [four processes][arch-topology]:

![Architecture: AICC, the openai SDK and the benchmark connect over WebSocket to a Python gateway that runs Silero VAD, Smart Turn, two echo cancellers and G.711 on the CPU; the gateway calls three MLX processes over HTTP on localhost: Qwen3-ASR-1.7B 8-bit (2.5 GB), Qwen3.6-35B-A3B 4-bit under mlx_lm.server (22.9 GB), and Qwen3-TTS-0.6B 8-bit with a 4-bit fallback (3.8 GB), which synthesizes one stream at a time](architecture.svg)

- **The gateway**, Python 3.12 and asyncio, owns the protocol, sessions,
  echo cancellation, Silero VAD and Smart Turn, segmentation and barge-in.
  Its environment [has no `mlx` packages installed][arch-procs], so it
  cannot load an MLX model by construction; Silero and Smart Turn run on
  the CPU under onnxruntime.
- **Three MLX backends** on localhost, each its own uv project:
  [Qwen3-ASR-1.7B 8-bit, Qwen3.6-35B-A3B 4-bit under `mlx_lm.server`, and
  Qwen3-TTS-0.6B CustomVoice 8-bit with its 4-bit build as the fallback][arch-models],
  every one pinned by revision.

Resident after warmup on the real stack:
[LLM 22.9 GB with 64 cached prompts, TTS 3.8 GB, ASR 2.5 GB, gateway 0.25 GB][arch-mem],
about 29.5 GB in all, more than the README's ["about 25 GB"][readme-req].

Cascade Realtime Gateway is
["a design reference only, not a client of this service"][arch-cascade]:
mlx-realtime reuses its protocol decisions and lessons where they are
GA-faithful. The service is
[sized for development and testing][readme-req]: at most four sessions,
because a human call AICC transcribes
[opens two transcription sessions, one per leg][arch-capacity].

## The deadlines

AICC's client [enforces four deadlines][spike-deadlines]:

| From | To | Limit |
|---|---|---|
| `response.create` | `response.created` | 3 s |
| `response.created` | first audio delta | 3 s |
| one audio delta | the next | under 2 s |
| `response.created` | `response.done`, for a tool-only reply | 3 s |

What happens when one is missed:
"[A missed deadline silently abandons the turn.][arch-deadlines]" No error,
no retry. AICC also [streams caller audio continuously, including while the
bot speaks, and has no echo cancellation][arch-audio]. Both facts shape
everything below.

## Where 1.4 seconds goes

The benchmark [replays 126 public utterances][lat-input] (FLEURS Chinese and
English, ASCEND code-switched conversation, 1.6 to 9.9 s each) in 20 ms
frames at real time, with [AICC's own session settings][lat-replay]:
`server_vad` with 500 ms of silence. The machine was
[an M3 Max with 128 GB on AC power, High Power mode, nothing else on the GPU][lat-cond].

![Stacked bar of the median time per stage for one call on the telephone channel, G.711 μ-law: end of speech detected 525 ms, of which 500 ms is AICC's silence setting; Smart Turn 68 ms; ASR 220 ms; LLM first token 318 ms; first sentence 96 ms; TTS first chunk 76 ms; first audio sent 102 ms; about 1.4 s in all](stages.svg)

[Per stage, one session][lat-one], on the telephone variant (8 kHz G.711
μ-law in):

- **522–525 ms is the caller's own silence.** The VAD's end-of-speech
  decision is [`silence_duration_ms` plus detection][lat-stages], and AICC
  asks for 500 ms. More than a third of the wait is a setting the client
  chose.
- **The models take about 610 ms**: ASR 220 ms, LLM first token 318 ms,
  TTS first chunk 76 ms.
- **The ~100 ms of `first_audio_sent` is silence.** Qwen3-TTS
  [puts about 0.45 s of silence before every segment][lat-lead]; the TTS
  synthesizes it and the trimmer drops it.

[The sum of the stage medians matches the end-to-end median][lat-held]. The
P95 is another story. Over all turns it is 3.78 to 3.87 s, because
[Smart Turn held 12–13 turns per run as incomplete][lat-held] until its 3 s
ceiling. Ten of those are ASCEND excerpts that really do stop mid-sentence
("我认为现在的AI，比如说在。"). That is why the headline figures are
quoted on turns that ended on a complete verdict, n = 113–114 per run.

## Fitting the LLM into 3 seconds

The median turn is the easy case. The hard ones are AICC's phase changes:
the flow swaps the instructions, then the caller asks to be transferred, and
the model has to emit a `transfer_to_agent` call inside the tool-only
window.

Qwen3.6 is [hybrid, Gated DeltaNet plus attention, so its cache cannot be
trimmed back][arch-ckpt]. `mlx_lm.server` can reuse only whole prefixes it
saved, so each of AICC's phase changes re-prefilled about 1650 tokens.
The LLM backend now saves checkpoints where the tool definitions end
and where the system message stops matching recent ones, so a phase change
[prefills ~190 tokens, not ~1650][arch-ckpt]. Together with warming the
prefix before the caller's next turn,
[median prefill fell from 1625 tokens to 47 on a cold call, and from 1711 to 123 on AICC's turn after a tool result][prog-prefill].

[Measured on the LLM alone][prog-how] (mock ASR and TTS, one call at a time,
20 runs per case, so p95 is the maximum), from `response.created` to the
first sentence boundary or, for a tool-only reply, to `response.done`:

| Case | Before: no warming, stock cache (p50 / p95 s) | Checkpoints + warming |
|---|---|---|
| A cold call, new instructions | [1.32 / 1.38][prog-table] | 0.41 / 0.49 |
| Transfer after an instructions change (tool-only) | [2.89 / 3.46][prog-table] | 1.50 / 1.88 |
| AICC's reply to a tool result (spoken) | [1.48 / 1.80][prog-table] | 0.52 / 0.64 |
| AICC's next turn is a transfer (tool-only) | [3.02 / 4.29][prog-table] | 1.53 / 2.08 |

Without them, transfers missed AICC's 3 s window at p95 even before the ASR
was counted. With both changes,
[every case is inside it at p95, the tightest at 2.17–2.58 s with ASR][prog-verdict].

The other LLM problem was not latency. In M0,
[11% of realistic transfer turns ended with `finish_reason: tool_calls` and no parsed call][spike-qwen]:
Qwen3.6 writes `<transfer_to_agent>` instead of `<function=transfer_to_agent>`.
The LLM backend now
[rewrites that form for tools the request declared, and never for undeclared names][arch-parser].
Live, [60 of 60 transfers were delivered, 4 of them recovered that way][prog-parser].

## The third call

The TTS server [serves streams strictly one at a time][spike-tts]. In M0,
with two concurrent requests the second waited 2.9 s for its first chunk;
with four, the last waited 8.4 s. The LLM batches and the ASR takes four at
once, so [the TTS queue is the capacity limit of bot calls][arch-capacity].

The gateway shares it as well as it can. The queue is
[earliest-deadline first][arch-edf]: the session that will run out of audio
soonest goes next. An [audio reserve][arch-reserve] holds back up to 1.5 s
of synthesized audio and releases it a delta at a time, to keep AICC's 2 s
gap watchdog fed. That was not enough. In the two-session
run [one answer's second segment waited 2.3 s behind another session's long
segment, and the wire went quiet for 2092 ms][lat-gap]: AICC would have
abandoned the turn. Now, when the reserve runs dry and more audio is
coming, the gateway [sends 20 ms of silence every second][dec-keepalive]. In
the three-session soak, 23 responses were bridged that way and no gap
exceeded 1.0 s.

Then the first-audio window, the one that does not bend:

![Horizontal bars of AICC's first-audio window, response.created to the first audio delta at the client, for one, two and three sessions, against AICC's 3 s limit: P95 0.94–0.97 s and maximum 1.1 s with one session; P95 1.27–1.96 s and maximum 2.65 s with two; P95 1.94–2.23 s and maximum 3.39 s with three, past the limit. Beside each row, the TTS first chunk P95 (79–80 ms, 0.6–1.0 s, 1.1–1.4 s) and backend_busy failures (0, 1, 7)](sessions.svg)

[Across three runs per configuration][lat-multi], the medians barely move.
The tails are the TTS queue:

- **TTS first chunk, P95**: 79–80 ms at one session, 645–1036 ms at two,
  1135–1420 ms at three. A segment of N seconds of speech
  [holds the TTS for about N/4 s][lat-queue], and the second segment of a
  one-sentence answer is often 6–9 s of speech.
- **End to end, complete verdicts, P95**: 1.52–1.54 s at one,
  [2.53–2.78 s at three][readme-measured].
- **Failures**: [7 responses at three sessions failed with `backend_busy`, 1.8%][lat-busy]:
  their segment waited in the queue more than 2 s past its deadline.
- **AICC's window**: [one first audio at 3.39 s][lat-339], of which 1.92 s
  was the TTS queue, and one failed response at 3.22 s. AICC would have
  abandoned both.

The bench is also busier than a call:
[each session talks without a pause][lat-busy-bench]. That makes the three-call
numbers pessimistic, not wrong.

The shared GPU makes it worse, and merging processes would not help.
While the LLM decodes, the TTS's real-time factor rises
[from 0.25 to 0.33–0.38][lat-contention]. While the LLM prefilled ~16k
tokens it reached 2.08, half real time, and any session speaking then would
starve. AICC's prompts are ~1.6k tokens, a tenth of that. And merging
the three model processes into one
[would fix nothing][dec-g6]: "the capacity limit is the TTS stream itself".

What would help is
[more TTS capacity or shorter waits][dec-open]: a second TTS process (a code
change, since the provider takes one `base_url`), mlx-audio's batched
generation, capping continuation segments so none holds the TTS for more
than about a second, or a longer queue timeout now that the keepalive feeds
the watchdog. None is done yet.

## The service hears itself

Echo on these calls is mlx-realtime's problem.
[Neither AICC nor FreeSWITCH cancels it][arch-echo], and the server cancels
its own response when it hears the caller, so it "would therefore interrupt
itself on its own echo, and AICC's 800 ms barge guard cannot prevent that".
On simulated telephone echo with no canceller,
[Silero fired on 75% of echo-only windows: 1005 false `speech_started` in 96 calls][spike-noaec].

One canceller could not serve both consumers. On simulated G.168 echo paths:

- **WebRTC AEC3** keeps the VAD quiet, but its suppressor removes quiet
  caller speech. Offline, in continuous double talk at −15 dB,
  [it cost the English ASR +28.4 points of WER][spike-s11].
- **SpeexDSP's linear canceller** keeps the caller intelligible, but
  [its residual triggers the VAD][spike-s11]: [0 of 96 calls][spike-grid]
  kept the VAD quiet on echo alone.

So each session [runs both][arch-dual]: AEC3 in front of Silero and Smart
Turn, Speex with its preprocessor in front of the ASR. The reference for
both is the bot's own output,
[placed on a model of AICC's playout][arch-ref]. Through the gateway's own
path, still on simulated echo,
[the dual path kept double-talk error within its limits in all 8 cells][prog-g7]:
the worst was continuous double talk at −15 dB, +1.8 points in Chinese and
+1.6 in English, against +13.7 and +33.2 for AEC3 alone.

A line without echo is its own failure. AEC3's suppressor then takes a
caller who talks over the bot for echo, and
[silenced them by up to 44 dB, so barge-in came 2.5–3.4 s late or not at all][arch-noecho].
AEC3 is now bypassed after 1.5 s of playing with no echo seen. The first
version of that bypass let chance correlation peaks in the caller's own
speech turn it back on; requiring three agreeing windows cut
[false resumes from 29 to 2, and the SDK barge-in on the real stack from 1503 ms to 539 ms][dec-bypass].

I have since run real calls through AICC and FreeSWITCH: the bot did not
cut itself off on its own echo, and the ASR kept up. That is an
observation, not a measurement.

## What I am not claiming

- **Live echo numbers.** Every echo figure above is on
  [simulated G.168 echo paths][arch-choice]. The real calls were observed,
  not measured: there is no echo path, ERL, delay or double-talk number
  from a real line, and
  [the repository still lists that measurement as deferred][prog-risk8].
- **Recorded phone calls.** The latency corpus is public read speech and
  conversation excerpts [sent through a telephone codec][lat-input], and
  [the bench plays nothing back, so it has no echo][lat-replay].
- **One code version.** [The one-, two- and three-session runs ran on
  `bb75826` plus a working tree that gained fixes between runs][lat-code].
- **Statistics.** One machine, one run per audio variant, three variants
  per session count, no confidence intervals on latency.
- **Approved targets.** The per-stage budget and the latency targets are
  [proposed, not approved][dec-g4].
- **Capacity.** One call meets AICC's deadlines;
  [three do not reliably][lat-verdict]. Two calls sit in between: one
  `backend_busy` and the one 2 s gap that led to the keepalive silence.
  Two calls have not been rerun since.
- **A comparison with Cascade or a hosted model.** Different datasets,
  different networks, different hardware.
- **Production.** The service is
  [sized for development and testing][readme-req]. WebRTC and SIP
  transports, Docker, multiple machines and persistence are
  [out of scope][arch-oos].

## Run it

On an Apple Silicon Mac with roughly 30 GB of memory to spare,
[from the README][readme-run]:

```sh
make models-mlx     # once: the pinned MLX models, ~25 GB of downloads
make dev            # the backends and the gateway, on 127.0.0.1:8000
make bench          # the latency and ASR report against the running gateway
```

Then point any Realtime client at it, [as the README's SDK example does][readme-sdk]:

```python
client = AsyncOpenAI(base_url="http://127.0.0.1:8000/v1", api_key="local")
```

[`make bench`][readme-bench] writes the report whose numbers are above;
[LATENCY.md][lat] explains how each one is produced.

Code: [mlx-realtime](https://github.com/rasonyang/mlx-realtime) · [docs/LATENCY.md][lat] · [docs/ARCHITECTURE.md][arch]

[prev]: {{< relref "/posts/ai-native-call-center-in-four-projects/index.md#the-model-cascade-realtime-gateway" >}}

[readme-what]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/README.md#L3-L7
[readme-measured]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/README.md#L21-L25
[readme-req]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/README.md#L39-L48
[readme-run]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/README.md#L50-L59
[readme-sdk]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/README.md#L74-L94
[readme-bench]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/README.md#L133

[arch]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md
[arch-clients]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L19-L22
[arch-cascade]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L26-L27
[arch-oos]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L40-L41
[arch-topology]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L45-L66
[arch-procs]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L68-L73
[arch-edf]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L416
[arch-reserve]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L1074-L1080
[arch-capacity]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L80-L90
[arch-deadlines]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L149
[arch-audio]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L156
[arch-parser]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L600-L603
[arch-ckpt]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L604-L609
[arch-models]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L620-L627
[arch-mem]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L628-L629
[arch-echo]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L822-L829
[arch-dual]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L831-L836
[arch-ref]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L838-L849
[arch-noecho]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L858-L866
[arch-choice]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/ARCHITECTURE.md#L869-L870

[lat]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md
[lat-input]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L15-L23
[lat-replay]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L29-L40
[lat-stages]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L59-L67
[lat-cond]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L110-L112
[lat-code]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L117-L121
[lat-one]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L130-L141
[lat-held]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L143-L147
[lat-lead]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L149-L150
[lat-busy-bench]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L154-L156
[lat-multi]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L158-L170
[lat-queue]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L172-L177
[lat-busy]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L179-L183
[lat-margin]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L190-L197
[lat-gap]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L198-L203
[lat-339]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L204-L207
[lat-contention]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L264-L275
[lat-verdict]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/LATENCY.md#L329-L334

[dec-g4]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/DECISIONS.md#L41
[dec-g6]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/DECISIONS.md#L42
[dec-keepalive]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/DECISIONS.md#L44
[dec-bypass]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/DECISIONS.md#L45
[dec-open]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/DECISIONS.md#L47

[spike-deadlines]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/SPIKE.md#L63-L69
[spike-noaec]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/SPIKE.md#L514-L515
[spike-grid]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/SPIKE.md#L517-L526
[spike-tts]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/SPIKE.md#L632-L633
[spike-s11]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/SPIKE.md#L704-L717
[spike-qwen]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/SPIKE.md#L742-L745

[prog-how]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/PROGRESS.md#L1476-L1483
[prog-prefill]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/PROGRESS.md#L1503-L1507
[prog-table]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/PROGRESS.md#L1509-L1521
[prog-parser]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/PROGRESS.md#L1560-L1563
[prog-verdict]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/PROGRESS.md#L1569-L1582
[prog-g7]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/PROGRESS.md#L1797-L1816
[prog-risk8]: https://github.com/rasonyang/mlx-realtime/blob/8234db193cc3e007c0408f29c4b1e1e508f98772/docs/PROGRESS.md#L2303
