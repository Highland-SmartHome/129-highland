# HA Assist Pipeline — Design Document

## Overview

The Highland voice assistant uses Home Assistant's Assist pipeline with a custom "Marvin" persona — dry, understated, cosmically resigned (Hitchhiker's Guide to the Galaxy). The pipeline combines local processing where practical (wake word, STT) with cloud services where quality matters (TTS, deep conversation).

---

## Architecture

```
┌─────────────────────┐
│   Voice Satellite   │
│   (M5Stack ATOM     │
│   Echo or ViewAssist)│
└──────────┬──────────┘
           │ Audio stream
           ▼
┌─────────────────────┐
│   Wake Word         │
│   (openWakeWord)    │
│   Custom: "Marvin"  │
└──────────┬──────────┘
           │ Wake detected
           ▼
┌─────────────────────┐
│   STT               │
│   (Local Whisper)   │
└──────────┬──────────┘
           │ Transcribed text
           ▼
┌─────────────────────┐
│   Conversation      │
│   Agent (2-tier)    │
│   ├─ Local: Ollama  │
│   │  (home-aware)   │
│   └─ Cloud: Gemini  │
│      (general)      │
└──────────┬──────────┘
           │ Response text
           ▼
┌─────────────────────┐
│   TTS               │
│   (Google Cloud     │
│   Neural2 - UK)     │
└──────────┬──────────┘
           │ Audio response
           ▼
┌─────────────────────┐
│   Voice Satellite   │
│   (Speaker output)  │
└─────────────────────┘
```

---

## Voice Satellites

### Hardware

**M5Stack ATOM Echo (×2 on hand):**
- ESP32-based
- Built-in microphone and speaker
- ESPHome firmware
- Used for initial validation and low-traffic areas

**Echo Show + LineageOS + ViewAssist (planned):**
- Repurposed Echo Show hardware
- LineageOS Android for full control
- ViewAssist for visual feedback + voice
- Target: 7 units for whole-house coverage

### Satellite Placement (Target)

| Location | Hardware | Notes |
|----------|----------|-------|
| Kitchen | ViewAssist | High traffic, visual useful for timers/recipes |
| Living Room | ViewAssist | Media control, general queries |
| Primary Bedroom | ATOM Echo | Low visual need, voice-only sufficient |
| Office | ViewAssist | Calendar, tasks, home control |
| Garage | ATOM Echo | Basic commands, durability |
| Basement | ATOM Echo | Basic commands |
| Outdoor (covered) | TBD | Weather-resistant option needed |

---

## Wake Word — openWakeWord

### Custom Wake Word: "Marvin"

Using openWakeWord's synthetic training pipeline to create a custom wake word model for "Marvin" (or "Hey Marvin").

**Training approach:**
- Generate synthetic audio samples using TTS
- Train on positive examples + negative examples (common words, background noise)
- Validate false positive/negative rates
- Iterate until acceptable accuracy

**Fallback:** If custom training proves difficult, use a built-in wake word ("Hey Jarvis", "Alexa", etc.) temporarily.

### Deployment

openWakeWord runs as part of the Assist pipeline in HA. Processing happens on HAOS — no external dependency.

---

## Speech-to-Text — Local Whisper

### Why Local

- Privacy: Audio never leaves the network
- Latency: Faster than cloud round-trip for short commands
- Cost: No per-request charges

### Deployment

Whisper runs on the Edge AI box via a container. HA connects to it as an external STT provider.

**Model selection:** `whisper-small` or `whisper-medium` — balance accuracy vs. speed. Test during implementation.

### HA Configuration

```yaml
stt:
  - platform: whisper
    model: small
    language: en
```

---

## Conversation Agent — Two-Tier

### Architecture

Two conversation backends, routed based on query type:

| Query Type | Backend | Rationale |
|------------|---------|----------|
| Home control ("turn on kitchen lights") | HA built-in intent | Direct, fast, reliable |
| Home-aware queries ("is the garage door open?") | Local Ollama | Privacy, speed, home context |
| General knowledge ("what's the capital of France?") | Cloud Gemini | Quality, breadth |
| Complex reasoning | Cloud Gemini | Capability |

