---
title: MindControl Helmet (Burning Man 2026)
description: A wearable helmet with a fully offline voice AI, animatronic eyes, a hacked Bluetooth LED display and a phone control portal, built on a Raspberry Pi 5.
date: 2026-08-27
status: ongoing
image: /images/mindcontrol/slides/title.jpg
tags: [burning-man, edge-ai, raspberry-pi, hailo, esp32, ble, reverse-engineering]
repos:
  - name: MindControlHelmet2_2026
    url: https://github.com/electronick-co/MindControlHelmet2_2026
    description: Raspberry Pi software, web portal, voice AI
  - name: HelmetCoprocessor
    url: https://github.com/electronick-co/HelmetCoprocessor
    description: ESP32 firmware for servos and LEDs
  - name: shining-helmet-bt-protocol
    url: https://github.com/electronick-co/shining-helmet-bt-protocol
    description: Reverse-engineered BLE display library
---

The MindControl Helmet is the second version of a wearable art piece built for Burning Man 2026. It talks back: you say "hey mycroft", ask a question, and the helmet answers out loud with no internet connection. It also has animatronic eyes, two LED strips, a front camera, a 48×12 LED matrix on the back, and a web portal that a phone uses to drive all of it.

| | |
|---|---|
| **Built for** | Burning Man 2026 |
| **Main computer** | Raspberry Pi 5 + Hailo-10H NPU (AI HAT+ 2) |
| **Co-processor** | Seeed XIAO ESP32-S3 (servos and LEDs) |
| **LLM** | Qwen2.5-Instruct 1.5B on `hailo-ollama` |
| **Speech** | openWakeWord, faster-whisper `base.en`, Piper TTS |
| **Connectivity** | Own WiFi hotspot, BLE to the LED display |

## Contents

