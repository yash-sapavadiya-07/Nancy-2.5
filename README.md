# Nancy Voice Assistant v2.5 - Full Code Working Data

This guide explains how the project works, what every important file does, and which function lives in which file. It focuses on source code and runtime files. Generated files such as `build/`, `dist/`, `__pycache__/`, logs, and temporary audio files are listed separately because they are not the main source code.

## 1. Project Root

Actual app root:

```text
C:\Users\yashs\OneDrive\Documents\Nancy Voice Assistent 2.5\Nancy Voice Assistent 2.2\Nancy Voice Assistent 2.2\Nancy Voice Assistent
```

Main application:

```bash
python main.py
```

API server only:

```bash
python api_server.py
```

Fast/offline mode:

```bash
python nancy_fast.py
```

Install dependencies:

```bash
pip install -r requirements.txt
```

## 2. High-Level Working Flow

```text
User voice or text command
    ->
Input layer
    - Voice: nancy_core/ear.py
    - Web UI: nancy_text_ui.html
    - REST API: api_server.py
    - Tkinter text UI: nancy_text_ui.py
    ->
Brain layer
    - nancy_core/brain.py
    - fast local command matching
    - memory loading/saving
    - LLM routing
    ->
Action layer
    - nancy_core/skills/*.py
    - app control, web search, file work, WhatsApp, system control, etc.
    ->
Output layer
    - voice speech: nancy_core/voice.py
    - web/API text response
    - desktop action result
    ->
Logs and memory
    - nancy_core.log
    - MongoDB collections
```

## 3. Main Source File Map

| File | Main work |
|---|---|
| `main.py` | Main desktop assistant runner. Starts tray icon, voice engine, microphone, brain, clipboard watcher, API server, hotkey, and listen loop. |
| `api_server.py` | FastAPI backend used by browser UI and external clients. Exposes health, chat history, command, and device endpoints. |
| `nancy_text_ui.html` | Main browser command center UI. Sends commands to `http://127.0.0.1:8000/api/text-command`. |
| `nancy_text_ui.py` | Simple Tkinter desktop text chat UI. Uses `Brain` and `VoiceEngine` directly. |
| `nancy_fast.py` | Older/alternate fast mode runner. Forces offline voice and simpler main loop. |
| `nancy_logo_widget.py` | Floating draggable Tkinter logo widget with right-click menu. |
| `nancy_autostart.py` | Windows startup/watchdog manager. Creates VBS launcher and keeps `main.py` running. |
| `config.py` | Older top-level config constants and aliases. The active core config is mostly `nancy_core/config.py`. |
| `requirements.txt` | Python dependencies. |
| `NancyVoiceAssistant.spec` | PyInstaller build config for creating `dist/NancyVoiceAssistant.exe`. |
| `README.md` | Short setup and usage notes. |
| `CLIENT_CODE_WORKTREE.md` | Client-friendly folder explanation. |
| `CLIENT_FLOWCHART.md` | Client-friendly architecture flowchart. |
| `ENHANCEMENTS.md` | Notes about v2.5 enhancements. |

## 4. Core Package Map

| File | Main work |
|---|---|
| `nancy_core/brain.py` | Central decision engine. Registers skills, manages MongoDB memory, chooses fast-track/local/LLM path. |
| `nancy_core/llm_wrapper.py` | Connects to NVIDIA chat completions API and parses `CALL: tool_name(...)` responses. |
| `nancy_core/ear.py` | Microphone selection, calibration, listening, speech recognition through RTX Whisper or Google. |
| `nancy_core/voice.py` | Text-to-speech queue using Edge TTS online and pyttsx3 offline fallback. |
| `nancy_core/config.py` | Active runtime config loaded from `.env`: NVIDIA keys, speech settings, MongoDB URI, wake words. |
| `nancy_core/utils.py` | Shared logger and directory helper. |
| `nancy_core/clipboard_watcher.py` | Background clipboard automation. Detects copied text, opens Notepad/search/apps/calculator. |
| `nancy_core/__init__.py` | Empty package marker. |

## 5. Skills Folder Map

| File | Main work |
|---|---|
| `nancy_core/skills/apps.py` | Open/close/list/refresh desktop applications. |
| `nancy_core/skills/desktop.py` | Brightness, window control, typing, hotkey pressing. |
| `nancy_core/skills/system_ops.py` | Volume, lock, shutdown, restart, sleep, Task Manager, processes, system info, WiFi. |
| `nancy_core/skills/web.py` | Open websites, Google-style search cards, website replication, YouTube, Google Maps. |
| `nancy_core/skills/utilities.py` | Time, date, battery, Wikipedia cards, clipboard, device info, translation. |
| `nancy_core/skills/file_ops.py` | Create/read/write/list/delete files, save clipboard, open Notepad. |
| `nancy_core/skills/editor_ops.py` | Write/edit content in files or active applications. |
| `nancy_core/skills/messaging_ops.py` | WhatsApp, Telegram, and email message flows. |
| `nancy_core/skills/settings.py` | Open Windows Settings pages through `ms-settings:` URIs. |
| `nancy_core/skills/startup_ops.py` | Enable/disable/check Windows autostart. |
| `nancy_core/skills/self_locator.py` | Finds Nancy folder after moves and repairs autostart registry path. |
| `nancy_core/skills/mic_selector.py` | Lists/ranks microphones and selects the best one. |
| `nancy_core/skills/gpu_ops.py` | Detects CUDA/NVIDIA GPU details. |
| `nancy_core/skills/vision.py` | Screenshot capture and screen-analysis marker. |
| `nancy_core/skills/__init__.py` | Empty package marker. |

## 6. Important Runtime Data

| File or folder | Meaning |
|---|---|
| `.env` | Private runtime secrets and settings. Used for NVIDIA API keys, MongoDB URI, RTX mode, microphone index, language. |
| `nancy_core.log` | Main log file from `nancy_core/utils.py`. |
| `nancy_autostart.log` | Startup/watchdog log from `nancy_autostart.py` and batch scripts. |
| `nancy_memory.json` | Older/local memory-style file. Current main memory path is MongoDB. |
| `PyWhatKit_DB.txt` | PyWhatKit generated support file. |
| `speech_*.mp3`, `temp_speech.mp3`, `test_speech_*.mp3` | Generated/test speech audio files. |
| `Nancy virtual assistant.png` | UI/tray/logo image. |
| `website_replicas/` | Output folder created by `web.replicate_website()`. |
| `Screenshots/` | Screenshot/output folder if present. |
| `build/` | PyInstaller build artifacts. Generated, not source. |
| `dist/` | Packaged executable and data output. Generated/delivery files. |
| `__pycache__/` | Python bytecode cache. Generated. |

## 7. Main App: `main.py`

Purpose: start Nancy as a background Windows desktop assistant.

Working sequence:

1. `check_singleton()` binds local port `12345` so only one Nancy instance runs.
2. `verify_and_heal()` checks the project folder and fixes startup registry path if moved.
3. `ensure_autostart_registered()` and `ensure_startup_shortcut()` register Windows autostart.
4. Creates `VoiceEngine`, `Ear`, and `Brain`.
5. Starts clipboard watcher.
6. Registers global hotkey `Ctrl+Alt+N`.
7. Starts `api_server.py` in the background if port `8000` is free.
8. Creates system tray icon with menu items.
9. Enters voice loop:
   - listen through `Ear.listen()`
   - check wake word or active follow-up window
   - strip wake word
   - send command to `process_command_async()`
   - speak reply through `VoiceEngine.speak()`

Functions:

| Function | Work |
|---|---|
| `get_greeting()` | Builds morning/afternoon/evening startup greeting. |
| `on_hotkey_pressed()` | Sets `HOTKEY_TRIGGERED` so the next loop listens immediately. |
| `_port_open(host, port)` | Checks if API port is already bound. |
| `start_api_server()` | Starts `api_server.py` in a hidden background process and waits for port `8000`. |
| `stop_api_server()` | Kills the background API server process. |
| `_build_tray_icon()` | Loads Nancy logo or creates fallback `N` icon. |
| `quit_app(icon, _item)` | Stops API, sets stop event, stops tray icon, exits. |
| `open_dashboard(icon, _item)` | Starts API and opens `nancy_text_ui.html`. |
| `open_web_ai(_icon, _item)` | Opens ChatGPT website. |
| `process_command_async(brain, voice, text)` | Runs `brain.think()` in a thread pool and speaks the reply. |
| `check_singleton()` | Uses a socket lock to prevent duplicate instances. |
| `main()` | Full boot, tray setup, hotkey setup, and voice listen loop. |

## 8. API Server: `api_server.py`

Purpose: expose Nancy through REST API for the web UI or other clients.

Global objects:

```python
brain = Brain()
voice = VoiceEngine()
```

API endpoints:

| Endpoint | Method | Work |
|---|---|---|
| `/api/health` | GET | Returns component status and database status. |
| `/api/chat-history` | GET | Returns in-memory chat history from `Brain`. |
| `/api/clear-chat` | POST | Clears memory in `Brain` and MongoDB. |
| `/api/text-command` | POST | Main command endpoint. Calls `brain.think()`, optionally speaks reply. |
| `/api/device-info` | GET | Returns current device info from `Brain`. |
| `/api/all-devices` | GET | Returns all device records from MongoDB. |

Models:

| Class | Work |
|---|---|
| `CommandRequest` | Validates command text, speak flag, mode, language. |
| `CommandResponse` | Standard success response with reply text. |
| `ErrorResponse` | Standard error shape. |
| `ChatEntry` | One chat history row. |
| `DeviceInfo` | Device metadata model. |

Functions:

| Function | Work |
|---|---|
| `health_check()` | Builds health JSON. |
| `get_chat_history()` | Returns safe chat history entries. |
| `clear_chat()` | Clears memory through `brain.clear_memory()`. |
| `text_command(req)` | Validates text, calls `brain.think(text, mode, language)`, speaks if requested. |
| `get_device_info()` | Returns `brain.get_device_info()`. |
| `get_all_devices()` | Reads `brain.device_collection` and converts `_id` to string. |
| `value_error_handler(request, exc)` | Converts validation errors to JSON. |

