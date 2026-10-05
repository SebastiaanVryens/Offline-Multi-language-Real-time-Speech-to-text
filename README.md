# Offline Multi-Language Real-Time Speech-to-Text

Built during a short summer internship at the **Belgian Defence, XR-Labs**.

A real-time, offline speech-to-text system supporting multiple languages,
designed to run alongside a C# application via a threaded pipeline.

---

## Overview

There are two variations:
- **Threaded version**, communicates with a C# program via an inter-process pipeline
- **Standalone version**, a simple Python script that runs independently

> **Recommendation:** Use CUDA if possible. CPU inference works but is significantly slower.

The threaded design exists because this was built to run in parallel with an existing application.
If you find any improvements, feel free to open an issue or a PR.

---

## Tech Stack

- **Python**: audio capture and transcription
- **Whisper (OpenAI)**: offline speech recognition model
- **CUDA / PyTorch**: GPU acceleration
- **C#**: command-matching pipeline (separate process)
- **Levenshtein distance**: fuzzy string matching

---

## How It Works

### Chunk-Based Audio Processing

Rather than recording a full clip and transcribing it all at once (which would cause noticeable delays),
audio is split into **10 continuous chunks** that are recorded and transcribed in parallel.

The order in which chunks are transcribed does not need to match the recording order.
If chunk 2 finishes transcribing before chunk 1, its result is passed along immediately.
This keeps the pipeline saturated and avoids stalls.

### String Matching via Levenshtein Distance

Once a word or sentence is transcribed, it needs to be matched against a set of known commands.
Exact string matching is too brittle for real-world speech, so this uses
[Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance)
to measure how similar two strings are.

![Levenshtein distance animation](https://github.com/user-attachments/assets/2f971679-5836-47ce-8cdc-cc4b5836ba52)

A similarity threshold (default: 50%) determines whether a transcribed string counts as a match.
Punctuation is stripped before comparison, and substring containment is checked first as a fast path.

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

- Python 3.x
- A CUDA-capable GPU *(strongly recommended)*

---

## Usage

**Standalone:**
```bash
python speech_to_text.py
```

**Threaded (with C# pipeline):**
```bash
python speech_to_text_threaded.py
```
