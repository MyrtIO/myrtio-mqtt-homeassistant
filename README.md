# myrtio-mqtt-homeassistant

A `#![no_std]` Home Assistant MQTT integration for embedded systems. It uses MQTT Discovery to automatically register devices and entities in Home Assistant.

This crate is designed to work seamlessly with the `myrtio-mqtt` modular runtime.

## Architecture

The crate is structured into three layers:

1.  **Domain Layer** (`device`, `entity`): Platform-independent DTOs representing physical devices and their entities (lights, numbers, etc.).
2.  **HA Wire Layer** (`ha`): Internal JSON-serializable structures that match Home Assistant's specific MQTT Discovery schema.
3.  **MQTT Integration** (`HaModule`): An implementation of `MqttModule` for `myrtio-mqtt` that handles discovery announcements, state publishing, and command routing.

## Supported Entities

-   **Lights**: Supports On/Off, Brightness, Color Temperature (Kelvin), RGB Color, and Effects.
-   **Numbers**: Supports numeric values with range (min/max), step, unit of measurement, and display modes (slider, box).

## Topic Conventions

The crate follows these standard topic formats:

-   **Discovery Config**: `homeassistant/{component}/{device_id}_{entity_id}/config`
-   **State**: `{device_id}/{entity_id}`
-   **Command**: `{device_id}/{entity_id}/set`

## How it Works

`HaModule` manages your entities and coordinates with the `myrtio-mqtt` runtime:

-   **Registration**: During startup, it registers all command topics so the runtime can route incoming MQTT messages to the module.
-   **Discovery**: It automatically publishes discovery payloads (QoS 1) on startup and periodically to ensure Home Assistant remains aware of the device.
-   **State**: It publishes the current state (QoS 0) of all entities periodically or immediately after a command is processed.
-   **Callbacks**: You provide plain function pointers for state polling and command handling, making it easy to integrate with static/global hardware state.

## Quickstart

```rust
use myrtio_mqtt_homeassistant::{
    Device, LightEntity, LightRegistration, LightState, HaModule, 
    ColorMode, LightCommand
};
use myrtio_mqtt::runtime::MqttRuntime;
use embassy_time::Duration;

// 1. Define your device
const DEVICE: Device = Device::new("kitchen_light", "Kitchen Main Light")
    .with_manufacturer("Myrtio")
    .with_model("Light Controller v1");

// 2. Define your entity
const LIGHT: LightEntity = LightEntity::new("main", "Main Light", &DEVICE)
    .with_brightness(true)
    .with_color_modes(&[ColorMode::Rgb, ColorMode::ColorTemp]);

// 3. Define callbacks
fn get_light_state() -> LightState {
    // Return current hardware state
    LightState::on().with_brightness(255)
}

fn handle_command(cmd: &LightCommand) {
    // Update hardware based on command
}

// 4. Create and configure the module
// Generic parameters: <'a, MAX_LIGHTS, MAX_NUMBERS, INTERNAL_BUF_SIZE>
let mut ha_module: HaModule<'_, 1, 0, 1024> = HaModule::new(Duration::from_secs(60));
ha_module.add_light(LightRegistration {
    entity: LIGHT,
    provide_state: get_light_state,
    on_command: handle_command,
}).expect("Failed to add entity");

// 5. Run with MqttRuntime (from myrtio-mqtt)
// let mut runtime = MqttRuntime::new(client, ha_module, publisher_rx);
// runtime.run().await?;
```

## API Map

| Category | Main Types |
|----------|------------|
| **Core** | `Device`, `HaModule`, `Error` |
| **Lights** | `LightEntity`, `LightState`, `LightCommand`, `LightRegistration`, `ColorMode`, `RgbColor` |
| **Numbers** | `NumberEntity`, `NumberRegistration` |
| **HA Wire** | `HaLightDiscovery`, `HaNumberDiscovery`, `HaDeviceInfo` (Re-exported for advanced use) |

## Implementation Notes

-   **Memory**: No dynamic allocation. The `HaModule` requires an internal buffer size (`BUF_SIZE`) large enough to hold the generated JSON discovery payloads.
-   **QoS**: Discovery messages are sent with `AtLeastOnce` (QoS 1) to ensure registration. State updates use `AtMostOnce` (QoS 0) for efficiency.
-   **Error Handling**: If serialization fails (e.g., if `BUF_SIZE` is too small), the operation is skipped without panicking.

