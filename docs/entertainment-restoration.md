# After-sync lighting preferences

The Hue app saves the entertainment area's after-sync preference in an enabled
`behavior_instance` with script ID `7719b841-6b3d-448d-a0e7-601ae9edb6a2`.
The group's `where` reference uses the V2 entertainment-configuration ID.

The entertainment worker copies each participating light's state before changing
its on/off, brightness or colour values. This snapshot lives only in that worker;
it is not written to configuration files. Both V1 and V2 starts use this worker.
Repeated start requests for an already active area do not replace its snapshot.

When the worker stops (API stop, stream EOF, or an exception), it releases its
relay and streaming mode, reads the current saved preference, and sends ordinary
light commands. The existing process ownership check prevents an obsolete worker
from restoring over a replacement stream.

| App option | Saved configuration | Result |
| --- | --- | --- |
| Pause on last light | `end_state: do_nothing` | No restoration command; keep the final streaming state. |
| Off | `end_state: off` | Send OFF to each participating light. |
| Last on state | `end_state: last_state` | Restore the pre-sync brightness and active colour mode for lights that were on; turn lights that were off back off. |
| Bright, Dimmed, Nightlight, Relax, Read, Concentrate, Energize | `end_state: scene`, `end_scene.rtype: recipe` | Apply the corresponding fixed preset. |

The app can retain a stale `end_scene` reference after selecting Off, Last on
state or Pause. It is only interpreted for `end_state: scene`. Missing/disabled
preferences and unknown choices keep the current output; unknown recipes are
logged rather than replaced with a different preset.

Snapshots preserve active CT, XY or hue/saturation colour modes and stored
gradient points. Off snapshots also restore cached colour and brightness without
sending commands that might inadvertently turn an off light on. Transient state
such as reachability and streaming mode is not replayed.

## Preset data

The fixed UUIDs are independently recorded in
[Bifrost's shared scene identifiers](https://github.com/chrivers/bifrost/blob/12d9e37e6ea032fb0708ddcd2faaa6db0133d7c8/crates/hue/src/scene_icons.rs).
The brightness and XY values follow
[observed Hue gallery presets](https://gist.github.com/Hypfer/a0a8b5b9429831a7306ec4300077eaaa).
Temperature-only lights use approximate white-temperature equivalents, clamped
to their model's supported range. Dimmable lights receive brightness only;
gradient lights receive a uniform colour for the fixed presets.

These values are not a documented Philips after-sync recipe specification.
The recipe reference for Relax is confirmed by an app-setting capture; the other
six mappings and the after-sync output should still be compared with a real
bridge before claiming exact Philips parity. Preset data is centralised in
`BridgeEmulator/functions/entertainment.py` for refinement.

## Validation

Run the offline tests from the repository root:

```sh
python -m unittest discover -s tests -v
```

Tests use the real light models, restoration helper, entertainment worker and
API handlers. Network/process boundaries are mocked; they do not control lights
or write bridge configuration. The worker test consumes decoded HueStream frames
and checks that frame updates do not corrupt the original snapshot. MQTT and
native multi-light tests inspect the actual driver payloads.

Hardware validation should exercise both an explicit stop and a sender
disconnect, with:

- Initially on lights using XY and CT, and a mixture of initially on/off lights.
- A stored gradient, where the backend supports gradients.
- Each of the seven presets, Off, Last on state and Pause.
- Switching the preference during a stream, then stopping.
- Starting another stream after restoration.

Pause sends no restoration command. Devices with their own realtime timeout or
automatic restoration may choose an output when that mode ends; verify this on
the target backend. Device-native effects/animations and a lamp's physical state
changed outside diyHue are not guaranteed to be captured by its cached state.
