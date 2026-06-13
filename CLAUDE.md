# CLAUDE.md

Instructions for Claude Code when working with this repository.

## Project Overview

OB-Xd module for Move Anything - an Oberheim OB-X synthesizer emulator based on the OB-Xd project.

## Architecture

```
src/
  dsp/
    obxd_plugin.cpp     # Main plugin wrapper
    Engine/             # OB-Xd synth engine
      SynthEngine.h     # Main synthesis
      Voice.h           # Voice management
      Filter.h          # 4-pole filter
      Oscillator.h      # BLEP oscillators
      Lfo.h             # LFO
  presets/              # Factory presets (.fxb banks)
  ui.js                 # JavaScript UI with parameter banks
  web_ui.html           # Graphical Remote UI (schwung-manager iframe)
  module.json           # Module metadata + ui_hierarchy mirror
```

## Key Implementation Details

### Plugin API

Implements Move Anything plugin_api_v2 (multi-instance):
- `create_instance`: Initializes synth engine, loads presets
- `destroy_instance`: Cleanup
- `on_midi`: Routes to synth engine
- `set_param`: preset, octave_transpose, and 75 synth parameters
- `get_param`: preset_name, preset_count, ui_hierarchy, chain_params, parameter values
- `render_block`: Renders synth output

### Parameters (75 total)

All parameters from the original OB-Xd engine (`ParamsEnum.h`, identical to upstream
2DaT/Obxd) are exposed via the `g_shadow_params[]` table in `obxd_plugin.cpp`. Adding a
parameter means: (1) add a row to `g_shadow_params[]`, (2) add a `case` to
`v2_apply_param_direct()` (and `v2_apply_preset()` if it should restore from .fxb), and
(3) place its key in the relevant `ui_hierarchy` level. chain_params JSON is generated
automatically from the table.

Parameters are organized into categories for Shadow UI hierarchy navigation:

**Global**
- `volume`, `tune`, `octave`, `voice_count`, `legato`, `portamento`, `unison`, `unison_det`, `as_played` (toggle), `economy` (toggle)

**Oscillator 1**
- `osc1_saw` (toggle), `osc1_pulse` (toggle), `osc1_pitch`, `osc1_mix`

**Oscillator 2**
- `osc2_saw` (toggle), `osc2_pulse` (toggle), `osc2_pitch`, `osc2_mix`, `osc2_detune`, `osc2_sync` (toggle), `osc_quantize` (toggle, stepped osc2 pitch)

**Osc Common**
- `pw`, `pw_env`, `pw_env_both` (toggle), `pw_ofs`, `noise`, `xmod`, `brightness`

**Filter**
- `cutoff`, `resonance`, `filter_env`, `key_follow`, `multimode`, `bandpass` (toggle), `fourpole` (toggle), `self_osc` (toggle), `fenv_inv` (toggle)

**Filter Envelope**
- `f_attack`, `f_decay`, `f_sustain`, `f_release`, `vel_filter`

**Amp Envelope**
- `attack`, `decay`, `sustain`, `release`, `vel_amp`

**LFO**
- `lfo_rate`, `lfo_sin` (toggle), `lfo_square` (toggle), `lfo_sh` (toggle), `lfo_sync` (toggle), `lfo_amt1`, `lfo_amt2`

**LFO Destinations**
- `lfo_osc1` (toggle), `lfo_osc2` (toggle), `lfo_filter` (toggle), `lfo_pw1` (toggle), `lfo_pw2` (toggle)

**Pitch Mod**
- `env_pitch`, `env_pitch_both` (toggle), `bend_range` (toggle), `bend_osc2` (toggle), `vibrato`

**Voice Variation** (per-voice analog drift)
- `filter_var`, `porta_var`, `env_var`, `level_var`, `pan_1` … `pan_8` (pan: 0=L, 0.5=center, 1=R)

**Other**
- `octave_transpose` (plugin-level, -3 to +3 octaves)

Toggle parameters use PARAM_TYPE_INT and display as on/off. Continuous parameters use PARAM_TYPE_FLOAT (0.0-1.0).

### Remote UI (`src/web_ui.html`)

Self-contained OB-Xd-skinned web panel, auto-discovered by schwung-manager and loaded in a
sandboxed iframe on `http://move.local:7700/remote-ui` when OB-Xd is the synth of a slot.
Uses the `schwungRemote` postMessage API (`/static/schwung-remote-api.js`): `getParam`,
`setParam`, `onParamChange`, `getHierarchy`, `getChainParams`. Param keys are component-
prefixed (`synth:cutoff`). Knobs/switches/steppers are defined declaratively in the `PANELS`
array; a fallback shim lets the page render standalone for local preview. The build script
copies `web_ui.html` into the dist tarball.

### Voice Management

6-voice polyphonic (balanced for Move's ARM CPU). Voice allocation with note stealing.

## Signal Chain Integration

Module declares `"chainable": true` and `"component_type": "sound_generator"` in module.json. Chain presets installed to main repo's `modules/chain/patches/` by install script.

## License

GPL-3.0 (inherited from OB-Xd)
