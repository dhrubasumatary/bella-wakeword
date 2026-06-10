# bella-wakeword

Custom on-device **"Hey Bella"** wake word for the
[Home Assistant Voice Preview Edition](https://www.home-assistant.io/voice-pe/)
(Voice PE) running ESPHome `micro_wake_word`.

## Files

| File | Purpose |
| --- | --- |
| `hey_bella.json` | microWakeWord **v2** manifest (points at `hey_bella.tflite`). |
| `hey_bella.tflite` | The trained wake-word model. |
| `home-assistant-voice-local.yml` | A local copy of the upstream Voice PE device YAML, patched to load `hey_bella` instead of the stock `okay_nabu` model. |

## How the model is loaded

In `home-assistant-voice-local.yml` the `micro_wake_word:` block lists the
models. The custom model is referenced by a **full raw HTTPS URL** to its
manifest:

```yaml
micro_wake_word:
  id: mww
  models:
    - model: https://raw.githubusercontent.com/alex323451/bella-wakeword/main/hey_bella.json
      id: hey_bella          # first slot => enabled by default on first boot
    - model: hey_jarvis
      id: hey_jarvis
    - model: hey_mycroft
      id: hey_mycroft
    - model: https://github.com/kahrendt/microWakeWord/releases/download/stop/stop.json
      id: stop
      internal: true         # required by the timer ("stop") logic
  vad:
```

ESPHome downloads the manifest, then resolves the manifest's relative
`"model": "hey_bella.tflite"` field **against the manifest URL**, fetching the
`.tflite` from the same directory. This is the same mechanism the stock
`okay_nabu` and `stop` entries use, so it is the most broadly compatible model
source — it does not depend on `github://` shorthand resolution or a git clone
inside the ESPHome add-on.

> The manifest's `model` field must stay a **relative filename**
> (`hey_bella.tflite`), not a full URL, so relative resolution works.

## Why `okay_nabu` is replaced (not added as a 5th model)

Voice PE ships four wake-word models. Putting `hey_bella` in the **first** slot
makes it the model enabled by default on first boot, and replacing `okay_nabu`
keeps the model count (and PSRAM/tensor-arena usage) the same as stock. The
`stop` model is kept because the timer logic enables/disables it, and
`hey_jarvis`/`hey_mycroft` are kept because the "Wake word sensitivity" select
lambda tunes them. The lambda was updated to tune `hey_bella` in place of
`okay_nabu`.

## Validate / flash

Requires ESPHome **2026.5.0+** (the device YAML sets `min_version: 2026.5.0`).

```bash
# Validate the configuration (downloads + checks all model manifests)
esphome config home-assistant-voice-local.yml

# Compile and flash to the connected Voice PE
esphome run home-assistant-voice-local.yml
```

Then say **"Hey Bella"** to trigger the assistant.

## Troubleshooting

`Invalid manifest file: Expected a file scheme or a URL scheme with host`

This is ESPHome's `cv.url` validator rejecting an invalid URL while parsing a
**model manifest**. It is caused by a malformed `hey_bella.json` — most often an
empty or non-URL `website` field (`"website": ""`) or invalid JSON (e.g. a
missing comma). Make sure `hey_bella.json` is valid JSON and that `website` is a
full URL (or omitted — it is optional in v2 manifests).

If you previously flashed with a broken manifest, clear the ESPHome model cache
so the corrected manifest is re-downloaded:

```bash
rm -rf .esphome/micro_wake_word   # next to the YAML, or ~/.esphome/... for the add-on
```
