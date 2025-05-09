# [ai-voice-agent](https://learn.deeplearning.ai/courses/building-ai-voice-agents-for-production/lesson/fizgb/introduction)

---

End-to-End Voice Assistant Stack (Proof-of-Concept Outline)

    Speech-to-Text: Open-source Whisper

    Local LLM: Mistral 7B quantized (llama.cpp)

    Text-to-Speech: Silero TTS or Coqui TTS

    Voice Activity Detection: Silero VAD

All components run locally, require no paid keys, and can be orchestrated from the command line or a single Python script. Once you’ve quantized the model, you can launch FastChat: