<p align="center">
    <img src="https://github.com/Dadangdut33/Speech-Translate/blob/master/speech_translate/assets/icon.png?raw=true" width="250px" alt="Speech Translate Logo">
</p>

<h1 align="center">Speech Translate</h1>

<p align="center">
    <a href="https://github.com/Dadangdut33/Speech-Translate/issues"><img alt="GitHub issues" src="https://img.shields.io/github/issues/Dadangdut33/Speech-Translate"></a>
    <a href="https://github.com/Dadangdut33/Speech-Translate/pulls"><img alt="GitHub pull requests" src="https://img.shields.io/github/issues-pr/Dadangdut33/Speech-Translate"></a>
    <a href="https://github.com/Dadangdut33/Speech-Translate/releases/latest"><img alt="github downloads"  src="https://img.shields.io/github/downloads/Dadangdut33/Speech-Translate/total?label=downloads (github)"></a> 
    <a href="https://github.com/Dadangdut33/Speech-Translate/releases/latest"><img alt="GitHub release (latest SemVer)" src="https://img.shields.io/github/v/release/Dadangdut33/Speech-Translate"></a>
    <a href="https://github.com/Dadangdut33/Speech-Translate/commits/master"><img alt="GitHub commits since latest release (by date)" src="https://img.shields.io/github/commits-since/Dadangdut33/Speech-Translate/latest"></a>
    <a href="https://github.com/Dadangdut33/Speech-Translate/compare/master...dev"><img alt="GitHub commits difference between master and dev branch" src="https://img.shields.io/github/commits-difference/dadangdut33/speech-translate?base=master&head=dev&label=commits%20difference%20with%20%40dev%20branch"></a><Br>
    <a href="https://github.com/Dadangdut33/Speech-Translate/stargazers"><img alt="GitHub Repo stars" src="https://img.shields.io/github/stars/Dadangdut33/Speech-Translate?style=social"></a>
    <a href="https://github.com/Dadangdut33/Speech-Translate/network/members"><img alt="GitHub forks" src="https://img.shields.io/github/forks/Dadangdut33/Speech-Translate?style=social"></a>
</p>

Speech Translate is a practical application that combines OpenAI's Whisper ASR model with free translation APIs. It serves as a versatile tool for both real-time / live speech-to-text and speech translation, allowing the user to seamlessly convert spoken language into written text. Additionally, it has the option to import and transcribe audio / video files effortlessly.

Speech Translate aims to expand whisper ability by combining it with some translation APIs while also providing a simple and easy to use interface to create a more practical application. This application is also open source, so you can contribute to this project if you want to.

---

## 📌 About this fork

This fork runs the **translation step on a local LibreTranslate container** instead of the public free APIs, and carries a few robustness fixes for long live-recording sessions.

**Why.** Live transcription produces a very high request rate — around 170 segments per minute in practice. No free public translation service sustains that: Google starts serving a CAPTCHA page (`Our systems have detected unusual traffic`) and MyMemory runs out of quota, both surfacing as a misleading `TooManyRequests: ... according to google, 5 requests per second` error. Running the engine locally removes the quota entirely.

Transcription was never affected — Whisper always runs locally on the GPU.

### Setup

Start the translation server once:

```bash
docker compose up -d --build
```

`docker-compose.yml` builds a thin layer over the official image. What it passes:

- `--load-only en,pt` keeps only the language pair in use (adjust to your languages)
- `--threads 2` avoids the default 4 gunicorn workers competing to load the model, which made them hit `WORKER TIMEOUT` and restart in a loop, so the first request never completed
- `--translation-cache all` matters in live mode, where the same partial sentence is re-translated as it grows
- `LT_TIMEOUT=60` replaces the stock 2400s, so a wedged worker costs a minute instead of forty
- the named volume keeps the ~159MB of language models across container recreations

**Why a derived image instead of plain `docker run`.** The official entrypoint starts gunicorn with `--max-requests 250`, recycling each worker once it has served 250 requests. That worker then never exits: it blocks on a futex, holding a lock the native translation libraries left behind in the child at fork time. The container still reads `Up` and the port still accepts connections, but nothing answers, and the arbiter only replaces the process once `--timeout` expires — 2400s by default. Live transcription burns through 250 requests in minutes, so the translation pane would go dead, come back for about ninety seconds while the fresh workers drained the backlog, and go dead again. The Dockerfile drops `--max-requests`; `LT_TIMEOUT` is the safety net in case anything else ever wedges a worker.

Then point the app at it — in **Settings → Translate**:

| Field | Value |
| :-- | :-- |
| Engine | `LibreTranslate` |
| LibreTranslate link | `http://localhost:5000` |
| API key | leave empty (a local instance needs none) |