1. [Architecture](#architecture)
2. [Hardware](#hardware)
3. [Voice AI](#voice-ai)
4. [Animatronic eyes and LEDs](#animatronic-eyes-and-leds)
5. [LED display (reverse engineered)](#led-display-reverse-engineered)
6. [Phone portal and networking](#phone-portal-and-networking)
7. [Reliability](#reliability)
8. [Previous versions](#previous-versions)
9. [History](#history)
10. [See also](#see-also)

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

The project is split across three repositories: the Pi software (<a class="gh-link" href="https://github.com/electronick-co/MindControlHelmet2_2026" target="_blank" rel="noopener"><svg width="14" height="14" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>MindControlHelmet2_2026</a>), the ESP32 firmware and host tools (<a class="gh-link" href="https://github.com/electronick-co/HelmetCoprocessor" target="_blank" rel="noopener"><svg width="14" height="14" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>HelmetCoprocessor</a>), and the BLE display library (<a class="gh-link" href="https://github.com/electronick-co/shining-helmet-bt-protocol" target="_blank" rel="noopener"><svg width="14" height="14" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>shining-helmet-bt-protocol</a>).

<figure>
  <img src="/images/mindcontrol/slides/pipeline.jpg" alt="Voice pipeline: wake word, speech to text and text to speech on the CPU, the LLM on the Hailo NPU" loading="lazy" />
  <figcaption>The voice pipeline. Only the LLM runs on the NPU; everything else shares the Pi's CPU.</figcaption>
</figure>

## Hardware

<div class="gallery">
  <figure><img src="/images/mindcontrol/components/pi5.jpg" alt="Raspberry Pi 5" loading="lazy" /><figcaption><strong>Raspberry Pi 5</strong>Main computer, runs every service</figcaption></figure>
  <figure><img src="/images/mindcontrol/components/hailo.jpg" alt="Raspberry Pi AI HAT+ 2 with Hailo-10H" loading="lazy" /><figcaption><strong>Hailo-10H (AI HAT+ 2)</strong>Runs the LLM</figcaption></figure>
  <figure><img src="/images/mindcontrol/components/camera.jpg" alt="Raspberry Pi Camera Module 3 NoIR" loading="lazy" /><figcaption><strong>Camera Module 3 NoIR</strong>Front camera, MJPEG to the phone</figcaption></figure>
  <figure><img src="/images/mindcontrol/components/audio.jpg" alt="Waveshare WM8960 Audio HAT" loading="lazy" /><figcaption><strong>WM8960 Audio HAT</strong>Microphone and speaker</figcaption></figure>
  <figure><img src="/images/mindcontrol/components/wifi.jpg" alt="TP-Link Archer T2U Plus" loading="lazy" /><figcaption><strong>TP-Link Archer T2U Plus</strong>WiFi access point for the phone</figcaption></figure>
  <figure><img src="/images/mindcontrol/components/xiao.jpg" alt="Seeed XIAO ESP32-S3 Sense" loading="lazy" /><figcaption><strong>XIAO ESP32-S3</strong>Drives the servos and LED strips</figcaption></figure>
  <figure><img src="/images/mindcontrol/components/servo.jpg" alt="Micro servo" loading="lazy" /><figcaption><strong>Micro servos</strong>Eye X, eye Y and eyelid</figcaption></figure>
  <figure><img src="/images/mindcontrol/components/leds.jpg" alt="24-pixel WS2812 LED ring" loading="lazy" /><figcaption><strong>WS2812 LEDs</strong>2 × 24-px goggle rings, ~200-px helmet strip</figcaption></figure>
  <figure><img src="/images/mindcontrol/components/shining.jpg" alt="SLShining LED helmet" loading="lazy" /><figcaption><strong>SLShining display</strong>48×12 RGB matrix over BLE</figcaption></figure>
</div>

Product photos from Raspberry Pi, Adafruit, Waveshare, TP-Link, Seeed Studio and an SLShining retailer. The servo and LED ring photos show representative parts.

## Voice AI

### Pipeline

Everything runs on the helmet:

1. **Wake word**: openWakeWord with the pretrained "hey mycroft" model (said "MY-kroft").
2. **Speech to text**: faster-whisper `base.en`, int8, on the CPU.
3. **LLM**: `qwen2.5-instruct:1.5b` served by `hailo-ollama` on the Hailo-10H.
4. **Text to speech**: Piper with the `en_US-lessac-medium` voice, played through the WM8960 speaker.

Replies stream sentence by sentence: each finished sentence goes to Piper while the LLM keeps generating, so speech starts before the answer is complete. Piper needs about 0.03 seconds of compute per second of audio, so the LLM's decode speed is the only real bottleneck.

<figure>
  <img src="/images/mindcontrol/slides/streaming.jpg" alt="The LLM decodes at about 7 tokens per second while Piper needs 0.03 seconds per second of speech" loading="lazy" />
</figure>

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

<figure>
  <img src="/images/mindcontrol/slides/npu-vs-cpu.jpg" alt="Bar chart: Hailo-10H NPU vs Raspberry Pi 5 CPU decode speed for five models" loading="lazy" />
  <figcaption>Decode tokens/s, NPU vs. CPU. Source: CNX Software, Jan 2026.</figcaption>
</figure>

### Earlier experiments

Before the Pi build, a laptop prototype (CloneVoiceTest) compared three approaches: faster-whisper with the Claude API and ElevenLabs, Gemini Live (one model, audio in and audio out), and a local Chatterbox voice clone. The cloud options needed internet, and the local voice clone took 10 to 30 seconds per reply on a CPU, so the final design moved everything onto the Pi with smaller, faster models.

<figure>
  <img src="/images/mindcontrol/slides/cloud.jpg" alt="Three laptop prototypes: cloud chain, Gemini Live, local voice clone" loading="lazy" />
</figure>

## Animatronic eyes and LEDs

The <a class="gh-link" href="https://github.com/electronick-co/HelmetCoprocessor" target="_blank" rel="noopener"><svg width="14" height="14" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>HelmetCoprocessor</a> firmware runs on a XIAO ESP32-S3 and talks to the Pi over a framed serial protocol. It passes RC PWM inputs through to three servos (eye X, eye Y, eyelid) and has three modes: BYPASS, AUTO and SLAVE (host control). It drives the two WS2812 strips with built-in effects (Matrix, Glitter, Comet, Flash, Gaze, Flashlight) or raw per-pixel streams, with a live 0–255 brightness per strip, since 200 pixels at full white would draw about 12 A.

The helmet strip is a single wire routed through eight physical segments, so the firmware keeps a measured map of which pixel index sits where on the helmet.

## LED display (reverse engineered)

The 48×12 panel on the back is a consumer SLShining bike-helmet display, normally controlled by the "Shining Display" Android app. The <a class="gh-link" href="https://github.com/electronick-co/shining-helmet-bt-protocol" target="_blank" rel="noopener"><svg width="14" height="14" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>shining-helmet-bt-protocol</a> library replaces the app:

- **Capture**: an Android Bluetooth HCI snoop log gave about 1,600 BLE packets.
- **Decode**: commands use a `[len, 0x00, cmd, sub, args]` framing; images upload as plain GIF/PNG files with a CRC32 header.
- **Decompile**: the app is React Native (Hermes bytecode) on a JieLi chipset. The JieLi authentication handshake only protects firmware updates; the display service accepts commands without it.
- **Control**: a Python library on `bleak` handles text, stills, animated GIFs and live streaming at about 7.7 fps. The vendor app's own animated-GIF upload is broken, so this is the only working uploader for the device.

Anyone in Bluetooth range can push frames to this panel.

<figure class="pixel">
  <img src="/images/mindcontrol/display/matrix-rain.gif" alt="Matrix rain animation rendered at 48 by 12 pixels" loading="lazy" />
  <figcaption>A matrix-rain animation generated by the library, at the panel's native 48×12 (upscaled).</figcaption>
</figure>

<div class="gallery-2">
  <figure class="pixel"><img src="/images/mindcontrol/display/gif-frames.png" alt="Animated hearts carved out of the Bluetooth capture" loading="lazy" /><figcaption>Frames of the app's "hearts" animation, carved straight out of the Bluetooth capture.</figcaption></figure>
  <figure class="pixel"><img src="/images/mindcontrol/display/live-preview.png" alt="Previews of text and generative effects" loading="lazy" /><figcaption>Previews of text and generative effects (plasma, sparkle, fills) before they are streamed.</figcaption></figure>
</div>

<figure>
  <img src="/images/mindcontrol/slides/ble-display.jpg" alt="Four steps: capture, decode, decompile, control" loading="lazy" />
</figure>

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

<figure>
  <img src="/images/mindcontrol/slides/lessons.jpg" alt="Four lessons: one NPU one process, remove the feature, silent task deaths, design for the crowd" loading="lazy" />
</figure>

## Previous versions

The LED eye work started on the helmet built for Maker Faire Shenzhen 2025 (<a class="gh-link" href="https://github.com/electronick-co/mfzs_LED_helmet" target="_blank" rel="noopener"><svg width="14" height="14" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>mfzs_LED_helmet</a>). The local voice stack started as <a class="gh-link" href="https://github.com/electronick-co/SteamTalk" target="_blank" rel="noopener"><svg width="14" height="14" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>SteamTalk</a>, a Pi 5 + Hailo voice assistant.

<figure>
  <img src="/images/mindcontrol/mfzs-devil-eyes.jpg" alt="Glowing red devil eyes graphic from the Maker Faire Shenzhen helmet" loading="lazy" />
  <figcaption>Eye graphics from the Maker Faire Shenzhen 2025 helmet.</figcaption>
</figure>

## History

| Date | Milestone |
|---|---|
| 2025-11 | Maker Faire Shenzhen LED helmet |
| 2026-04 | SteamTalk: first local voice assistant on Pi 5 + Hailo |
| 2026-06-10 | Shining Display BLE protocol fully reverse engineered |
| 2026-08-15 | Hotspot, audio HAT, Hailo NPU, local LLM and voice chat working |
| 2026-08-26 | Wake word, BLE display integration and new LED effects |
| 2026-08-27 | Goggles controls redesigned, reliability pass for field use |

## See also

- [Notes: EdgeAI @ Burning Man at Google Dev Day](/blog/google-dev-day-edgeai-burning-man)
- [Google's Gemma Translator](https://github.com/google-gemma/gemma-translator), a similar offline voice pipeline on a CPU-only Raspberry Pi 5
