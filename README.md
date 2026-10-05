# Offline Multi-Language Real-Time Speech-to-Text

Built during a short summer internship at the **Belgian Defence, XR-Labs**.

A real-time, offline speech-to-text system built on faster-whisper.
It can run on its own, or stream its transcriptions to a C# application through a Windows named pipe.

---

## Overview

There are two variations:
- **Pipe version** (`ProgramWithPipe.py`): sends every transcription to a C# program through a named pipe
- **Standalone version** (`ProgramWithoutPipe.py`): prints transcriptions to the console

> **Recommendation:** Use CUDA if possible. CPU inference works but is significantly slower.

Both versions record and transcribe on separate threads, because this was built to run in parallel with an existing application.
If you find any improvements, feel free to open an issue or a PR.

---

## Tech Stack

- **Python**: audio capture and transcription
- **[faster-whisper](https://github.com/SYSTRAN/faster-whisper)**: offline speech recognition (a faster reimplementation of OpenAI's Whisper), using the `small` model
- **PyAudio**: microphone input
- **CUDA**: GPU acceleration (PyTorch is used to detect whether CUDA is available)
- **pywin32**: Windows named pipe to the C# application
- **C#**: command matching (separate process, not included in this repository)
- **Levenshtein distance**: fuzzy string matching

---

## How It Works

### Chunk-Based Audio Processing

Rather than recording a full clip and transcribing it all at once (which would cause noticeable delays),
audio is recorded continuously in **chunks of about 2 seconds**.
Chunks that stay below a volume threshold are treated as silence and skipped.
Every other chunk is transcribed on its own thread while recording continues.

The order in which chunks are transcribed does not need to match the recording order.
If chunk 2 finishes transcribing before chunk 1, its result is passed along immediately.
This keeps the pipeline saturated and avoids stalls.

### Communication with the C# Application

`ProgramWithPipe.py` creates a named pipe called `\\.\pipe\CSServer` and waits for the C# application to connect
before it loads the model. Each transcription is then written to the pipe as a single line ending in `\n`.

### String Matching via Levenshtein Distance

Once a word or sentence is transcribed, the C# application matches it against a set of known commands.
That C# code lives in the XR-Labs codebase and is not part of this repository; the snippet below shows the matching logic for reference.

Exact string matching is too brittle for real-world speech, so this uses
[Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance)
to measure how similar two strings are.

![Levenshtein distance animation](https://github.com/user-attachments/assets/2f971679-5836-47ce-8cdc-cc4b5836ba52)

A similarity threshold (default: 50%) determines whether a transcribed string counts as a match.
Trailing punctuation (`.`, `!`, `?`) is stripped before comparison, and an exact match is checked first as a fast path.

```csharp
private static int LevenshteinDistance(string source, string target)
{
    if (string.IsNullOrEmpty(source)) return target?.Length ?? 0;
    if (string.IsNullOrEmpty(target)) return source.Length;

    int[] previousRow = new int[target.Length + 1];
    int[] currentRow = new int[target.Length + 1];

    for (int targetIndex = 0; targetIndex <= target.Length; targetIndex++)
        previousRow[targetIndex] = targetIndex;

    for (int sourceIndex = 1; sourceIndex <= source.Length; sourceIndex++)
    {
        currentRow[0] = sourceIndex;

        for (int targetIndex = 1; targetIndex <= target.Length; targetIndex++)
        {
            bool charactersMatch = source[sourceIndex - 1] == target[targetIndex - 1];

            int insertionCost    = currentRow[targetIndex - 1] + 1;
            int deletionCost     = previousRow[targetIndex] + 1;
            int substitutionCost = previousRow[targetIndex - 1] + (charactersMatch ? 0 : 1);

            currentRow[targetIndex] = Math.Min(Math.Min(insertionCost, deletionCost), substitutionCost);
        }

        Array.Copy(currentRow, previousRow, currentRow.Length);
    }

    return previousRow[target.Length];
}

private static bool IsAtLeastXPercentSimilar(string source, string target, double threshold = 50.0)
{
    if (string.IsNullOrWhiteSpace(source) || string.IsNullOrWhiteSpace(target)) return false;

    // Symmetrically trim punctuation to ensure a fair comparison
    source = source.TrimEnd('.', '!', '?').Trim();
    target = target.TrimEnd('.', '!', '?').Trim();

    if (source == target) return true;

    int distance = LevenshteinDistance(source, target);
    int maxLength = Math.Max(source.Length, target.Length);
    double similarityPercent = (1.0 - (double)distance / maxLength) * 100.0;

    return similarityPercent >= threshold;
}
```

---

## Requirements

- Windows (both scripts use pywin32)
- Python 3.x
- A microphone
- A CUDA-capable GPU *(strongly recommended)*
- An internet connection on the first run, to download the Whisper model. After that, everything runs offline.

---

## Installation

```bash
pip install faster-whisper numpy pyaudio pywin32 torch
```

For GPU support, install the CUDA build of PyTorch from [pytorch.org](https://pytorch.org/get-started/locally/)
instead of the default one, and install the NVIDIA libraries listed in
[faster-whisper's GPU requirements](https://github.com/SYSTRAN/faster-whisper#gpu).

---

## Usage

**Standalone:**
```bash
python ProgramWithoutPipe.py
```

**With the C# application:**
```bash
python ProgramWithPipe.py
```
Then start the C# application. Transcription only begins once it has connected to the `CSServer` pipe.

---

## Configuration

The settings are defined directly in the scripts:

| Setting | Default | Where to change it |
|---|---|---|
| Language | Dutch (`language="nl"`) | `transcribe_chunk`: use any Whisper language code, or remove the argument to let Whisper detect the language |
| Model size | `small` | `main`: `WhisperModel("small", ...)` |
| Microphone | The first microphone with "VIVE" in its name, otherwise the first input device | `find_vive_microphone` |
| Chunk length | 2 seconds | `record_chunk`: `chunk_length` |
| Silence threshold | Peak amplitude of 500 | `record_chunk`: `threshold` |

Whisper sometimes hallucinates phrases such as "Bedankt voor het kijken!" ("Thanks for watching!") on near-silent audio.
These known phrases are filtered out in `main`. If you switch to another language, you may need to add that language's equivalents.
