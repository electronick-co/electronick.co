---
title: MindControl Helmet (Burning Man 2026)
description: A wearable helmet with a fully offline voice AI, animatronic eyes, a hacked Bluetooth LED display and a phone control portal, built on a Raspberry Pi 5.
date: 2026-08-27
status: ongoing
tags: [burning-man, edge-ai, raspberry-pi, hailo, esp32, ble, reverse-engineering]
---

The MindControl Helmet is the second version of a wearable art piece built for Burning Man 2026. It talks back: you say "hey mycroft", ask a question, and the helmet answers out loud with no internet connection. It also has animatronic eyes, two LED strips, a front camera, a 48×12 LED matrix on the back, and a web portal that a phone uses to drive all of it.

| | |
|---|---|
| **Event** | Burning Man 2026 |
| **Main computer** | Raspberry Pi 5 + Hailo-10H NPU (AI HAT+ 2) |
| **Co-processor** | Seeed XIAO ESP32-S3 (servos and LEDs) |
| **LLM** | Qwen2.5-Instruct 1.5B on `hailo-ollama` |
| **Speech** | openWakeWord, faster-whisper `base.en`, Piper TTS |
| **Connectivity** | Own WiFi hotspot, BLE to the LED display |
| **Repositories** | [MindControlHelmet2_2026](https://github.com/electronick-co/MindControlHelmet2_2026), [HelmetCoprocessor](https://github.com/electronick-co/HelmetCoprocessor), [shining-helmet-bt-protocol](https://github.com/electronick-co/shining-helmet-bt-protocol) |

## Contents

1. [Architecture](#architecture)
2. [Hardware](#hardware)
3. [Voice AI](#voice-ai)
4. [Animatronic eyes and LEDs](#animatronic-eyes-and-leds)
5. [LED display (reverse engineered)](#led-display-reverse-engineered)
6. [Phone portal and networking](#phone-portal-and-networking)
7. [Reliability](#reliability)
8. [History](#history)
9. [See also](#see-also)

## Architecture

The Raspberry Pi is the orchestrator. A phone joins the helmet's own hotspot and opens an HTTPS web portal; the portal drives the camera, the audio, the LLM chat and both pieces of helmet hardware.

```
                 phone (on the "mindcontrol" WiFi hotspot)
                                │  HTTPS + WebSocket
                                ▼
                 webportal (aiohttp, Raspberry Pi 5)
          │           │            │             │            │
       camera     mic/speaker   LLM + STT      serial         BLE
     (picamera2)   (WM8960)   (hailo-ollama,  ESP32 bridge   Shining Display
                               faster-whisper)  → servos,      48×12 panel
                                                  LED strips
```

The project is split across three repositories: the Pi software (MindControl), the ESP32 firmware and host tools (HelmetCoprocessor), and the BLE display library (shining-helmet-bt-protocol).

## Hardware

| Part | Role |
|---|---|
| Raspberry Pi 5 | Main computer, runs every service |
| Hailo-10H on the M.2 HAT+ | Runs the LLM |
| Camera Module 3 NoIR | Front camera, streamed to the phone as MJPEG |
| Waveshare WM8960 Audio HAT | Microphone and speaker |
| TP-Link Archer T2U Plus | Dedicated WiFi access point for the phone |
| Seeed XIAO ESP32-S3 | Drives three servos (eye X, eye Y, eyelid) and two WS2812 strips |
| WS2812 LEDs | Two 24-pixel goggle rings in series, plus a ~200-pixel helmet strip |
| SLShining "Shining Display" | 48×12 RGB matrix on the back, controlled over BLE |

## Voice AI

### Pipeline

Everything runs on the helmet:

1. **Wake word**: openWakeWord with the pretrained "hey mycroft" model (said "MY-kroft").
2. **Speech to text**: faster-whisper `base.en`, int8, on the CPU.
3. **LLM**: `qwen2.5-instruct:1.5b` served by `hailo-ollama` on the Hailo-10H.
4. **Text to speech**: Piper with the `en_US-lessac-medium` voice, played through the WM8960 speaker.

Replies stream sentence by sentence: each finished sentence goes to Piper while the LLM keeps generating, so speech starts before the answer is complete. Piper needs about 0.03 seconds of compute per second of audio, so the LLM's decode speed is the only real bottleneck.

### Model choice

`hailo-ollama` ships five models for the Hailo-10H:

| Model | Tokens/s | Verdict for voice |
|---|---|---|
| llama3.2:3b | 2.7 | Too slow for conversation |
| **qwen2.5-instruct:1.5b** | **6.8** | Chosen: best balance of speed and quality |
| qwen2:1.5b | 8.0 | Faster, but noticeably weaker answers |
| qwen2.5-coder:1.5b | 7.9 | Tuned for code, not chat |
| deepseek_r1_distill_qwen:1.5b | 6.8 | Long reasoning traces delay every answer |

Speeds are from [schwab.sh's AI HAT+ 2 benchmarks](https://www.schwab.sh/blog/hailo-ai-hat-benchmarks/) (decode, batch 1, Q4_0). A 1.5B model needs guardrails: replies are capped at 800 characters and 12 sentences, a turn stops after the same sentence repeats three times, and every turn has a 60-second hard timeout.

### NPU vs. CPU

Public reviews show the Pi 5's own CPU generating faster than the Hailo-10H on every one of these models (for example 11.7 vs. 6.7 tokens/s for Qwen2.5 1.5B, per [CNX Software](https://www.cnx-software.com/2026/01/20/raspberry-pi-ai-hat-2-review-a-40-tops-ai-accelerator-tested-with-computer-vision-llm-and-vlm-workloads/)). The NPU is still useful here because the CPU is busy with speech to text, the wake word, TTS, the camera stream and the web portal; offloading the LLM keeps those running.

### Earlier experiments

Before the Pi build, a laptop prototype (CloneVoiceTest) compared three approaches: faster-whisper with the Claude API and ElevenLabs, Gemini Live (one model, audio in and audio out), and a local Chatterbox voice clone. The cloud options needed internet, and the local voice clone took 10 to 30 seconds per reply on a CPU, so the final design moved everything onto the Pi with smaller, faster models.

## Animatronic eyes and LEDs

The HelmetCoprocessor firmware runs on a XIAO ESP32-S3 and talks to the Pi over a framed serial protocol. It passes RC PWM inputs through to three servos (eye X, eye Y, eyelid) and has three modes: BYPASS, AUTO and SLAVE (host control). It drives the two WS2812 strips with built-in effects (Matrix, Glitter, Comet, Flash, Gaze, Flashlight) or raw per-pixel streams, with a live 0–255 brightness per strip, since 200 pixels at full white would draw about 12 A.

The helmet strip is a single wire routed through eight physical segments, so the firmware keeps a measured map of which pixel index sits where on the helmet.

## LED display (reverse engineered)

The 48×12 panel on the back is a consumer SLShining bike-helmet display, normally controlled by the "Shining Display" Android app. The [shining-helmet-bt-protocol](https://github.com/electronick-co/shining-helmet-bt-protocol) library replaces the app:

- **Capture**: an Android Bluetooth HCI snoop log gave about 1,600 BLE packets.
- **Decode**: commands use a `[len, 0x00, cmd, sub, args]` framing; images upload as plain GIF/PNG files with a CRC32 header.
- **Decompile**: the app is React Native (Hermes bytecode) on a JieLi chipset. The JieLi authentication handshake only protects firmware updates; the display service accepts commands without it.
- **Control**: a Python library on `bleak` handles text, stills, animated GIFs and live streaming at about 7.7 fps. The vendor app's own animated-GIF upload is broken, so this is the only working uploader for the device.

Anyone in Bluetooth range can push frames to this panel.

## Phone portal and networking

- The Pi runs its own access point (`mindcontrol`) on a USB WiFi adapter, switchable between 2.4 GHz and 5 GHz from the portal. 5 GHz removed most stutter in a noisy RF environment.
- The portal is served over HTTPS because Chrome only allows microphone access (`getUserMedia`) in a secure context.
- Features: camera view, live audio from the helmet mic, push-to-talk to the helmet speaker, LLM chat (typed or voice), goggles and LED control, and the LED display.
- Android Chrome sent push-to-talk audio to the phone earpiece until echo cancellation, noise suppression and auto-gain were turned off.

## Reliability

Lessons from getting it field-ready:

- **One NPU, one process.** `hailo-ollama` holds the Hailo-10H for its whole lifetime. A second process using the NPU (Hailo's Whisper) made it crash, so speech to text runs on the CPU.
- **Remove the feature, not the bug.** An onnxruntime bug ([#23878](https://github.com/microsoft/onnxruntime/issues/23878)) in openWakeWord's optional voice-activity filter crashed every audio frame. Turning the filter off fixed it.
- **Silent task deaths.** Background loops died without crashing the process, so systemd never restarted anything. Every loop now catches, logs and continues.
- **Retry forever.** All services use `Restart=on-failure` with `StartLimitIntervalSec=0`, so a persistent failure keeps retrying instead of giving up after five attempts.
- **Design for the crowd.** There is no always-on listening; every question needs a fresh wake word so the helmet does not pick up strangers.

## History

| Date | Milestone |
|---|---|
| 2026-06-10 | Shining Display BLE protocol fully reverse engineered |
| 2026-08-15 | Hotspot, audio HAT, Hailo NPU, local LLM and voice chat working |
| 2026-08-26 | Wake word, BLE display integration and new LED effects |
| 2026-08-27 | Goggles controls redesigned, reliability pass for field use |

## See also

- [Notes: EdgeAI @ Burning Man at Google Dev Day](/blog/google-dev-day-edgeai-burning-man)
- [Google's Gemma Translator](https://github.com/google-gemma/gemma-translator), a similar offline voice pipeline on a CPU-only Raspberry Pi 5
