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
private static int LevenshteinDistance(string s, string t)
{
    int[,] d = new int[s.Length + 1, t.Length + 1];
    for (int i = 0; i <= s.Length; i++) d[i, 0] = i;
    for (int j = 0; j <= t.Length; j++) d[0, j] = j;

    for (int i = 1; i <= s.Length; i++)
    {
        for (int j = 1; j <= t.Length; j++)
        {
            int cost = (t[j - 1] == s[i - 1]) ? 0 : 1;
            d[i, j] = Math.Min(
                Math.Min(d[i - 1, j] + 1, d[i, j - 1] + 1),
                d[i - 1, j - 1] + cost);
        }
    }
    return d[s.Length, t.Length];
}

private static double SimilarityPercentage(string s, string t)
{
    int distance = LevenshteinDistance(s, t);
    int maxLength = Math.Max(s.Length, t.Length);
    return (1.0 - (double)distance / maxLength) * 100;
}

private static bool IsAtLeastXPercentSimilar(string s, string t, float threshold = 50.0f)
{
    if (s.Length > 0)
    {
        char lastChar = s[s.Length - 1];
        if (lastChar == '.' || lastChar == '!' || lastChar == '?')
            s = s.Substring(0, s.Length - 1);

        if (s.Contains(t) || t.Contains(s))
            return true;

        return SimilarityPercentage(s, t) >= threshold;
    }
    return false;
}
```

---

## Requirements

- Python 3.x
- A CUDA-capable GPU *(strongly recommended)*
- Dependencies: see `requirements.txt`

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
