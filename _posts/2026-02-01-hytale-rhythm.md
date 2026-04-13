---
layout: post
author: qbb84
tags: ["Java", "Hytale", "Python", "Audio Analysis", "Game Development", "ML"]
---

*A dev log covering rhythm, codenamed: Pulse — a Hytale server plugin that performs offline audio analysis to produce a beat map, then drives beat-synchronised in-world visual effects at runtime using Hytale's server API.*

## The Problem

Hytale has no concept of music-driven world state. Sound events play, the world does its thing, and the two are entirely decoupled. The goal of Pulse was to bridge that gap — to make the game world *react* to music in real time, with effects timed precisely to beats, onsets, section changes, and even individual words.

The constraint that shaped everything: Hytale's server runs on the main game thread. Any heavy computation done at runtime would introduce lag. Audio analysis — spectral decomposition, onset detection, ML-based transcription — is expensive. It cannot happen live.

The solution is a two-stage pipeline:

```
[Offline]  Audio file → Python analyser → Beat map JSON
[Runtime]  Beat map JSON → Pulse event dispatcher → World effects
```

The hard work happens offline, once per song. Runtime is just reading timestamps from a file and dispatching events.

---

## Stage 1 — The Python Analyser

The analyser (`analyze.py`) takes an audio file and produces a JSON beat map containing everything the plugin needs to drive effects. It uses `librosa` for audio analysis and optionally `whisperx` or `whisper` for lyric alignment.

### Tempo Detection

Reliable tempo detection is harder than it sounds — especially for hip-hop and electronic music where the perceived BPM can be half or double the detected value. I implemented a voting system across five independent methods:

- Standard onset envelope tempo estimation
- Percussive component with hip-hop prior
- Low-frequency (kick drum) autocorrelation
- Fourier tempogram peak detection
- Standard `beat_track` fallback

Candidates are binned into a histogram and the median of the peak cluster is returned. A manual BPM override is also supported for cases where the voting system drifts.

```python
def get_tempo_with_voting(y, sr, hop_length, y_percussive=None):
    candidates = []
    # ... five independent estimates ...
    hist, bin_edges = np.histogram(candidates, bins=np.arange(60, 180, 5))
    peak_bin = np.argmax(hist)
    bin_center = (bin_edges[peak_bin] + bin_edges[peak_bin + 1]) / 2
    near_peak = candidates[np.abs(candidates - bin_center) < 10]
    return float(np.median(near_peak))
```

### Frequency-Separated Onset Detection

Rather than treating the audio as a single signal, the analyser separates it into frequency bands — bass (kick drum), mids (snare, melodic), highs (hi-hats, cymbals), and vocals — and detects onsets independently in each:

```python
y_bass  = bandpass_filter(y_percussive, sr, 20, 150, hop_length)
y_mids  = bandpass_filter(y, sr, 500, 2000, hop_length)
y_highs = bandpass_filter(y, sr, 4000, 16000, hop_length)
y_vocals = bandpass_filter(y_harmonic, sr, 300, 3500, hop_length)
```

This means the plugin can drive different effects for different frequency events — bass onsets trigger ground-level effects, hi-hat onsets trigger rapid upper-register effects, vocal onsets trigger word-aligned effects.

### Continuous Curves

Beyond discrete events, the beat map also contains continuous curves sampled at regular intervals:

- **Energy curve** — normalised RMS envelope, 50ms samples
- **Spectral flux** — rate of change in the frequency spectrum
- **Pitch contour** — normalised pitch (0–1) with direction (rising/falling/stable)
- **Frequency bands** — per-band energy (sub, bass, lowMid, mid, highMid, high) for equaliser-style effects

These enable smooth interpolation between states rather than sharp on/off transitions.

### Section Detection & Structural Analysis

The analyser uses `librosa.segment.agglomerative` on combined MFCC and chroma features to find structural boundaries, then classifies each section:

```python
features = np.vstack([
    librosa.util.normalize(mfcc, axis=1),
    librosa.util.normalize(chroma, axis=1)
])
bounds = librosa.segment.agglomerative(features, k=num_sections)
```

A recurrence matrix identifies which sections repeat — chorus sections appear multiple times and get flagged accordingly. Each section is classified as intro, verse, chorus, hook, bridge, buildup, drop, breakdown, or outro, with a mood label (energetic, aggressive, melancholic, peaceful, dark, happy, etc.) derived from energy, brightness, and tempo.

### Lyric Alignment

If lyrics are provided — either as a file or fetched from the Genius API — the analyser uses WhisperX for forced alignment:

```python
result = whisperx.align(
    result["segments"], model_a, metadata, audio, device,
    return_char_alignments=False
)
```

The key insight in the alignment system: we trust our lyrics for the *text*, but use the transcription for the *timing*. A fuzzy matching pass maps known correct words to their nearest transcribed timestamp, handling the minor errors Whisper inevitably makes. The result is word-level timestamps with 100% textual accuracy.

The analyser is compiled into a platform-specific binary (Windows/Mac/Linux) via PyInstaller and called from the plugin as a subprocess:

```java
ProcessBuilder pb = new ProcessBuilder(
    getAnalyzerPath(),
    audioFile.getAbsolutePath(),
    outputFile.getAbsolutePath(),
    "--bpm", String.valueOf(bpm),
    "--song", songTitle,
    "--artist", artist
);
```

