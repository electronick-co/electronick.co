---
title: EdgeAI @ Burning Man at Google Dev Day
date: 2026-10-01
excerpt: Notes from a 10-minute talk on running a voice AI fully offline on a Raspberry Pi 5 inside a Burning Man helmet.
tags: [talk, edge-ai, raspberry-pi, burning-man]
---

A 10-minute talk for Google Dev Day about the [MindControl Helmet](/projects/burning-man-helmet-2026): what it takes to run a voice assistant with no cloud at all, on hardware you can wear in the desert.

I didn't go to Burning Man myself. The helmet was built for someone else to wear on the playa.

<figure>
  <img src="/images/mindcontrol/slides/title.jpg" alt="Title slide: EdgeAI @ Burning Man, Nick Velásquez (ElectroNick)" loading="lazy" />
</figure>

## The short version

- The desert has no reliable internet, so the AI has to live on the helmet.
- A Raspberry Pi 5 runs the whole pipeline: wake word, speech to text, LLM, text to speech.
- Qwen2.5-Instruct 1.5B at about 7 tokens/s is fast enough to hold a conversation. Bigger models were too slow; a reasoning model spent too long thinking out loud.
- The Hailo-10H NPU is not faster than the Pi 5 CPU at LLMs. Its job is to free the CPU for everything else the helmet does.

## Cloud first, then local

The first prototype ran on a laptop and compared three setups: a classic chain (faster-whisper, the Claude API, ElevenLabs), Gemini Live doing audio in and audio out with a single model, and a local Chatterbox voice clone. The cloud setups sounded good but needed internet. The local clone took 10 to 30 seconds per reply. Neither works on the playa.

<figure>
  <img src="/images/mindcontrol/slides/cloud.jpg" alt="Three laptop prototypes compared" loading="lazy" />
</figure>

## The local stack

<figure>
  <img src="/images/mindcontrol/slides/components.jpg" alt="The nine parts that make up the helmet" loading="lazy" />
  <figcaption>The helmet's nine off-the-shelf parts.</figcaption>
</figure>

| Step | Tool | Runs on |
|---|---|---|
| Wake word | openWakeWord, "hey mycroft" | CPU |
| Speech to text | faster-whisper `base.en`, int8 | CPU |
| LLM | Qwen2.5-Instruct 1.5B on `hailo-ollama` | Hailo-10H NPU |
| Text to speech | Piper, `lessac-medium` | CPU |

Text to speech is nearly free (about 0.03 s of compute per second of audio), so the LLM is the bottleneck. Streaming the reply one sentence at a time means the helmet starts talking before the answer is finished.

<figure>
  <img src="/images/mindcontrol/slides/pipeline.jpg" alt="The voice pipeline on one Raspberry Pi 5" loading="lazy" />
</figure>

<figure>
  <img src="/images/mindcontrol/slides/models.jpg" alt="Model comparison table for the five hailo-ollama models" loading="lazy" />
</figure>

## NPU vs. CPU

Public benchmarks show the Pi 5 CPU beating the Hailo-10H on every model `hailo-ollama` offers, for example 11.7 vs. 6.7 tokens/s on Qwen2.5 1.5B ([CNX Software](https://www.cnx-software.com/2026/01/20/raspberry-pi-ai-hat-2-review-a-40-tops-ai-accelerator-tested-with-computer-vision-llm-and-vlm-workloads/)). The NPU uses less power and leaves the CPU idle, which matters on a helmet whose CPU also handles speech, the camera stream, the phone portal and the LED display.

<figure>
  <img src="/images/mindcontrol/slides/npu-vs-cpu.jpg" alt="Bar chart of NPU vs CPU decode speed" loading="lazy" />
</figure>

Google's own [Gemma Translator](https://github.com/google-gemma/gemma-translator) makes the same point from the other side: Gemma 4 E2B on LiteRT-LM runs a full offline voice pipeline on a CPU-only Pi 5 at 7.6 to 9 tokens/s. One AI job fits on the CPU; the helmet runs several.

<figure>
  <img src="/images/mindcontrol/slides/gemma.jpg" alt="Comparison table: Gemma Translator vs MindControl helmet" loading="lazy" />
</figure>

## What broke

- `hailo-ollama` owns the NPU for its whole lifetime, and a second NPU process crashed it. Speech to text moved to the CPU.
- An onnxruntime bug in an optional voice filter crashed every audio frame. The fix was turning the filter off.
- Background loops died silently while the process stayed up, so systemd never restarted them.
- The wake word is "MY-kroft", not "Microsoft". That one took a while.

<figure>
  <img src="/images/mindcontrol/slides/lessons.jpg" alt="Four lessons from the helmet" loading="lazy" />
</figure>

## Bonus: the LED display

The 48×12 LED matrix on the back of the helmet came with a phone app. A Bluetooth snoop log, a decompiled React Native app and a JieLi SDK later, it turned out the display accepts commands from anyone in Bluetooth range, no authentication. A small Python library now drives it directly, including live video at 7.7 fps.

<figure>
  <img src="/images/mindcontrol/slides/ble-display.jpg" alt="Four reverse-engineering steps for the LED display" loading="lazy" />
</figure>

## Takeaways

- Choose the model for latency, not size.
- Give the LLM its own chip when the CPU has other work to do.
- Stream replies in sentences so speech starts early.
- Plan for silent failures: guard every background loop and let systemd retry forever.

<a class="gh-link" href="https://github.com/electronick-co" target="_blank" rel="noopener"><svg width="14" height="14" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>github.com/electronick-co</a>
