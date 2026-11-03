# Multimodal and Realtime Agents

## Role in the Agent System

This subsystem handles inputs and outputs beyond plain text: images, audio, speech, video, documents, and realtime streams. It normalizes modality-specific data into events and references that the runtime can process safely.

## 1. Modality pipeline

Ingestion → media validation → transcription/OCR/vision preprocessing → normalized content blocks → Context assembly → model invocation → generated media or speech → artifact registration → delivery.

## 2. Message model

Represent content as typed blocks rather than a single string:
- text;
- image reference;
- audio input/output reference;
- video frame with timestamp;
- document/page reference;
- structured JSON;
- tool call/result.

## 3. Realtime voice

Separate audio transport, turn detection, transcription, model response, TTS generation, interruption/barge-in, and session persistence. Text and audio should share a stable conversation identity while preserving modality-specific metadata.

## 4. Streaming

Support partial transcript, audio chunks, text deltas, tool progress, and final turn boundaries. Sequence numbers and timestamps are required for synchronization.

## 5. Model capability routing

A Provider Capability Descriptor should declare vision, audio input, audio output, realtime duplex, video/frame support, structured output, context window, and tool calling. Route only to compatible models.

## 6. Storage and privacy

Large media belongs in object storage, not graph state. Use signed references, retention policies, consent controls, transcription privacy rules, and access logs. Do not persist raw audio/video indefinitely by default.

## 7. Evaluation

Test transcription accuracy, speaker attribution, image grounding, timestamp alignment, interruption handling, audio latency, modality fallback, and end-to-end task success.