## 9. Brain: `nancy_core/brain.py`

Purpose: central intelligence and command router.

Main responsibilities:

- Connect to MongoDB.
- Load/save memory.
- Register all skill functions.
- Try fast local regex patterns first.
- If online LLM is available, call `LLMWrapper`.
- If LLM fails/refuses, run local command handler.
- Save device info to MongoDB.

Important data:

| Field | Meaning |
|---|---|
| `db_available` | Whether MongoDB is reachable. |
| `memory_collection` | MongoDB collection for chat memory. |
| `device_collection` | MongoDB collection for device info. |
| `history` | In-memory chat history, capped at 100 items. |
| `tools_list` | All callable automation functions exposed to the LLM. |
| `llm` | `LLMWrapper` instance. |

Functions/methods:

| Method | Work |
|---|---|
| `__init__()` | Initializes DB, starts background memory load, registers tools, creates LLM wrapper. |
| `chat(user_input, mode, language)` | Backward-compatible wrapper around `think()`. |
| `_init_database()` | Connects to MongoDB and sets collections. |
| `_register_tools()` | Registers apps, system, web, utilities, desktop, file, startup, editor, and messaging tools. |
| `_background_load_memory()` | Loads memory in a background thread. |
| `load_memory(limit)` | Reads latest memory from MongoDB in chronological order. |
| `save_memory(user_text, model_text)` | Adds memory to RAM and MongoDB. |
| `clear_memory()` | Clears RAM history and deletes MongoDB memory entries. |
| `think(user_input, mode, language)` | Main command processor. Uses fast-track, LLM, fallback local commands, then saves memory. |
| `_fast_track(user_input)` | Regex fast path for time, date, app launch, lock, WhatsApp, editor, maps, translation. |
| `_handle_local_command(user_input)` | Local fallback for app/system/web/file/clipboard/volume/window/startup/messaging/settings commands. |
| `get_device_info()` | Builds hostname, OS, CPU, Python, and IP info. |
| `save_device_info()` | Upserts device info into MongoDB. |

Tool categories registered in `_register_tools()`:

```text
apps: open/close/list/refresh apps
system_ops: volume, power, task manager, process, system info, WiFi
web: website, search, YouTube, website replication, maps
utilities: time, date, battery, Wikipedia, clipboard, device info, translation
vision: screenshot
settings: Windows Settings pages
desktop: brightness, windows, keyboard
file_ops: save/read/create/list/paste/delete/open files
startup_ops: autostart
editor_ops: write/edit files and applications
messaging_ops: WhatsApp/Telegram/email
```

## 10. LLM Wrapper: `nancy_core/llm_wrapper.py`

Purpose: talk to NVIDIA's OpenAI-compatible chat completion API and execute tool calls.

Important behavior:

- Uses `config.NVIDIA_API_KEY`.
- Sends a system prompt with available tools.
- Expects tool call text like:

```text
CALL: open_application(app_name=notepad)
```

- Parses the function name and arguments with regex.
- Executes matching Python function from `tools_list`.
- Filters refusal-style replies and sends them back to `Brain` as `__LOCAL__:<command>`.

Functions/methods:

| Function | Work |
|---|---|
| `_is_refusal(text)` | Detects refusal or limitation phrases. |
| `_build_system_prompt(persona, tools_info, language_name, language)` | Builds LLM instruction text with tool list and language rule. |
| `LLMWrapper.__init__(tools_list)` | Stores API headers, model URL, and tools. |
| `route_query(user_input, history, mode, language)` | Calls `query_nvidia()` if NVIDIA key exists. |
| `_get_persona(mode)` | Maps mode `1..5` to response style. |
| `_execute_tool(func_name, args)` | Finds a registered function and calls it. |
| `query_nvidia(user_input, history, mode, language)` | Sends request to NVIDIA, parses reply, handles tool calls, returns final text. |
| `_encode_image(image_path)` | Base64-encodes an image file. Not actively integrated into main vision flow. |

## 11. Ear: `nancy_core/ear.py`

Purpose: voice input and speech recognition.

Working sequence:

1. Create `speech_recognition.Recognizer`.
2. Choose microphone:
   - configured `MIC_DEVICE_INDEX`
   - best microphone from `mic_selector`
   - system default fallback
3. Calibrate ambient noise.
4. If RTX mode and Faster-Whisper are available, load GPU Whisper.
5. On every `listen()`:
   - capture audio
   - transcribe through RTX Whisper if possible
   - fallback to Google speech recognition
   - return lowercase text or `None`

Methods:

| Method | Work |
|---|---|
| `__init__()` | Initializes recognizer, mic, RTX settings. |
| `_init_mic()` | Selects/open microphone. |
| `_try_open_mic(device_index)` | Tests and opens one microphone. |
| `_switch_to_next_best_mic()` | Re-scans mic list after repeated failures. |
| `_init_rtx_power()` | Loads Faster-Whisper GPU model if enabled. |
| `calibrate()` | Re-calibrates ambient noise. |
| `_maybe_recalibrate()` | Starts background calibration every 5 minutes. |
| `listen(timeout, phrase_time_limit)` | Captures and transcribes speech. |
| `_transcribe_rtx(audio)` | Saves temp WAV and transcribes with Faster-Whisper. |
| `_transcribe_google(audio)` | Uses Google Speech Recognition. |
| `get_status()` | Returns microphone/RTX diagnostic state. |

## 12. Voice: `nancy_core/voice.py`

Purpose: text-to-speech output.

Working sequence:

1. Starts a speech queue.
2. Starts a worker thread.
3. Starts an asyncio event loop for Edge TTS.
4. Initializes `pygame.mixer`.
5. Initializes offline `pyttsx3` inside worker thread.
6. `speak()` adds text to queue.
7. Worker plays one message at a time.
8. Online mode uses Edge TTS MP3 chunks.
9. Offline fallback uses pyttsx3.

Methods:

| Method | Work |
|---|---|
| `__init__()` | Creates queue, threads, event loop, pygame mixer. |
| `_run_async_loop()` | Runs asyncio loop forever. |
| `_voice_worker()` | Serially consumes speech queue and plays each item. |
| `setup_offline_voice()` | Picks female/Zira voice and sets speech rate. |
| `speak(text, mode)` | Shortens and queues text. |
| `_shorten_for_speech(text)` | Limits output to about two sentences/300 chars. |
| `process_online_speech(text, mode)` | Generates Edge TTS MP3 chunks and plays via pygame. |
| `process_offline_speech(text)` | Speaks through pyttsx3. |
| `stop()` | Clears queue and stops playback. |

## 13. Config: `nancy_core/config.py`

Purpose: active configuration loaded from `.env`.

Important settings:

| Setting | Work |
|---|---|
| `NVIDIA_API_KEY` | Main LLM API key. |
| `NVIDIA_IMAGE_API_KEY` | Image API key placeholder/support. |
| `NVIDIA_TRANSLATION_API_KEY` | Translation key placeholder/support. |
| `NVIDIA_INVOKE_URL` | NVIDIA chat completions endpoint. |
| `NVIDIA_MODEL` | Current model: `mistralai/mistral-small-4-119b-2603`. |
| `RTX_MODE` | Enables local GPU transcription if possible. |
| `GPU_MODEL_SIZE` | Faster-Whisper model size. |
| `EDGE_TTS_VOICE` | Default online voice. |
| `SPEECH_LANGUAGE` | Speech recognition language. |
| `WAKE_WORDS` | Wake words like `nancy`, `hey nancy`, `assistant`. |
| `ACTIVE_LISTENING_WINDOW` | Follow-up command window in seconds. |
| `MIC_DEVICE_INDEX` | Configured microphone index, `-1` means default/auto. |
| `CLIPBOARD_AUTOMATION_ENABLED` | Enables clipboard watcher. |
| `MONGODB_URI` | MongoDB connection string. |
| `DB_NAME` | Database name: `nancy_ai`. |
| `COLLECTION_NAME` | Memory collection: `user_memory`. |
| `SCREENSHOT_DIR` | Screenshot save directory. |
| `LOG_FILE` | Log file name: `nancy_core.log`. |

Function:

| Function | Work |
|---|---|
| `validate_config()` | Logs warnings for missing critical values. |

## 14. Utils: `nancy_core/utils.py`

Purpose: shared logging and directory helper.

Functions:

| Function | Work |
|---|---|
| `log(message, level)` | Writes to queue-based file and console logger. |
| `ensure_dir(path)` | Creates directory if missing. |

## 15. Clipboard Watcher: `nancy_core/clipboard_watcher.py`

Purpose: background clipboard automation.

Functions:

| Function | Work |
|---|---|
| `_contains_number(text)` | Returns true if clipboard text contains a digit. |
| `clipboard_loop(stop_event)` | Watches clipboard changes. Opens app guesses, Notepad, web search, and Calculator when copied text has numbers. |
| `start_clipboard_watcher(stop_event)` | Starts daemon thread if enabled in config. |

## 16. Skills: `apps.py`

Purpose: launch and close applications.

Functions:

| Function | Work |
|---|---|
| `_load_appopener()` | Lazy imports AppOpener open/close functions. |
| `open_application(app_name)` | Opens apps by aliases, fast system paths, AppOpener, or fallback commands. Also routes settings/system tools. |
| `close_application(app_name)` | Closes via AppOpener or `taskkill`. |
| `list_installed_applications()` | Uses AppOpener cache and returns visible app list. |
| `refresh_application_cache()` | Refreshes AppOpener app cache. |

## 17. Skills: `desktop.py`

Purpose: desktop UI automation.

Functions:

| Function | Work |
|---|---|
| `_get_sbc()` | Lazy loads `screen_brightness_control`. |
| `_get_gw()` | Lazy loads `pygetwindow`. |
| `_get_brightness_powershell()` | Reads current brightness through WMI PowerShell. |
| `_set_brightness_powershell(level)` | Sets brightness through WMI PowerShell. |
| `set_brightness(level)` | Sets brightness using library or PowerShell fallback. |
| `brightness_up(amount)` | Increases brightness. |
| `brightness_down(amount)` | Decreases brightness. |
| `minimize_current_window()` | Minimizes active window. |
| `maximize_current_window()` | Maximizes active window. |
| `close_current_window()` | Closes active window. |
| `switch_window(window_title)` | Activates first matching window title. |
| `type_text(text)` | Types text into active UI using pyautogui. |
| `press_keys(keys)` | Presses hotkey combinations like `ctrl s`. |

## 18. Skills: `system_ops.py`

Purpose: Windows system actions.

Functions:

| Function | Work |
|---|---|
| `_run_vbs(script_content)` | Runs temporary VBScript. Currently not heavily used. |
| `set_volume(level)` | Approximates absolute volume by pressing volume down/up keys. |
| `volume_up(amount)` | Presses volume-up key multiple times. |
| `volume_down(amount)` | Presses volume-down key multiple times. |
| `mute_volume()` | Toggles mute. |
| `lock_screen()` | Calls Windows `LockWorkStation`. |
| `shutdown_system(seconds)` | Schedules shutdown. |
| `cancel_shutdown()` | Cancels pending shutdown. |
| `restart_system(seconds)` | Schedules restart. |
| `sleep_system()` | Calls Windows suspend state command. |
| `open_task_manager()` | Opens Task Manager. |
| `open_control_panel()` | Opens Control Panel. |
| `kill_process(process_name)` | Kills a named process using `taskkill`. |
| `list_running_processes()` | Lists top processes by memory using psutil. |
| `empty_recycle_bin()` | Empties Windows Recycle Bin. |
| `open_file_explorer(path)` | Opens File Explorer, optionally at a known folder/path. |
| `open_run_dialog()` | Sends `Win+R`. |
| `get_system_info()` | Returns OS, CPU, RAM, disk, GPU, battery, network, uptime, Python version. |
| `toggle_wifi()` | Enables/disables WiFi interface through `netsh`. |

## 19. Skills: `web.py`

Purpose: web actions and search rendering.

Functions:

| Function | Work |
|---|---|
| `open_website(url)` | Opens URL in default browser. |
| `replicate_website(url)` | Downloads HTML/CSS/JS/images into `website_replicas/`. |
| `google_search(query)` | Returns `RENDER_CARDS:` JSON using Google News RSS, Mojeek, Wikipedia, or fallback card. |
| `play_youtube(video_topic)` | Plays topic through PyWhatKit YouTube helper. |
| `open_google_maps(address)` | Opens Google Maps search for address. |

## 20. Skills: `utilities.py`

Purpose: small utility actions.

Functions:

| Function | Work |
|---|---|
| `get_current_time()` | Returns current local time. |
| `get_current_date()` | Returns current local date. |
| `get_battery_status()` | Returns battery percentage and charging state. |
| `search_wikipedia(topic)` | Returns Wikipedia result as `RENDER_CARDS:` JSON. |
| `read_clipboard()` | Reads text clipboard. |
| `write_to_clipboard(text)` | Writes text to clipboard. |
| `get_device_name()` | Returns hostname. |
| `get_device_info()` | Returns platform, OS, Python, IP, CPU metadata. |
| `translate_text(text, target_language)` | Uses unofficial Google Translate endpoint and language-name mapping. |

## 21. Skills: `file_ops.py`

Purpose: file and Notepad automation.

Functions:

| Function | Work |
|---|---|
| `_resolve_path(filepath)` | Resolves bare filenames to Desktop path. |
| `save_text_to_file(text, filename)` | Writes text to a file. |
| `read_file(filepath)` | Reads up to 10 KB from a file. |
| `create_file(filename)` | Creates empty file. |
| `list_files(directory)` | Lists files/folders in Desktop/Documents/Downloads/full path. |
| `paste_to_notepad(text)` | Opens Notepad with text or clipboard content via temp file. |
| `save_clipboard_to_file(filename)` | Saves clipboard text to Desktop/file path. |
| `append_to_file(text, filename)` | Appends text to a file. |
| `delete_file(filename)` | Deletes a file. |
| `notepad_create_save(text, filename)` | Creates/opens/saves Notepad file. |
| `open_in_notepad(filepath)` | Opens existing file or blank Notepad. |

## 22. Skills: `editor_ops.py`

Purpose: write/edit text in active apps and files.

Functions:

| Function | Work |
|---|---|
| `_resolve(filepath)` | Resolves aliases and bare filenames to real paths. |
| `_bring_window_to_front(window_title_fragment)` | Focuses window by partial title. |
| `_type_text_safely(text, interval)` | Clipboard-pastes text into active window, preserving original clipboard. |
| `write_in_active_app(text)` | Types into current active window. |
| `write_in_app(app_name, text)` | Focuses/opens target app and types text. |
| `open_file_and_write(filename, text, mode)` | Appends/overwrites a file and opens in Notepad. |
| `find_and_replace_in_file(filename, find_text, replace_text)` | Replaces exact text in file. |
| `clear_and_write_file(filename, text)` | Overwrites whole file. |
| `insert_line_in_file(filename, text, line_number)` | Inserts a line at a 1-based line number. |
| `append_line_to_file(filename, text)` | Appends one line to file. |
| `get_file_contents(filename, max_chars)` | Reads file content for display/speech. |
| `select_all_and_replace_in_app(app_name, new_text)` | Focuses app, selects all, deletes, pastes new text. |

