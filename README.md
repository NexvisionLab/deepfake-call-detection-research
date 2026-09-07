# Real-Time Video-Call Liveness & Deepfake Detection: Feasibility Research

**Status: early-stage technical research. Not a shipped product.**

This report summarizes exploratory research into detecting AI-generated ("deepfake") participants on live video calls in real time — the kind of attack that cost the engineering firm Arup $25 million in January 2024, when finance staff in its Hong Kong office were tricked into 15 separate wire transfers after joining a video call where every other participant, including senior executives, was an AI-generated deepfake ([reporting: CNN/Fortune, May 2024](https://fortune.com/europe/2024/05/17/arup-deepfake-fraud-scam-victim-hong-kong-25-million-cfo)). Attacks like this are built from nothing more exotic than publicly available footage of the people being impersonated — company town halls, conference talks, earnings calls — run through commercially available real-time face-swap and voice-cloning tools.

## The problem

Most deepfake-detection research targets **pre-recorded video**: given a finished clip, look for compression artifacts, inconsistent lighting, or unnatural blinking. A live video call is a harder and more time-sensitive problem — detection has to work on a continuous stream, fast enough to warn a participant *during* the call, not after money has already moved.

## Our approach: remote photoplethysmography (rPPG)

Rather than looking for visual artifacts (which improve every year as generation models improve), our research explores a physiological signal instead: a living person's skin shows a faint, periodic color change driven by blood flow with every heartbeat — invisible to the eye, but recoverable from ordinary video. A synthetic or face-swapped video has no underlying circulatory system to reproduce this signal, even when it looks visually convincing. This is the same principle behind Intel's published FakeCatcher research.

We implemented **CHROM** (de Haan & Jeanne, 2013), a chrominance-based pulse-extraction method chosen specifically for its resistance to head motion and lighting changes — both common on a real video call, where a naive single-color-channel approach is easily fooled.

## What we've validated so far

All results below come from synthetic ground-truth testing of the signal-processing core — real skin-tone data with realistic sensor noise and a known, injected pulse signal — used to prove the underlying method before investing in real-video pipelines.

**Frequency recovery.** Across a physiologically realistic range (50–165 beats per minute) with a realistic pulse amplitude (0.4% of the baseline signal — real rPPG signals are this faint), our implementation recovered the true heart rate to within 0–2 beats per minute in almost every case.

**Robustness under motion.** This is the result we consider most significant. We injected a shared brightness artifact — simulating head movement or a lighting change — directly into the same frequency band as the pulse signal, at up to five times the pulse's own amplitude. A naive single-channel approach was completely fooled: it locked onto the motion artifact and reported it as the "pulse" with *higher* confidence than a real signal would produce. Our CHROM-based implementation correctly ignored the artifact and stayed locked onto the true pulse throughout.

**A real limit, not hidden.** Below roughly 0.2% pulse amplitude, our implementation cannot reliably distinguish a genuine-but-faint pulse from no pulse at all — both score in the same low-confidence range. Real-world pulse signals are already in the 0.1–1% range before video-call compression (which specifically degrades exactly this kind of subtle color detail) makes things harder still. This is the single largest open risk to the whole approach and is discussed further below.

## What we have not yet validated

We're publishing this while the research is still in progress, not after the fact, because we think the open questions are as useful to share as the results:

- **Real video.** Every result above comes from synthetic data engineered to have known properties. Face detection and region-of-interest extraction from an actual camera feed have not yet been tested against real footage.
- **Video-call compression.** Real calls run through lossy codecs (H.264, VP8) tuned to discard exactly the kind of subtle color detail this method depends on. Whether the signal survives real-world compression is untested.
- **Deepfake-specific behavior.** Published research has shown that *some* face-swap and voice-conversion techniques can inadvertently preserve fragments of the original source video's physiological signal. Whether this undermines detection in practice is a literature question we haven't yet resolved, not a synthetic-data question.
- **True real-time operation.** Our validation used static, several-second-long analysis windows. A production system watching a live call continuously would need a sliding-window design, trading detection latency against frequency resolution — an engineering tradeoff we haven't yet made.

## Why this approach, in one sentence

Visual deepfake artifacts are a moving target that gets harder to detect as generation models improve; a physiological signal grounded in actual blood flow is a much harder thing for a generative model to fake, because nothing in its training or inference process ever modeled a circulatory system in the first place.

## Next steps

1. Validate the full pipeline (face detection through pulse extraction) against real camera footage.
2. Test signal survival through realistic video-call compression pipelines.
3. Review the literature on rPPG-signal leakage in modern face-swap and voice-conversion methods.
4. Design a sliding-window architecture suitable for genuinely continuous, real-time monitoring.

---

*This is a research summary, not a product announcement or a claim of production readiness. Figures cited are from internal synthetic validation and are not an independent benchmark.*