---

## Stage 2 — The Hytale Plugin

### The Beat Map

`SongBeatMap` is a flat POJO that mirrors the JSON output exactly:

```java
public class SongBeatMap {
    public double bpm;
    public double duration;
    public String mood;
    public double[] beatTimestamps;
    public double[] downbeats;
    public double[] eighthNotes;
    public double[] sixteenthNotes;
    public Onset[] bass;
    public Onset[] mids;
    public Onset[] highs;
    public Section[] sections;
    public Buildup[] buildups;
    public Drop[] drops;
    public EnergySample[] energyCurve;
    public PitchSample[] pitchContour;
    public Word[] words;
    // ...
}
```

Gson deserialises the JSON into this structure once at song load. Runtime access is just array indexing.

### The Event System

Each musical event type maps to a Hytale `IEvent` implementation:

```java
public class PulseMainBeatEvent  extends PulseEvent implements IEvent<Void>
public class PulseDownbeatEvent  extends PulseEvent implements IEvent<Void>
public class PulseOnsetEvent     extends PulseEvent implements IEvent<Void>
public class PulseSectionEvent   extends PulseEvent implements IEvent<Void>
public class PulseSongStartEvent extends PulseEvent implements IEvent<Void>
public class PulseSongEndEvent   extends PulseEvent implements IEvent<Void>
```

All inherit from `PulseEvent`, which carries a `SongContext` record — BPM, average energy, duration, beats per bar, and mood — so every listener has full song context without needing to query a shared state object:

```java
public record SongContext(
    double BPM,
    double averageEnergy,
    double duration,
    double beatsPerBar,
    String mood
) {}
```

The dispatcher reads the beat map's timestamp arrays, schedules events ahead of time using `HytaleServer.SCHEDULED_EXECUTOR`, and fires them as the song progresses. All scheduling happens on a background thread — the main game thread only runs the actual world mutations triggered by the event listeners.

### Visual Effects

`PulseEffects` implements the world-space effects. A few highlights:

**Equaliser** — a row of block columns that rise and fall in a sine wave pattern on each beat, timed with staggered delays for a ripple effect:

```java
double wave = Math.sin(Math.PI * i / (barCount - 1));
final int height = 1 + (int)(wave * (maxHeight - 1));
final int staggerDelay = i * 20; // ripple left to right
```

**Circular pillar wave** — pillars arranged in a ring, rising sequentially around the circumference so the wave propagates spatially:

```java
double angle = (2 * Math.PI / pillarCount) * i;
final int bx = (int)(center.x + Math.cos(angle) * distance);
final int bz = (int)(center.z + Math.sin(angle) * distance);
```

**Bloom pulse** — the post-processing bloom intensity is driven directly via `UpdatePostFxSettings`, allowing the screen to pulse with the beat without any world geometry changing:

```java
UpdatePostFxSettings pulsePacket = new UpdatePostFxSettings(
    intensity, 8.0f, 4.0f, 0.25f, 0.3f
);
player.getPacketHandler().writePacket(pulsePacket, true);
```

**Camera shake** — on strong hits above a threshold, `CameraShakeEffect` is sent directly to the player's packet handler:

```java
if (intensity > 0.8f) {
    playerRef.getPacketHandler().writePacket(
        new CameraShakeEffect(1, intensity * 0.3f, AccumulationMode.Average), true
    );
}
```

**Mood-mapped colours** — section mood drives the colour palette for all effects:

```java
private Color getMoodColor(String mood) {
    return switch (mood) {
        case "energetic"  -> Color.YELLOW;
        case "aggressive" -> Color.RED;
        case "melancholic"-> Color.BLUE;
        case "peaceful"   -> Color.CYAN;
        case "dark"       -> Color.MAGENTA;
        case "happy"      -> Color.PINK;
        default           -> Color.WHITE;
    };
}
```

### The Spline

`PulseSpline` wraps Hytale's `IPath` API to expose a `getPositionAt(double t)` method — given a normalised time value (0–1), it returns the corresponding world-space position along a designer-placed path. This is how effects are anchored to the world spatially: the progress of the song maps directly to position along a spline, so a four-minute song traverses the entire path exactly once:

```java
public Vector3d getPositionAt(double t) {
    t = Math.max(0, Math.min(1, t));
    int segments = points.size() - 1;
    double scaledT = t * segments;
    int segment = Math.min((int) scaledT, segments - 1);
    double localT = scaledT - segment;
    // linear interpolation between adjacent waypoints
    ...
}
```

---

## Architecture Summary

The two-stage design is the key architectural decision. Hytale servers handle real-time game state for many concurrent players — introducing a Python audio analysis pipeline into the hot path would be catastrophic. By doing all analysis offline and reducing runtime to scheduled event dispatch from a pre-computed data structure, Pulse adds zero measurable overhead to the server tick.

The event system mirrors Hytale's own `IEvent` pattern, meaning Pulse effects integrate naturally with any other plugin listening to the same bus. A game designer could respond to `PulseMainBeatEvent` the same way they'd respond to any other server event — no special integration required.

[View Repo](https://github.com/qbb84/Hytale-Rhythm)
