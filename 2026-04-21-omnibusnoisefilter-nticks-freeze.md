# Bug: OmnibusNoiseFilter Freezes `m_nticks` from First Event, Causing Zero-Padding Corruption in SP

**Date**: 2026-04-21  
**File**: `OmnibusNoiseFilter.cxx`  
**Symptom**: Saturated/corrupted deconvolved waveforms on top APAs (apa3/apa4) for event 448369 only; events 448361 and 448365 look normal. Affects all views (U, V, W) of the top APAs. Bottom APAs unaffected.

---

## Root Cause

`OmnibusNoiseFilter::m_nticks` is initialized to 0 and set **only once** from the first trace it processes:

```cpp
// OmnibusNoiseFilter.cxx (before fix)
if (! m_nticks) {
    m_nticks = traces.at(0)->charge().size();
}
```

This value is then used every subsequent event to allocate output traces and pad/truncate input waveforms:

```cpp
signal->charge().assign(charge.begin(), charge.begin() + std::min(m_nticks, ncharges));
signal->charge().resize(m_nticks, 0.0);  // pads with zeros if ncharges < m_nticks
```

### Why it breaks on event 3 (448369), top APAs only

The raw HDF5 data delivers a **variable number of samples per event** for top-APA channels:

| APA | Events 448361 & 448365 | Event 448369 |
|-----|------------------------|--------------|
| Bottom (idents 0–3) | 9766 samples @ 512 ns | 9766 samples @ 512 ns |
| Top (idents 4–7)   | **10048 samples @ 500 ns** | **10000 samples @ 500 ns** |

Bottom APAs go through a **Resampler** (by design) which converts 9766 → 10000 ticks @ 500 ns — this resampler reads the actual per-channel sample count so it always outputs a clean fixed-length frame regardless of input size.

Top APAs have **no Resampler**. Their raw sample count flows directly into `OmnibusNoiseFilter`.

Sequence of events:
1. **Event 1 (448361)**: top APA traces have 10048 samples → `m_nticks = 10048`. NF output: 10048 ticks. SP processes 10048 ticks. ✓
2. **Event 2 (448365)**: `m_nticks` still 10048. Raw traces have 10048 → assign fits exactly. ✓
3. **Event 3 (448369)**: `m_nticks` still 10048. Raw traces have only **10000** samples. `assign` copies 10000 real samples, `resize` **pads 48 zeros at the end**. The NF output has 10048 ticks: 10000 real + 48 appended zeros. SP receives a waveform with an abrupt zero-step at tick 10000 → deconvolution FFT spreads this discontinuity across all ticks → **entire waveform corrupted** (Gibbs phenomenon). ✗

### Secondary issue: output truncated to 6000 ticks

`wclsFrameSaver` (`spsaver`) was configured with `nticks: -1`, which causes it to call `detProp.NumberTimeSamples()`. This value is set to 6000 in `all.fcl` (a library FCL not directly editable per-run), so the `recob::Wire` output was capped at 6000 ticks regardless of actual signal length.

---

## Fix

### Fix 1 — `OmnibusNoiseFilter.cxx`

Always update `m_nticks` from the actual frame, not just on first call:

```cpp
// Before:
if (! m_nticks) {
    m_nticks = traces.at(0)->charge().size();
}

// After:
{
    const size_t frame_nticks = traces.at(0)->charge().size();
    if (m_nticks && m_nticks != frame_nticks) {
        log->warn("call={} nticks changed from {} to {}, updating", m_count, m_nticks, frame_nticks);
    }
    m_nticks = frame_nticks;
}
```

After this fix, event 448369 top APAs deliver `m_nticks=10000` → NF outputs a clean 10000-tick frame → SP processes 10000 ticks with no zero-padding → deconvolution correct.

### Fix 2 — `wcls-nf-sp.jsonnet`

Set explicit `nticks` in `spsaver` to avoid dependence on `DetectorProperties.NumberTimeSamples`:

```jsonnet
// Before:
nticks: -1,  // falls back to detProp.NumberTimeSamples() = 6000

// After:
nticks: params.daq.nticks,  // 10000
```

---

## Files Changed

- `wire-cell-toolkit/sigproc/src/OmnibusNoiseFilter.cxx` — lines ~146–150
- `wire-cell-cfg/pgrapher/experiment/protodunevd/wcls-nf-sp.jsonnet` — `spsaver.nticks`

## How to distinguish bottom vs top APAs in the pipeline

The pipeline builds 8 separate branches at **configuration time** (not runtime) via a loop over `n = 0..7` in `wcls-nf-sp.jsonnet`:
- `n < 4` → bottom anodes (idents 0–3) → Resampler inserted between ChannelSelector and NF
- `n ≥ 4` → top anodes (idents 4–7) → no Resampler

The Resampler protects bottom APAs from this class of bug by always outputting a fixed-length clean frame. Top APAs are vulnerable to variable raw sample counts.
