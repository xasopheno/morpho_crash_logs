# Neutone Morpho — Crash Log Analysis

## Summary

All four logs come from `AUHostingServiceXPC_arrow`, the out-of-process Audio
Unit host that macOS uses to sandbox AUv2 plug-ins. In every report the
offending plug-in is **Neutone Morpho** (`com.Neutone.neutone_morpho`,
`/Library/Audio/Plug-Ins/Components/Neutone Morpho.component`). The host
itself is just the wrapper — the bug is in Neutone Morpho.

- Host machine: Mac16,5, Apple Silicon (arm64e), 128 GB RAM
- OS: macOS 15.7.4 (24G517)
- Date: 2026-04-07
- DAW (where identifiable): Logic Pro

## What's obvious

There are two distinct failure modes.

### 1. Null-pointer crash on the audio render thread (3 crashes)

Files:
- `AUHostingServiceXPC_arrow-2026-04-07-110609.ips`
- `AUHostingServiceXPC_arrow-2026-04-07-112325.ips`
- `AUHostingServiceXPC_arrow-2026-04-07-113956.ips`

All three are the same crash:

- **Exception:** `EXC_BAD_ACCESS (SIGSEGV)` /
  `KERN_INVALID_ADDRESS at 0x0000000000000000`
- **Faulting thread:** the AU render thread
- **Backtrace (top frames):**
  ```
  Neutone Morpho   NeutoneAudioProcessor::processBlock(...)   <-- crash
  Neutone Morpho   NeutoneAudioProcessor::processBlock(...)
  Neutone Morpho   JuceAU::Render(...)
  Neutone Morpho   AUBase::DoRender(...)
  Neutone Morpho   AUMethodRender(...)
  AudioToolboxCore AudioUnitRender
  AudioToolboxCore AUAudioUnitV2Bridge_Renderer::renderBlock()
  ```

A null-pointer dereference inside `NeutoneAudioProcessor::processBlock` on the
realtime audio thread. The fault address `0x0` and the recursive
`processBlock → processBlock` frames suggest the inner call is dispatching
into a member (model, buffer, or wrapped processor) that is `nullptr` —
likely because the model/engine isn't fully constructed (or has been torn
down) by the time `processBlock` is invoked. Classic lifetime/init-order bug:
the render thread is running while the model pointer is either not yet set
or has just been reset.

The three crashes happened within ~33 minutes of each other
(11:06, 11:23, 11:39), which is consistent with the user repeatedly
re-loading or re-instantiating the plug-in after each crash.

### 2. Disk-write resource exhaustion (1 diagnostic report)

File: `AUHostingServiceXPC_arrow_2026-04-07-214453.diag`

This is **not** a crash — it's a `disk writes` resource report. The AU host
process was killed/flagged for writing too much to disk:

- **2147.49 MB** of file-backed memory dirtied over **23,185 seconds**
  (~6.4 hours)
- Average **92.62 KB/s**, against a sustained limit of **24.86 KB/s** over 24h
- Triggered while hosted by **Logic Pro**

The heaviest stack points squarely at Neutone Morpho's WebView-based UI
bridge:

```
juce::WebViewDelegateClass::didReceiveScriptMessage(...)
  tomduncalf::BrowserIntegration::BrowserComponent::scriptMessageReceived(juce::var)
    BrowserIntegration::BrowserIntegration(...)::$_0
      NeutoneBrowserIntegrationPluginClient::setupBrowserPluginIntegration()::$_6
        juce::Base64::convertFromBase64(juce::OutputStream&, ...)
          juce::OutputStream::writeByte(char)
```

The plug-in's embedded browser is receiving JS→native script messages
carrying base64-encoded payloads, decoding them, and writing the bytes out
one byte at a time via `OutputStream::writeByte`. Over a multi-hour Logic
session this added up to ~2 GB of dirtied file-backed pages — enough for
macOS to flag it as a runaway disk writer.

Likely culprits:
- Writing decoded base64 blobs (model files? cached assets? telemetry?
  waveform/preview data?) to disk on every script message
- Doing it byte-by-byte with no buffering, which amplifies dirtied pages
- No throttling / dedup, so repeated UI activity keeps re-writing the same
  data

## Suggested next steps

1. **Render-thread crash** — audit `NeutoneAudioProcessor::processBlock` for
   any pointer it dereferences without a null-check (model, inference
   engine, ring buffer, child processor). Confirm `prepareToPlay` /
   model-load completes-before any `processBlock` call, and that
   `releaseResources` / model-swap can't null something out from under an
   in-flight render block. A simple `if (model == nullptr) { buffer.clear();
   return; }` guard at the top of `processBlock` would at least convert the
   crash into silence.
2. **Disk-write blow-up** — in `BrowserIntegration` /
   `NeutoneBrowserIntegrationPluginClient::setupBrowserPluginIntegration`,
   wrap the `OutputStream` used by `Base64::convertFromBase64` in a
   `BufferedOutputStream`, and figure out *why* the script-message handler
   is writing GB-scale data over a session — it's almost certainly either
   unbounded logging or repeatedly re-persisting the same asset.