The *"LibreTranslate API key is not set"* warning is expected with a local server; tick **Supress Empty API Key** in the same tab to silence it.

### Running

```bash
docker compose up -d          # ~15s until it answers
run-app.bat                   # or: .venv\Scripts\python.exe Run.py
```

Check it with `docker compose ps`: the healthcheck calls `/languages`, which only answers when a worker is actually alive, so `healthy` tells you more here than `Up` does.

If the engine goes down mid-session the app no longer fails silently. After two failures in a row it says so on the recording modal (`⚠️ LibreTranslate not answering (N skipped)`), stops spending a request timeout on every sentence, and resumes on its own once the engine answers again.

### Note on Whisper as the translation engine

Picking a Whisper model in the *Translate* dropdown runs translation locally with no API at all — but Whisper only translates **into English** (`WHISPER_TARGET = ["English"]`). For any other target language an external engine is required, which is what this setup provides locally.

### Changes in this fork

- Translation requests are rate limited and back off exponentially on rejection, with finished sentences requeued instead of dropped; local engines are exempt from the throttle
- An engine that stops answering is detected after two consecutive failures and reported on the recording modal, rather than leaving the translation pane to go quiet with no explanation. While it is down the worker probes once every few seconds instead of spending a request timeout per sentence, and recovers by itself
- The recording thread no longer waits on the translation queue during an outage. That wait is capped at 3s per buffer break, and in a 92 minute session against a dead engine it added up to 39 minutes of stalled recording — it degraded the transcript, not just the translation
- Network failures from LibreTranslate log one line instead of a full stack trace. One outage wrote 1,716 tracebacks and 8.7MB of log, which buries anything real
- Stopping a recording drains the pending translations instead of discarding them, bounded so a dead engine cannot hold the stop button hostage
- A live partial that finishes translating after its sentence was already closed is dropped instead of overwriting the current text
- `bc.tl_sentences` is trimmed to the session's sentence cap, like the transcript pane already was
- The saved audio device is looked up by name when its index no longer matches, so pairing a Bluetooth headset does not silently shift the recording to a different device
- A stream that breaks mid frame (a Bluetooth endpoint being reconfigured, for instance) no longer aborts the recording session with a reshape error
- The Silero/auto-threshold disable handlers no longer raise `TclError` over a destroyed widget, which used to mask the original error

---

<p align="center">
  <img src="preview/1.png" width="700" alt="Speech Translate Preview">
</p>

<details close>
  <summary>Preview - Usage</summary>
  <p align="center">
    <img src="preview/7.png" width="700" alt="Record">
    <img src="preview/8.png" width="700" alt="File import">
    <img src="preview/9.png" width="700" alt="File import in progress">
    <img src="preview/10.png" width="700" alt="Align result">
    <img src="preview/11.png" width="700" alt="Refine result">
    <img src="preview/12.png" width="700" alt="Translate Result">
    <img src="preview/13.png" width="700" alt="Transcribe mode on subtitle window (English)"><br />
    Transcribe mode on detached window (English)    
    <img src="preview/14.png" width="700" alt="Translate mode on subtitle window (English to Indonesia)"><br />
    Translate mode on detached window (English to Indonesia)
  </p>
</details>

<details close>
  <summary>Preview - Setting</summary>
  <p align="center">
    <img src="preview/2.png" width="700" alt="Setting - General">
    <img src="preview/3.png" width="700" alt="Setting - Record">
    <img src="preview/4.png" width="700" alt="Setting - Whisper">
    <img src="preview/4-5.png" width="700" alt="Setting - File Export">
    <img src="preview/5.png" width="700" alt="Setting - Translate">
    <img src="preview/6.png" width="700" alt="Setting - Textbox">
  </p>
</details>

<br />

<h1>Table Of Contents</h1>

