# RF Jamming Detection via Complex-Valued Spectogram Autoencoding

## The Problem

Modern radar and IFF (identify friend-or-foe) systems rely on recognising a
known, clean signal such as a specific chirp at a specific frequency which arrives at an expected time. Electronic attacks disrupt this in two different ways:

- **Noise jamming** — This floods the receiver with broadband noise so the real signal gets buried. This is easy to generate, and easy(ish) to detect because the receiver just looks noisy. Noise jamming means you know the general direction of the aircraft in some range, but can't tell how close or far it is relative to your position.
  
- **Deceptive jamming (DRFM / Range-Gate Pull-Off)** — Here, a Digital Radio Frequency Memory (DRFM) jammer records the real chirp, delays it slightly and then replays it back. That means to a receiver, the signal looks like a second, fake target. Deceptive jamming records the chirp that the enemy transmits to identify where you are, and then replays it with a delay, and since both the signal from the aircraft itself and this delayed recording gets sent to the enemy, they see 2 planes. But since the chirp that is replayed is a much juicier signal, the enemy radar believes that is the plane. (Radars use Automatic Gain Control (AGC), so when they see this juicy signal, they dial down sensitivity to prevent the circuits from being overloaded, which means the real reflection of the plane gets pushed underneath the noise.) This means DRFM can create the illusion of a fake plane flying away, by slowly increasing the intervals at which the chirp is replayed by. Since radar computers are programmed to follow moving targets, it would follow the fake signal until it gets far enough away, at which point the plane could turn of the replay, leaving no trace of any plane (presumably the plane has travelled outside the range of detection in that time). 

A system that can only catch noise jamming isn't solving the interesting
part of this problem. The goal here is to build a detector that can flag
*both* — including the deceptive case, where the "attack" looks structurally
like a real signal.

**Approach:** train an autoencoder on clean, authenticated signal only (the
IFF framing makes this natural — you only ever get to train on your own
friendly frequencies). Anything the model reconstructs badly, or reconstructs
*as* jamming, gets flagged. No jamming examples are needed at training time,
which matters in practice — you can't train on jamming that hasn't been
invented yet.

## Method Overview

- Signals are generated as complex **IQ (in-phase/quadrature) data**, not
  real-valued waveforms, to preserve phase information the way real RF
  hardware does.
- A **complex-valued convolutional autoencoder** takes I and Q as two input
  channels and learns a compressed representation of "normal" signal.
- The decoder is **dual-headed**: one head reconstructs the clean signal,
  the other isolates the jamming component itself, rather than just scoring
  "anomalous / not anomalous."
- Detection performance is evaluated across a **sweep of Signal-to-Jamming
  Ratio (SJR)**, from favourable (+10 dB) to hostile (−20 dB, jammer 100×
  stronger than the signal), to find where the approach actually breaks —
  a single accuracy number at one SJR would hide the more useful result.

## Build Stages

Each stage is a working checkpoint in its own right, not just a step toward
the next one.

### Stage 1 — Signal simulator (IQ, noise jamming, Doppler/attenuation)
Build a parameterised radar chirp + noise-jammer simulator in complex IQ
form, with a moving-transmitter model (Doppler shift, distance-based
attenuation).
**Checkpoint:** spectrograms of clean vs. noise-jammed signal, visibly
different.

### Stage 2 — Deceptive jamming (DRFM / Range-Gate Pull-Off)
Extend the simulator with a DRFM jammer: capture the chirp, delay it, replay
it as a fake target.
**Checkpoint:** spectrograms distinguishing noise jamming from DRFM spoofing.

### Stage 3 — Baseline autoencoder (real-valued)
A plain, real-valued convolutional autoencoder trained on clean spectrograms
only, scored by reconstruction error. This is the fallback that proves the
core idea works, and the baseline the fancier model is measured against.
**Checkpoint:** reconstruction error clearly separates clean from jammed.

### Stage 4 — Complex-valued, dual-head autoencoder
Native I/Q input channels, dual decoder heads (clean signal / isolated
jamming pattern), compound MSE + SSIM loss to preserve the shape of sweeping
jammers on the spectrogram, not just pixel-level error.
**Checkpoint:** the jamming-isolation head produces something recognisably
"the jammer," not noise — and outperforms the Stage 3 baseline.

### Stage 5 — Robustness evaluation
ROC curve, confusion matrix, and detection performance across the SJR sweep
(+10 dB → −20 dB). The interesting result is *where* it breaks, not just
whether it works at a convenient SJR.

### Stage 6 — Latency & quantisation *(stretch goal)*
Per-frame inference latency, checked against realistic radar pulse
repetition intervals. INT8 quantisation (ONNX Runtime), with the
accuracy-vs-speed trade-off documented honestly.

### Stage 7 — Presentation
Before/after spectrogram comparison at the top of this README once Stage 4
produces a real result. Optional lightweight Streamlit dashboard with
adjustable jammer type/power.

## Results

*To be filled in as stages complete.*

## Background

This project follows on from a placement at Leonardo, where the signal processing team covered jamming types and
frequency-based identification, and from a project on architectural
representation constraints during the Non-Trivial Fellowship.