## 23. Skills: `messaging_ops.py`

Purpose: messaging actions, mainly WhatsApp Desktop/Web.

Functions:

| Function | Work |
|---|---|
| `_clean_phone(raw)` | Normalizes phone number. |
| `_is_phone_number(s)` | Detects phone-like strings. |
| `_get_whatsapp_window()` | Finds WhatsApp/WhatsApp Beta window. |
| `_is_whatsapp_open()` | Checks if WhatsApp window exists. |
| `_bring_whatsapp_to_front()` | Restores and focuses WhatsApp. |
| `_wait_for_window(timeout)` | Waits for WhatsApp window. |
| `_find_whatsapp_exe()` | Searches common WhatsApp install paths. |
| `_get_whatsapp_aumid()` | Finds Microsoft Store app AUMID from registry. |
| `_launch_whatsapp()` | Launches WhatsApp through exe, AppsFolder, or URI scheme. |
| `_paste(text)` | Pastes text through clipboard. |
| `_open_chat_desktop(contact)` | Opens/searches chat in WhatsApp Desktop. |
| `_send_via_pywhatkit(phone, message)` | Sends WhatsApp Web message through PyWhatKit. |
| `_open_wame(phone, message)` | Opens `wa.me` URL with optional message. |
| `send_whatsapp_message(contact, message)` | Sends WhatsApp message using desktop first, then web fallbacks. |
| `call_on_whatsapp(contact)` | Opens chat/call flow for WhatsApp. |
| `open_whatsapp_chat(contact)` | Opens a chat without sending. |
| `send_message_to_app(app_name, contact, message)` | Routes to WhatsApp, Telegram, or email. |

## 24. Skills: `settings.py`

Purpose: open Windows Settings pages.

Function:

| Function | Work |
|---|---|
| `open_settings(setting_name)` | Maps friendly names like `wifi`, `display`, `battery`, `privacy` to `ms-settings:` URI and opens it. |

## 25. Skills: `startup_ops.py`

Purpose: manage Windows startup from voice/tool calls.

Functions:

| Function | Work |
|---|---|
| `get_startup_folder()` | Returns Windows startup folder path. |
| `get_shortcut_path()` | Returns expected Nancy shortcut path. |
| `enable_registry_startup()` | Writes `NancyVoiceAssistant` Run key. |
| `disable_registry_startup()` | Removes Run key. |
| `enable_autostart(silent)` | Creates startup shortcut and registry entry. |
| `disable_autostart()` | Removes shortcut and registry entry. |
| `get_startup_status()` | Reports enabled/stale/disabled startup state. |

## 26. Skills: `self_locator.py`

Purpose: keep autostart correct if the project folder moves.

Functions:

| Function | Work |
|---|---|
| `_get_all_drives()` | Lists mounted Windows drives. |
| `_is_nancy_root(path)` | Checks for `nancy_core/` and root files. |
| `_search_drive(drive)` | Walks one drive searching for Nancy root. |
| `find_nancy_root_on_machine()` | Searches all drives in parallel. |
| `_get_registry_nancy_path()` | Reads current registry Run value. |
| `_set_registry_nancy_path(command)` | Writes registry Run value. |
| `get_current_root()` | Returns current project root from `__file__`. |
| `verify_and_heal(log_fn)` | Verifies root and updates registry if needed. |
| `ensure_autostart_registered(log_fn)` | Idempotently writes registry startup command. |
| `ensure_startup_shortcut(log_fn)` | Creates/updates Startup folder shortcut. |

## 27. Skills: `mic_selector.py`

Purpose: choose best microphone.

Functions:

| Function | Work |
|---|---|
| `_score_name(name)` | Scores mic name by preferred/blacklisted keywords. |
| `list_all_mics(log_fn)` | Lists visible SpeechRecognition microphones with scores. |
| `_measure_snr(device_index, duration)` | Measures ambient energy/quality for one mic. |
| `_snr_test_candidates(candidates, log_fn, timeout)` | Tests top mic candidates in parallel. |
| `get_best_mic_index(log_fn, run_snr_test)` | Returns best mic index or `-1`. |
| `refresh_mic_in_config(log_fn)` | Writes selected mic index back into `.env`. |

## 28. Skills: `gpu_ops.py`

Purpose: GPU detection for RTX mode.

Functions:

| Function | Work |
|---|---|
| `is_nvidia_gpu_available()` | Checks Torch CUDA first, then `nvidia-smi`. |
| `get_gpu_info()` | Returns availability, GPU name, VRAM, CUDA version. |

## 29. Skills: `vision.py`

Purpose: screenshots.

Functions:

| Function | Work |
|---|---|
| `take_screenshot()` | Saves screenshot to Pictures folder and returns path. |
| `analyze_screen(prompt)` | Takes screenshot and returns `SCREENSHOT_TAKEN:<path>` marker. |

## 30. Web UI: `nancy_text_ui.html`

Purpose: full browser command center.

Important frontend pieces:

| JS function/data | Work |
|---|---|
| `CONFIG` | API URL, health interval, UI limits. |
| `state` | Current UI state: mode, counts, command history, listening state. |
| `PALETTE_COMMANDS` | Quick command palette items. |
| `escapeHtml(text)` | Prevents unsafe HTML injection in messages. |
| `timeStr()` | Current timestamp string. |
| `relativeTime(ms)` | Formats uptime. |
| `getGreeting()` | UI greeting text. |
| `formatTextWithLinks(text)` | Converts URLs/text into display HTML. |
| `shortenUrl(url)` | Short display URL. |
| `addOutputLine(text, type)` | Adds user/Nancy/system line to terminal. |
| `renderSearchCards(data)` | Renders `RENDER_CARDS:` search results. |
| `showTyping()` / `hideTyping()` | Shows/hides typing indicator. |
| `sendCommand(text)` | POSTs command to `/api/text-command`. |
| `updateAvgSpeed()` | Tracks average response time. |
| `checkHealth()` | Calls `/api/health` and updates status widgets. |
| `updateConnectionStatus(connected)` | Shows online/offline UI state. |
| `performClear()` | Clears terminal UI counters. |
| `openPalette()` / `closePalette()` | Shows/hides command palette. |
| `renderPaletteResults(query)` | Filters command palette. |
| `bindChips(container)` | Binds quick chips. |
| `loadHistory()` | Loads `/api/chat-history` but is disabled in initialization. |
| `toggleListening()` / `stopListening()` | Browser Web Speech API microphone input. |

## 31. Tkinter Text UI: `nancy_text_ui.py`

Purpose: simple Python desktop chat interface.

Class:

| Class/method | Work |
|---|---|
| `NancyTextUI.__init__()` | Creates `VoiceEngine`, `Brain`, Tkinter window, output box, input box. |
| `append_system(text)` | Adds system line. |
| `append_user(text)` | Adds user line. |
| `append_nancy(text)` | Adds Nancy line. |
| `_append(text)` | Writes to disabled scrolled text area. |
| `on_send(event)` | Reads entry and starts background command thread. |
| `_process_command(user_text)` | Calls `brain.think()`, updates UI, speaks reply. |
| `run()` | Starts Tkinter mainloop. |

## 32. Logo Widget: `nancy_logo_widget.py`

Purpose: floating desktop logo window.

Class:

| Class/method | Work |
|---|---|
| `NancyLogoWidget.__init__(logo_path, size)` | Creates transparent borderless topmost logo window. |
| `_exit_app()` | Closes widget window. |
| `_find_logo()` | Finds Nancy logo image. |
| `show_context_menu(event)` | Shows right-click menu. |
| `open_web_ui()` | Opens local `nancy_text_ui.html`. |
| `open_web_ai()` | Opens ChatGPT. |
| `start_move(event)` | Stores drag start point. |
| `do_move(event)` | Moves widget while dragging. |
| `run()` | Starts Tkinter mainloop. |
| `launch_logo_widget()` | Starts widget in background thread. |

## 33. Autostart: `nancy_autostart.py`

Purpose: configure startup and watchdog.

Functions:

| Function | Work |
|---|---|
| `get_root()` | Returns script directory. |
| `log(message)` | Prints and appends to `nancy_autostart.log`. |
| `is_running(script_name)` | Checks running Python command lines for script name. |
| `start_process(script_name)` | Starts script with `pythonw.exe` hidden. |
| `setup_autostart()` | Creates VBS launcher, registry Run key, startup shortcut. |
| `create_vbs_launcher()` | Writes `nancy_boot.vbs`. |
| `run_watchdog()` | Starts `main.py` and restarts it every minute if stopped. |

## 34. Batch/VBS Launchers

| File | Work |
|---|---|
| `run_nancy.bat` | Runs `python nancy_fast.py`. |
| `run_forever.bat` | Watchdog batch file. Kills existing Python processes, starts `main.py` and `api_server.py`, monitors both. |
| `run_jarvis.bat` | Attempts to run `jarvis.py`, but `jarvis.py` is not present in this project tree. |
| `set_startup.bat` | Creates Startup folder shortcut to `nancy_boot.vbs`. |
| `nancy_boot.vbs` | Hidden Windows launcher for `nancy_autostart.py --run`. Current hardcoded path appears stale and should be regenerated by `nancy_autostart.py` or self-locator. |

## 35. Test and Debug Files