### Local: Ollama on Edge AI Box

**Model:** TBD — 13B to 32B parameter range. Candidates:
- Mistral 7B (fast, capable)
- Llama 2 13B (good balance)
- Mixtral 8x7B (if memory allows)

**Hardware constraint:** Edge AI box has 32GB RAM. Model must fit with headroom for Coral TPU workloads (CPAI).

**Home context:** Ollama receives a system prompt with:
- Current home state (lights, doors, sensors)
- Recent events (motion, arrivals)
- Time of day, weather summary
- Marvin persona instructions

### Cloud: Gemini

For queries that exceed local model capabilities or require broad knowledge.

**Routing logic:** Start simple — if Ollama response confidence is low or query is clearly general knowledge, escalate to Gemini. Refine based on observed patterns.

---

## Text-to-Speech — Google Cloud Neural2

### Voice Selection

**Google Cloud TTS Neural2** with a UK English accent. This provides a unified voice across all responses — consistent "Marvin" character.

**Voice ID:** `en-GB-Neural2-B` (male, British) or similar. Test options during implementation.

### Marvin Persona

The TTS voice is just the delivery mechanism. The *persona* comes from the conversation agent's response text. System prompt instructs:

> You are Marvin, the home assistant. You are highly intelligent but perpetually underwhelmed by everything. Your responses are dry, understated, and tinged with cosmic resignation. You help efficiently but always convey a sense that you've seen it all before and find most tasks beneath you. Never be rude or unhelpful — just wearily competent.

**Example responses:**
- "The garage door is open. Again. I've closed it."
- "It's 72 degrees. Practically tropical, by local standards."
- "I've turned on the kitchen lights. You're welcome, I suppose."

---

## Pipeline Integration

### HA Assist Pipeline Configuration

```yaml
assist_pipeline:
  - name: Marvin
    wake_word:
      engine: openwakeword
      model: marvin
    stt:
      engine: whisper
      model: small
    conversation:
      engine: ollama
      model: mistral
    tts:
      engine: google_cloud
      voice: en-GB-Neural2-B
```

### Satellite Configuration

Each satellite is configured to use the "Marvin" pipeline:

```yaml
voice_assistant:
  microphone: ...
  speaker: ...
  pipeline: Marvin
```

---

## Speaker Recognition (Future)

**Monitoring:** `github.com/EuleMitKeule/speaker-recognition`

Potential future path for satellite-based user identification using Resemblyzer neural voice embeddings. Would enable:
- Personalized responses ("Good morning, Joseph")
- User-specific automations (presence detection via voice)
- Access control (certain commands require specific voices)

**Status:** Backlog. Evaluate after baseline pipeline is stable.

---

## Implementation Sequence

1. **HAOS baseline** — Confirm Assist pipeline works with default components
2. **Whisper STT** — Deploy on Edge AI box, integrate with HA
3. **Google Cloud TTS** — Configure Neural2 voice
4. **openWakeWord** — Train custom "Marvin" wake word
5. **Ollama** — Deploy on Edge AI box, create home-aware system prompt
6. **Satellite deployment** — Start with ATOM Echo units
7. **ViewAssist** — LineageOS on Echo Show hardware
8. **Gemini integration** — Add cloud fallback for complex queries

All of this follows baseline infrastructure (Communication Hub → HAOS → Workflow) being stable.

---

## Open Items

| Item | Notes |
|------|-------|
| Ollama model selection | Test candidates when Edge AI box is built |
| Whisper model size | Balance accuracy vs. latency |
| Custom wake word training | openWakeWord synthetic pipeline |
| Google Cloud TTS voice | Test UK male options |
| LineageOS on Echo Show | Validate feasibility |
| Speaker recognition | Evaluate after baseline |

---

*Last Updated: 2026-03-11*
