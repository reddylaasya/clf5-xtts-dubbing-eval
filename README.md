# Controlled Evaluation of Cross-Lingual TTS Backends for Indic-to-English Dubbing

*This repository accompanies a paper under double-blind review. Author and
institutional information have been withheld and will be added upon
acceptance.*

A controlled comparison of two cross-lingual TTS backends — **Cross-Lingual
F5-TTS (CLF5)** and **XTTS-v2** — inside one fixed Indic-to-English dubbing
pipeline. Source separation, recognition and diarization, translation,
reference selection, duration fitting, and remixing use the same
implementation and configuration in both runs; only the synthesis stage is
swapped.

Evaluation covers 18 publicly accessible Hindi, Telugu, and Tamil clips
(79.2 minutes, six per language, 1–5 diarized speaker clusters), all dubbed
into English on a single NVIDIA T4.

## Pipeline

| Stage | Component | Varies between runs |
|---|---|---|
| 1 Audio extraction | FFmpeg demux, 16-bit PCM, 44.1 kHz | no |
| 2 Music–speech separation | Demucs `htdemucs`, two-stem | no |
| 3 Recognition & diarization | WhisperX large-v3 + wav2vec2 alignment, pyannote 3.1 | no |
| 4 Translation | NLLB-200 distilled 1.3B, explicit FLORES source code | no |
| 5 Speaker reference construction | own-vocals extraction, SNR/flatness/clipping scored | no |
| 6 **Cross-lingual synthesis** | **CLF5 or XTTS-v2** | **yes** |
| — Duration fitting (time-lock) | slot = own span + 85% of following silence, rate clamp 0.85–1.15 | no |
| 7 Cooperative mix & remux | 70 Hz HPF, −14 dB duck, −24 LUFS, stream-copy remux | no |

## Contents

| File | What it runs |
|---|---|
| `clf5_pipeline.ipynb` | Full CLF5 pipeline: dubbing, calibration, the nine metrics, and the no-cloning baseline arm |
| `xtts_pipeline.ipynb` | Full XTTS-v2 pipeline: same stages 1–5 and 7, plus the time-lock ablation |
| `clf5_eval_standalone.ipynb` | Evaluation only, against existing run records. Loads no pipeline models (Whisper only, for re-transcription). Also runs the paired cloned-vs-no-cloning tests |

### Configuration flags

Both pipeline notebooks are single-file and switch condition by flag rather
than by forking into separate copies:

- `ABLATE_TIMELOCK` (setup cell) — `False` runs the paper pipeline; `True`
  disables silence borrowing and the no-drop policy and writes to a separate
  output directory.
- `no_cloning` / `run_tag` (config cell, CLF5) — synthesises every segment
  from one fixed generic English prompt instead of the speaker's own
  reference. Per-speaker references are still extracted, because they remain
  the ReferenceSim targets.
- `freeze_segments` (config cell, CLF5) — reloads the cloned run's ASR, MT,
  diarization, and segment timings verbatim, so the baseline arm is scored
  against byte-identical references.

## Reproducing

**Requirements.** A CUDA GPU (results reported on a single T4), FFmpeg, and a
Hugging Face token for pyannote. Dependency installation is in the second cell
of each pipeline notebook; `coqui-tts`, `f5-tts`, and `whisperx` all pin
`transformers`, so the dependency smoke-test cell that follows is a hard gate,
not a formality. The notebooks assume a Colab runtime with Drive-mounted
persistence — substitute a local path in the setup cell to run elsewhere.

**Inputs.** Place source videos and reference transcripts in the two
directories named in the batch cell. Videos are public YouTube clips; the
repository does not redistribute them. Two references accompany each clip —
a source-language transcript and an English translation — both produced by
systems that appear nowhere in the pipeline, and both diagnostic rather than
human ground truth.

**Run order.** Setup → install → restart runtime → smoke test → config →
model loading → calibration → batch run → evaluation. Calibration derives
`tts_chars_per_sec` and `tts_speed_hi`, which are properties of the
synthesizer and must be recalibrated per backend; never carry a value across
from the other backend. Paste the calibrated values into the config cell
before the batch run, and freeze them for any baseline or ablation arm — a
re-derived value gives the second arm a different duration target for every
segment, which is a confound rather than an ablation.

**Outputs.** Each clip writes a dubbed MP4 and a JSON run record to
`eval_records/`. The metric cells read those records; the ablation and
baseline comparisons read two record directories and pair by clip ID,
reporting only clips present in both.

## Reported results

Backend comparison over 18 clips (∆ = CLF5 − XTTS-v2, two-sided Wilcoxon
signed-rank on per-clip values):

| Metric | CLF5 | XTTS-v2 | ∆ | p |
|---|---|---|---|---|
| WER (TTS) ↓ | 0.187 | 0.181 | +0.006 | 1.00 |
| CER (TTS) ↓ | 0.157 | 0.150 | +0.006 | 1.00 |
| ReferenceSim ↑ | 0.867 | 0.822 | +0.045 | 0.0003 |
| Speech Coverage % ↑ | 99.94 | 99.94 | 0.000 | — |
| RTF ↓ | 1.551 | 1.250 | +0.301 | 0.0023 |

Intelligibility is bounded below ±0.04 WER by two one-sided tests
(p = 0.040; CER p = 0.035). Seventeen listeners rate CLF5 higher on
naturalness (3.98 vs 3.73, p = 0.008) and perceived speaker similarity
(3.94 vs 3.69, p = 0.004). Against a generic-voice floor of 0.555, source
cloning raises CLF5 ReferenceSim by 0.312. Disabling silence borrowing and
the no-drop policy cuts coverage to 95.57% (CLF5) and 81.90% (XTTS-v2) over
a six-clip subset, locating timeline behaviour in the scheduling policy
rather than in either synthesizer.

Exact figures, confidence intervals, and effect sizes are in the paper.

## Not included

Following the paper's ethics statement, this repository releases no cloned
audio, no speaker-conditioning artifacts, and no dubbed videos. Source clips
remain under their original terms and are analysed rather than redistributed.
Model checkpoints are downloaded from their upstream sources at runtime.

## Scope

Results describe two specific backend configurations on 18
convenience-sampled clips in three source languages and one target
direction — a configuration-specific trade-off, not a general ranking.
Prosodic appropriateness, lip synchronization, emotional fit, and
background-mix quality are not evaluated.

## License

MIT.
