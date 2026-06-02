Problem B — Real-time voice assistant using open models
Live Demo (Web App): https://4f5dba4203781.notebooksn.jarvislabs.net/

What it does
This project is an ultra-low latency, voice-to-voice IT Helpdesk assistant that allows employees to resolve common technical issues (like VPN access, Wi-Fi configuration, or password resets) through natural, spoken conversation. It eliminates the friction of typing out support tickets and provides instant, conversational troubleshooting directly through the browser.

Why I built this
I chose this problem because internal IT support is often bottlenecked by repetitive, low-tier queries, leading to frustration for both employees and support staff. While text-based chatbots exist, voice interactions offer a much more natural and accessible experience, especially when an employee is away from their keyboard or dealing with screen lockouts. Furthermore, building a real-time voice pipeline presented a fantastic engineering challenge: minimizing end-to-end latency while successfully managing concurrent ML models (ASR, LLM, TTS) in VRAM on a single GPU.

How to run it
Clone the repository and install the required dependencies (ensure you have PyTorch with CUDA support).

Bash
pip install fastapi uvicorn faster-whisper transformers accelerate numpy kokoro
Open main.py and replace hf_your_actual_token_here with your valid Hugging Face Access Token.

Start the application using Uvicorn (configured for port 6006 to allow JarvisLabs web dashboard port forwarding):

Bash
uvicorn main:app --host 0.0.0.0 --port 6006
Navigate to http://localhost:6006 in your web browser, grant microphone permissions, and click "Start Conversation".

Architecture decisions
Pipelined Chunking vs. Turn-based Processing: Waiting for the LLM to generate a full response before triggering Text-to-Speech (TTS) creates an unnatural delay. I decided to implement a threaded generation approach with sentence-boundary chunking. The TTS begins synthesizing the first sentence while the LLM is still reasoning the second. This slashed the Time to First Audio (latency) down to ~850ms.

Gemma-2B-it vs. 7B Class Models: While 7B models offer robust reasoning, loading an ASR (Whisper), an LLM, and a TTS (Kokoro) simultaneously into VRAM caused heavy memory bottlenecks. I opted for google/gemma-2b-it loaded in float16. This specific trade-off prioritized ultra-fast token generation and stable concurrent model execution over raw parameter size, which is vital for a real-time conversational agent.

WebSocket PCM Streaming vs. HTTP Polling: I needed to process audio chunks in real-time. I bypassed standard HTTP endpoints and implemented WebSockets to stream raw 16-bit PCM audio arrays bidirectionally. This avoids HTTP overhead entirely and allows the frontend Web Audio API to play audio buffers instantly as they arrive.

What I used AI for
Generated: I utilized AI (ChatGPT/Claude) to scaffold the boilerplate FastAPI WebSocket logic and the frontend HTML/JS Web Audio API script. Handling raw PCM audio buffers and AudioContext nodes in the browser can be highly syntax-heavy, and AI sped up this initial scaffolding.

Written by hand: The core orchestration logic—specifically the thread management for the LLM's TextIteratorStreamer and the buffer logic to parse sentence boundaries for the Kokoro TTS pipeline—was written and heavily tuned by hand to hit my latency targets.

Overridden: Early AI suggestions recommended saving the generated audio to temporary .wav files on the disk and serving them via HTTP endpoints. I entirely overrode this architectural suggestion, rewriting the backend to convert float32 numpy arrays into 16-bit PCM and stream them directly from memory to the WebSocket to avoid disk I/O latency.

What I would change with 4 more weeks
If I were to ship this to real users in production, I would prioritize the following:

Voice Activity Detection (VAD) & Interruption: Currently, the system relies on continuous audio buffering. I would integrate a fast VAD model (like Silero VAD) to detect exactly when the user starts and stops speaking. I would also add logic to halt the TTS playback and LLM generation if the user interrupts (barge-in).

RAG Integration: I would connect the LLM to a vector database containing actual company IT documentation. This would ground the agent's responses in factual, company-specific troubleshooting steps rather than relying solely on the base model's internal weights.

Session Management & Security: The current WebSocket handles a single global stream. I would implement user authentication and distinct WebSocket session IDs to isolate audio streams, allowing multiple employees to use the helpdesk concurrently.
