# Localized Edge-AI Assistant (Project BMO)

## Project Overview
Project BMO is an autonomous, 100% offline, locally-hosted AI entity. While themed around a fictional character, the primary engineering objective of this project is demonstrating secure, localized LLM inference without reliance on third-party cloud APIs. 

This project features a custom-trained Llama-3 personality, a natively cloned neural TTS voice, local computer vision, and an agentic logic loop running entirely on a Raspberry Pi 5.

## Architecture
* Compute: Raspberry Pi 5 (8GB RAM), Debian Trixie (Kernel 6.12).
* Brain (LLM): llama3.2:3b (Fine-tuned via Unsloth/QLoRA), deployed via Ollama (.gguf Q4_K_M).
* Voice (TTS): Piper TTS ONNX model (Approx. 1300-epochs).
* Ears (STT & Wake Word): Vosk/OpenWakeWord for continuous listening, Faster-Whisper (tiny.en, int8) for transcription.
* Vision (VLM): Moondream via Ollama, utilizing Raspberry Pi Camera Module 3.
* UI/Display: Pygame running natively via KMSDRM/Wayland (Direct Hardware Rendering).

## Fine-Tuning & Data Processing
Instead of relying on generic system prompts, the model weights were fine-tuned to restrict outputs to a specific operational persona.
* Data Scraping: Developed a custom Python scraper (BeautifulSoup/MediaWiki API) to extract 845 lines of canonical dialogue, preserving conversational context.
* Model Fine-Tuning: Formatted data into ChatML and utilized Unsloth (RTX 4080 SUPER, 16GB VRAM) to execute a 300-epoch QLoRA fine-tune. 

## Voice: Audio Isolation & TTS Training
A custom-trained neural vocoder was engineered from raw audio files.
* Vocal Isolation: Extracted MKV episodes to WAV, utilizing UVR5 and the MDX-Net model to surgically strip ambient noise and sound effects.
* Forced Alignment: Leveraged WhisperX with a custom fuzzy-matching script to compare scraped text against clean audio tracks, automatically generating 306 timestamped audio snippets.
* Voice Training: Compiled the monotonic_align C++ engine from source and trained a custom .onnx voice model via Piper TTS for 1300 epochs.

## Sensory Input & Processing
* Hardware Resampling Bridge: Engineered a custom Python sounddevice bridge utilizing NumPy decimation (pcm[::3]) to downsample 48kHz microphone audio to 16kHz in real-time, preventing ALSA Invalid Sample Rate crashes.
* Wake Word & STT: Utilizes a custom .tflite model for 100% offline triggering, handing off to faster-whisper (CPU-bound) for sub-second transcription.
* Computer Vision: Triggered by vocal keywords, `rpicam-still` silently captures the environment, passing the frame to the Moondream (1.6GB) Vision-Language Model to generate descriptions of the physical world.

## Challenges & Resolutions
Building a localized OS on a Raspberry Pi 5 presented significant hardware and software hurdles:

1. The ReSpeaker 2-Mics HAT I2C Failure
Issue: The Pi 5's RP1 architecture broke legacy Seeed Studio drivers. Attempts to manually bind the tlv320aic3104 codec via I2C (0x18) resulted in "Device or resource busy" loops.
Resolution: Pivoted to a Sabrent USB Audio Adapter. Spliced a traditional headphone driver and integrated a 3.5mm pin-mic. 

2. Pygame Audio Locks & ALSA Conflicts
Issue: `pygame.init()` spawned a background audio mixer that seized exclusive control of the USB soundcard, locking `arecord` and `aplay`.
Resolution: Stripped Pygame initialization down to `pygame.display.init()`. Implemented aggressive `threading.Event()` logic (`stop_listening.set()`) to force the microphone stream to yield hardware control prior to the STT recording phase.

3. KMSDRM Direct Rendering
Issue: Pygame failed to initialize via SSH on the 4-inch DSI screen due to Wayland compositor conflicts.
Resolution: Reverted the Pi 5 to X11, enabled Console Autologin, and injected `MESA_LOADER_DRIVER_OVERRIDE=vc4` into the systemd service file to force direct hardware rendering for the UI.

## Next Steps
* Physical Fabrication: 3D print the hardware case (Scheduled post-certification).
* Touch Integration: Integrate TTP223 capacitive touch sensors via GPIO pins to trigger the AI logic loop via physical interactions.
* Agentic Capabilities Integration: Upgrade from a reactive conversationalist to a ReAct logic agent, with the hopes of granting autonomous capabilities to check internal battery levels, manage files, and execute self-contained Python scripts.