| File | Work |
|---|---|
| `debug_mic.py` | Lists microphones and tests default mic speech recognition. |
| `test_voice_fix.py` | Tests configured/default mic, calibration, Google recognition, wake word detection. |
| `test_voice_queue.py` | Queues several voice messages to test serialized playback. |
| `test_search.py` | Tests `web.google_search()` and `utilities.search_wikipedia()` card markers. |
| `verify_new_arch.py` | Smoke test for voice, utility/system skills, and Brain response. |
| `check_models.py` | Lists NVIDIA models through API key. |
| `list_models.py` | Same NVIDIA model-list helper. |
| `nancy_core/list_models.py` | Older OpenAI model-list helper; references `config.OPENAI_API_KEY`, which is not defined in current `nancy_core/config.py`. |
| `index_apps.py` | Forces AppOpener to index applications. |
| `test_base64.py` | Tests image-to-base64 HTML display, but references a missing/old generated image path. |
| `test_direct_api.py` | Direct NVIDIA image API experiment. Contains a hardcoded API token and should not be shared. |

## 36. Scratch Files

| File/folder | Work |
|---|---|
| `scratch/debug_search.py` | Search debugging helper. |
| `scratch/debug_search_html.py` | HTML search debugging helper. |
| `scratch/debug_search_mojeek.py` | Mojeek search debugging helper. |
| `scratch/debug_news_rss.py` | Google News RSS debugging helper. |
| `scratch/debug_google_direct.py` | Direct Google scraping/debug helper. |
| `scratch/bing.html`, `scratch/mojeek.html` | Saved/debug HTML samples. |

## 37. Data Flow by User Command Type

Voice command:

```text
main.py
  -> Ear.listen()
  -> Brain.think()
  -> Brain._fast_track() or LLMWrapper.route_query() or Brain._handle_local_command()
  -> skill function if action needed
  -> Brain.save_memory()
  -> VoiceEngine.speak()
```

Browser UI command:

```text
nancy_text_ui.html sendCommand()
  -> POST /api/text-command
  -> api_server.text_command()
  -> Brain.think()
  -> skill/LLM/local fallback
  -> VoiceEngine.speak() if speak=true
  -> JSON reply
  -> UI addOutputLine()
```

Local Tkinter text command:

```text
nancy_text_ui.py NancyTextUI.on_send()
  -> background thread
  -> Brain.think()
  -> UI append_nancy()
  -> VoiceEngine.speak()
```

Fast-track action example, "open notepad":

```text
Brain.think()
  -> _fast_track()
  -> _handle_local_command()
  -> apps.open_application("notepad")
  -> subprocess.Popen(["notepad.exe"])
  -> response: "Opening notepad."
```

LLM tool action example, "search latest AI news":

```text
Brain.think()
  -> LLMWrapper.query_nvidia()
  -> LLM returns CALL: google_search(query=latest AI news)
  -> LLMWrapper._execute_tool()
  -> web.google_search()
  -> returns RENDER_CARDS JSON string
  -> API/UI displays cards
```

## 38. MongoDB Data

Configured in `nancy_core/config.py`:

```text
Database: nancy_ai
Memory collection: user_memory
Device collection: user_devices
```

Memory entry shape:

```json
{
  "timestamp": "date-time string",
  "user": "user command",
  "nancy": "assistant reply"
}
```

Device entry shape includes:

```json
{
  "hostname": "...",
  "platform": "...",
  "system": "...",
  "release": "...",
  "version": "...",
  "machine": "...",
  "processor": "...",
  "python_version": "...",
  "ip_address": "...",
  "last_seen": "..."
}
```

If MongoDB is not running, Nancy still works with local commands and LLM, but chat history/device APIs may return empty data or service-unavailable errors.

## 39. Security and Maintenance Notes

Important:

- `.env` should hold private API keys. Do not commit/share it.
- `test_direct_api.py` contains a hardcoded NVIDIA-style API token. Move it to `.env` and rotate/revoke that token if it was real.
- `nancy_boot.vbs` contains a hardcoded path pointing to an older Downloads location. Run `python nancy_autostart.py --setup` or let `main.py` self-locator/autostart code regenerate it.
- `run_jarvis.bat` points to `jarvis.py`, but that file is not present.
- `nancy_core/list_models.py` references `OPENAI_API_KEY`, but the current core config has NVIDIA keys, not `OPENAI_API_KEY`.
- `system_ops.py` has duplicate import blocks. It works, but can be cleaned.
- Some existing Markdown/docs show mojibake characters due to encoding display issues. The Python source still runs, but documentation text can be normalized later.

## 40. Files Usually Safe to Ignore During Code Review

```text
build/
dist/
__pycache__/
.idea/
.vscode/
*.log
speech_*.mp3
test_speech_*.mp3
temp_speech.mp3
scratch/*.html
```

## 41. Most Important Files to Understand First

Read in this order:

1. `main.py`
2. `nancy_core/brain.py`
3. `nancy_core/llm_wrapper.py`
4. `nancy_core/ear.py`
5. `nancy_core/voice.py`
6. `api_server.py`
7. `nancy_text_ui.html`
8. `nancy_core/skills/apps.py`
9. `nancy_core/skills/system_ops.py`
10. `nancy_core/skills/web.py`
11. `nancy_core/skills/messaging_ops.py`

This order explains the whole product from startup to input, decision-making, actions, UI, and output.
