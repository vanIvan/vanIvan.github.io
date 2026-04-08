---
layout: post
title: "We benchmarked 4 neural forced aligners across multiple languages. Here's what actually works."
date: 2026-04-07
image: /assets/posts/forced-aligner-bench/forced-alignment-heatmap.png
excerpt: >-
  Notes from a benchmark comparing four neural forced aligners — Seamless,
  WhisperX, Qwen3-ForcedAligner, and a commercial cloud ASR — on FLEURS
  across 9 languages, 4 language families, and 3 writing systems.
description: >-
  Implicit WER-based evaluation of Seamless, WhisperX, Qwen3-ForcedAligner,
  and a commercial cloud ASR on FLEURS across 9 languages (EN/ES/FR/RU/DE/TR/HI/KO/JA).
---

Many TTS data pipelines rely on forced alignment for segmentation and labeling — but there's no good public benchmark comparing neural approaches **across languages**. Existing evaluations ([Aligner-SUPERB](https://github.com/lifeiteng/Aligner-SUPERB), [Tradition or Innovation, Interspeech 2024](https://arxiv.org/abs/2406.19363)) compare aligners against ground-truth word timestamps on English corpora (TIMIT, Buckeye). That's useful academically, but ground-truth timestamps are expensive to produce and practically don't exist outside English.

The thing is — you don't actually need them. Quality multilingual (audio, transcript) datasets already exist: FLEURS covers 100+ languages, Common Voice and MLS cover dozens more. The bottleneck isn't data, it's the evaluation method. So we built one that works with any (audio, transcript) corpus.

We tested [Seamless](https://github.com/facebookresearch/seamless_communication) (Meta), [WhisperX](https://github.com/m-bain/whisperX), [Qwen3-ForcedAligner](https://huggingface.co/Qwen/Qwen3-ForcedAligner-0.6B), and a commercial cloud ASR API on [FLEURS](https://huggingface.co/datasets/google/fleurs) across 9 languages — EN, ES, FR, RU, DE, TR, HI, KO, JA — spanning 4 language families and 3 writing systems.

## Method

Instead of ground-truth timestamps, we use an implicit WER-based evaluation:

1. Run a forced aligner on (audio, transcript) pairs to get word-level timestamps
2. Crop audio at the aligned word boundaries
3. Re-transcribe each crop with an independent ASR (Whisper large-v3-turbo)
4. Compute WER between the crop transcription and the expected text

**Better alignment = tighter crops = lower WER.** Accurate boundaries mean each crop contains exactly the target word. Misaligned boundaries cut words or bleed neighboring audio, causing transcription errors.

To isolate alignment quality from aligner ASR quality, results are filtered to utterances where **all** aligners produced perfect transcription reconstruction (strict mode). Inner words only (`[1:-1]`) are used to avoid edge effects at utterance boundaries.

![Benchmark pipeline: align → crop → re-transcribe → score]({{ "/assets/posts/forced-aligner-bench/forced-aligner-bench-scheme-rounded.drawio.png" | relative_url }}){: .post-figure }

## Aligners

| Aligner | Type | Approach |
|---------|------|----------|
| **Seamless** | NAR T2U + unit extraction | Duration prediction with monotonic alignment (RAD-TTS based) |
| **Qwen3** | Transformer forced aligner | LLM-based slot-filling with discrete timestamp prediction |
| **WhisperX** | CTC-based | Word boundaries extracted from wav2vec2 CTC emissions |
| **Cloud ASR** | Cloud API | Commercial transcription API with alignment output |

## Results

Dataset: FLEURS test split, strict filtering (intersection of utterances with perfect transcription across all aligners), inner words only.

![Mean error rate by language and aligner]({{ "/assets/posts/forced-aligner-bench/forced-alignment-heatmap-9lang.png" | relative_url }}){: .post-figure }

A few notes on the extended set: KO and JA use CER instead of WER (character-level evaluation avoids tokenization granularity issues — see below). Qwen3 doesn't support TR or HI. Hindi is hard for everyone (~36% best error rate), partly because strict-perfect filtering reduces the test set to just 73 utterances.

## Takeaways

**Seamless is the most consistent performer overall** — best on 6 of 9 languages (EN, ES, DE, TR, KO, JA). Duration prediction with monotonic alignment produces stable word boundaries across languages and scripts. The model explicitly learns duration, not just token occurrence.

**WhisperX degrades significantly beyond Western European languages.** CTC emissions predict token occurrence in sequence, not precise duration. The spiked posteriors make boundary extraction unreliable — on Korean, median WER is 0.946, meaning boundaries are essentially destroyed. The 1-second minimum input of wav2vec2 CTC is hostile to short per-word crops on agglutinative languages.

**Qwen3-ForcedAligner** is competitive on Western European languages (consistently 2nd behind Seamless on EN/ES/FR/RU/DE), but struggles on KO and JA where its morphological tokenization fragments words and produces long CER tails. It also only supports 11 languages, leaving TR and HI uncovered.

**Cloud ASR edges out on FR, RU, and HI**, but margins are small on FR/RU and Seamless wins on average. Worth noting: alignment comes bundled with their transcription response, so we filtered to utterances where the API's recognition exactly matched FLEURS ground truth. This reduced the test set and may slightly favor it.

**Japanese is a special case.** No word boundaries in text means each aligner tokenizes differently: Seamless produces ~3 sentencepiece chunks per utterance, Qwen3 ~30 morphemes, WhisperX ~50 CTC tokens. After `[1:-1]` filtering, Seamless test set shrinks to 116 utterances vs 234 for others. Seamless still wins on CER, but the comparison isn't fully apples-to-apples.

**Seamless' practical limitation:** fixed set of 38 languages. If yours isn't covered, you're back to CTC-based approaches.

For our production pipeline at [Palabra AI](https://palabra.ai), Seamless is the current choice.

---

## Implementation

The benchmark infrastructure — alignment wrappers, crop logic, WER computation, multi-GPU extraction — was built with [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview). Each aligner is wrapped behind a unified interface, and the full pipeline (align → crop → transcribe → score) runs with a couple of bash scripts.

Code & benchmark: [github.com/PalabraAI/forced-aligners-bench](https://github.com/PalabraAI/forced-aligners-bench)