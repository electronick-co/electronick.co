---
title: EdgeAI @ Burning Man at Google Dev Day
date: 2026-10-01
excerpt: Notes from a 10-minute talk on running a voice AI fully offline on a Raspberry Pi 5 inside a Burning Man helmet.
tags: [talk, edge-ai, raspberry-pi, burning-man]
---

A 10-minute talk for Google Dev Day about the [MindControl Helmet](/projects/burning-man-helmet-2026): what it takes to run a voice assistant with no cloud at all, on hardware you can wear in the desert.

I didn't go to Burning Man myself. The helmet was built for someone else to wear on the playa.

## The short version

- The desert has no reliable internet, so the AI has to live on the helmet.
- A Raspberry Pi 5 runs the whole pipeline: wake word, speech to text, LLM, text to speech.
- Qwen2.5-Instruct 1.5B at about 7 tokens/s is fast enough to hold a conversation. Bigger models were too slow; a reasoning model spent too long thinking out loud.
- The Hailo-10H NPU is not faster than the Pi 5 CPU at LLMs. Its job is to free the CPU for everything else the helmet does.

## Cloud first, then local

The first prototype ran on a laptop and compared three setups: a classic chain (faster-whisper, the Claude API, ElevenLabs), Gemini Live doing audio in and audio out with a single model, and a local Chatterbox voice clone. The cloud setups sounded good but needed internet. The local clone took 10 to 30 seconds per reply. Neither works on the playa.

## The local stack

| Step | Tool | Runs on |
|---|---|---|
| Wake word | openWakeWord, "hey mycroft" | CPU |
| Speech to text | faster-whisper `base.en`, int8 | CPU |
| LLM | Qwen2.5-Instruct 1.5B on `hailo-ollama` | Hailo-10H NPU |
| Text to speech | Piper, `lessac-medium` | CPU |

Text to speech is nearly free (about 0.03 s of compute per second of audio), so the LLM is the bottleneck. Streaming the reply one sentence at a time means the helmet starts talking before the answer is finished.

## NPU vs. CPU

Public benchmarks show the Pi 5 CPU beating the Hailo-10H on every model `hailo-ollama` offers, for example 11.7 vs. 6.7 tokens/s on Qwen2.5 1.5B ([CNX Software](https://www.cnx-software.com/2026/01/20/raspberry-pi-ai-hat-2-review-a-40-tops-ai-accelerator-tested-with-computer-vision-llm-and-vlm-workloads/)). The NPU uses less power and leaves the CPU idle, which matters on a helmet whose CPU also handles speech, the camera stream, the phone portal and the LED display.

Google's own [Gemma Translator](https://github.com/google-gemma/gemma-translator) makes the same point from the other side: Gemma 4 E2B on LiteRT-LM runs a full offline voice pipeline on a CPU-only Pi 5 at 7.6 to 9 tokens/s. One AI job fits on the CPU; the helmet runs several.

## What broke

- `hailo-ollama` owns the NPU for its whole lifetime, and a second NPU process crashed it. Speech to text moved to the CPU.
- An onnxruntime bug in an optional voice filter crashed every audio frame. The fix was turning the filter off.
- Background loops died silently while the process stayed up, so systemd never restarted them.
- The wake word is "MY-kroft", not "Microsoft". That one took a while.

## Bonus: the LED display

The 48×12 LED matrix on the back of the helmet came with a phone app. A Bluetooth snoop log, a decompiled React Native app and a JieLi SDK later, it turned out the display accepts commands from anyone in Bluetooth range, no authentication. A small Python library now drives it directly, including live video at 7.7 fps.

## Takeaways

- Choose the model for latency, not size.
- Give the LLM its own chip when the CPU has other work to do.
- Stream replies in sentences so speech starts early.
- Plan for silent failures: guard every background loop and let systemd retry forever.

Code: [github.com/electronick-co](https://github.com/electronick-co)