- [🚀 Features](#-features)
- [📜 Requirements](#-requirements)
- [🔧 Installation](#-installation)
  - [From Prebuilt Binary (.exe)](#from-prebuilt-binary-exe)
  - [As A Module](#as-a-module)
  - [From Git](#from-git)
- [📚 More Information](#-more-information)
- [🛠️ Development](#️-development)
  - [Setup](#setup)
  - [Running the app](#running-the-app)
  - [Building](#building)
  - [Compatibility](#compatibility)
- [💡 Contributing](#-contributing)
- [License](#license)
- [Attribution](#attribution)
- [Other](#other)

# 🚀 Features

- Speech to text and/or Speech translation (transcribed text can be translated to other languages) with live input from mic or speaker 🎙️
- Customizable [subtitle window](https://github.com/Dadangdut33/Speech-Translate/raw/master/preview/13.png) for live speech to text and/or speech translation
- Batch file processing of audio / video files for transcription and translation with output of (.txt .srt .ass .tsv .vtt .json) 📂
- Result [refinement](https://github.com/jianfch/stable-ts#refinement)
- Result [alignment](https://github.com/jianfch/stable-ts#alignment)
- Result translation (Translate only the result.json)

# 📜 Requirements

- Compatible OS Installation:

|   OS    | Installation from Prebuilt binary | Installation as a Module | Installation from Git |
| :-----: | :-------------------------------: | :----------------------: | :-------------------: |
| Windows |                ✔️                 |            ✔️            |          ✔️           |
|  MacOS  |                ❌                 |            ✔️            |          ✔️           |
|  Linux  |                ❌                 |            ✔️            |          ✔️           |

\* Python 3.8 or later (3.11 is recommended) for installation as module.

- **Speaker input** only work on _windows 8 and above_ (Alternatively, you can make a loopback to capture your system audio as virtual input (like mic input) by using this guide/tool: [[Voicemeeter on Windows]](https://voicemeeter.com/)/[[YT Tutorial]](https://youtu.be/m6rp9lkiFBU) - [[pavucontrol on Ubuntu with PulseAudio]](https://wiki.ubuntu.com/record_system_sound) - [[blackhole on MacOS]](https://github.com/ExistentialAudio/BlackHole))
- Internet connection is needed **only for translation with API & downloading models** (If you want to go fully offline, you can setup [LibreTranslate](https://github.com/LibreTranslate/LibreTranslate) on your local machine and set it up in the [app settings](https://github.com/Dadangdut33/Speech-Translate/wiki/Options#libre-translate-setting))
- **Recommended** to have `Segoe UI` font installed on your system for best UI experience (For OS other than windows, you can see this: [Ubuntu](https://github.com/mrbvrz/segoe-ui-linux) - [MacOS](https://github.com/tejasraman/segoe-ui-macos))
- **Recommended** to have capable [GPU with CUDA compatibility](https://developer.nvidia.com/cuda-gpus) (prebuilt version is using CUDA 11.8) for faster result. Each whisper model has different requirements, for more information you can check it directly at the [whisper repository](https://github.com/openai/whisper).

|  Size  | Parameters | Required VRAM | Relative speed |
| :----: | :--------: | :-----------: | :------------: |
|  tiny  |    39 M    |     ~1 GB     |      ~32x      |
|  base  |    74 M    |     ~1 GB     |      ~16x      |
| small  |   244 M    |     ~2 GB     |      ~6x       |
| medium |   769 M    |     ~5 GB     |      ~2x       |
| large  |   1550 M   |    ~10 GB     |       1x       |

\* This information is also available in the app (hover over the model selection in the app and there will be a tooltip about the model info). Also note that when using faster-whisper, the model speed will be significantly faster and have smaller vram usage, for more information about this please visit [faster-whisper repository](https://github.com/guillaumekln/faster-whisper)

# 🔧 Installation

> [!IMPORTANT]  
> Please take a look at the [Requirements](#requirements) first before installing. For more information about the usage of the app, please check the [wiki](https://github.com/Dadangdut33/Speech-Translate/wiki)

## From Prebuilt Binary (.exe)

> [!NOTE]  
> The prebuilt binary is shipped with CUDA 11.8, so it will only work with GPU that has CUDA 11.8 compatibility. If your GPU is not compatible, you can try [installation as module](#as-a-module) or [from git](#From-Git) below.

1. Download the [latest release](https://github.com/Dadangdut33/Speech-Translate/releases/latest) (There are 2 versions, CPU and GPU/CUDA)
2. Install/extract the downloaded file
3. Run the program
4. Set the settings to your liking
5. Enjoy!

## As A Module

> [!NOTE]  
> Use python 3.11 for best compatibility and performance

> [!WARNING]  
> You might need to have [Build tools for Visual Studio](https://visualstudio.microsoft.com/visual-cpp-build-tools/) (or the equivalent of it on your OS) installed

To install as module, we can use pip, with the following command.

- Install with **GPU (Cuda compatible)** support:

  `pip install -U git+https://github.com/Dadangdut33/Speech-Translate.git --extra-index-url https://download.pytorch.org/whl/cu118`

  cu118 here means CUDA 11.8, you can change it to other version if you need to. You can check older version of pytorch [here](https://pytorch.org/get-started/previous-versions/) or [here](https://download.pytorch.org/whl/torch_stable.html).

- **CPU** only:

  `pip install -U git+https://github.com/Dadangdut33/Speech-Translate.git`

You can then run the program by typing `speech-translate` in your terminal/console. Alternatively, when installing as a module, you can also clone the repo and install it locally by running `pip install -e .` in the project directory. (Don't forget to add `--extra-index-url` if you want to install with GPU support)

**Notes For Installation as Module:**

- If you are **updating from an older version**, you need to add `--upgrade --force-reinstall` at the end of the command, if the update does not need new dependencies you can add `--no-deps` at the end of the command to speed up the installation process.
- If you want to **install** from a **specific branch or commit**, you can do it by adding `@branch_name` or `@commit_hash` at the end of the url. Example: `pip install -U git+https://github.com/Dadangdut33/Speech-Translate.git@dev --extra-index-url https://download.pytorch.org/whl/cu118`
- The **--extra-index-url here is for the version of CUDA**. If your device is not compatible or you need to use other version of CUDA you can check older version of pytorch [here](https://pytorch.org/get-started/previous-versions/) or [here](https://download.pytorch.org/whl/torch_stable.html).

## From Git

If you prefer cloning the app directly from git/github, you can follow the guide in [development (wiki)](https://github.com/Dadangdut33/Speech-Translate/wiki/Development) or [below](#setup). Doing it this way might also provide a more stable environment.

# 📚 More Information

Check out the [wiki](https://github.com/Dadangdut33/Speech-Translate/wiki) for more information about the app, user settings, how to use it, and more.

# 🛠️ Development

![](speech_translate/assets/splash.png)

> [!NOTE]  
> Check the [wiki](https://github.com/Dadangdut33/Speech-Translate/wiki) for more details

## Setup

> [!NOTE]  
> It is recommended to create a virtual environment, but it is not required. I also use python 3.11.6 for development, but it should work with python 3.8 or later

> [!WARNING]  
> You might need to have [Build tools for Visual Studio](https://visualstudio.microsoft.com/visual-cpp-build-tools/) installed

1. Clone the repo with its submodules by running `git clone --recurse-submodules https://github.com/Dadangdut33/Speech-Translate.git`
2. `cd` into the project directory
3. Create a [virtual environment](https://docs.python.org/3/library/venv) by running `python -m venv venv`
4. [Activate your virtual environment](https://docs.python.org/3/library/venv.html#how-venvs-work)
5. Install all the dependencies needed by running `pip install -r requirements.txt --extra-index-url https://download.pytorch.org/whl/cu118` if you are using GPU or `pip install -r requirements.txt` if you are using CPU.
6. Run `python Run.py` in root directory to run the app.

Notes:

- If you forgot the `--recure-submodules` flag when cloning the repository and the submodules is not cloned correctly, you can do `git submodule update --init --recursive` in the project directory to pull the needed submodules.
- The `--extra-index-url` is needed to install CUDA version of pytorch and for this one we are using CUDA 11.8. If your device is not compatible or you need to use other version of CUDA you can check the previous pytorch version in this [link](https://pytorch.org/get-started/previous-versions/) or [this](https://download.pytorch.org/whl/torch_stable.html).

## Running the app

You can run the app by running the [`Run.py`](./Run.py) located in **root directory**. Alternatively you can also run it using `python -m speech_translate` in the **root directory**.

## Building

**Before compiling the project**, make sure you have installed all the dependencies and setup your pytorch correctly. Your pytorch version will control wether the app will use GPU or CPU (that's why it's recommended to make virtual environment for the project).

The pre compiled version in this project is built using cx_freeze, we have provided the script in [build.py](./build.py). This build script is only configured for windows build at the moment, but feel free to contribute if you know how to build properly for other OS.

To compile it into an exe run `python build.py build_exe` in the **root directory**. This will produce a folder containing the compiled project alongside an executable in the `build` directory. After that, use innosetup script to create an installer. You can use the provided [installer.iss](./installer.iss) to create the installer.

## Compatibility

This project should be compatible with Windows (preferrably windows 10 or later) and other platforms. But I haven't tested it extensively on other platforms. If you find any bugs or issues, feel free to create an issue.

# 💡 Contributing

Feel free to contribute to this project by forking the repository, making your changes, and submitting a pull request. You can also contribute by creating an issue if you find a bug or have a feature request. Also, feel free to give this project a star if you like it.

# License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

# Attribution

- [Sunvalley TTK Theme](https://github.com/rdbende/Sun-Valley-ttk-theme/) (used for app theme although i modified it a bit)
- [Noto Emoji](https://fonts.google.com/noto/specimen/Noto+Emoji) for the icons used in the app

# Other

Check out my other similar project called [Screen Translate](https://github.com/Dadangdut33/Screen-Translate/) a screen translator / OCR tools made possible using tesseract.
