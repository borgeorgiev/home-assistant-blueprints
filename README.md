# home-assistant-blueprints

## SONOFF TRVZB external temperature sync for ZHA

Use one automation per SONOFF TRVZB Gen 1 radiator valve. The blueprint copies a compatible Celsius room sensor, such as the SNZB-02D, into the valve's ZHA **External temperature sensor value** number. It sends an update on temperature changes, when the entities become ready, at Home Assistant startup if ready, and every 60 minutes by default even if the reading stays the same.

### Install and configure

1. Use Home Assistant 2024.10 or later with ZHA. On the valve's device page, verify that **External temperature sensor** (switch) and **External temperature sensor value** (number) are enabled entities. Set the room sensor and number entity to °C. Confirm the number shows a numeric state and enable the external sensor switch manually. Availability of these entities and support for writing them depend on the installed ZHA quirk and valve firmware. [ZHA TRVZB quirk](https://github.com/zigpy/zha-device-handlers/blob/dev/zhaquirks/sonoff/trvzb.py)
2. In **Settings → Automations & scenes → Blueprints → Import blueprint**, use the GitHub URL of the [blueprint file](sonoff_trvzb_zha_external_temperature_sync.yaml) once published. For local installation, copy the file to `blueprints/automation/sonoff/sonoff_trvzb_zha_external_temperature_sync.yaml` under your Home Assistant configuration directory and reload automations. [Home Assistant blueprint installation](https://www.home-assistant.io/docs/automation/using_blueprints/)
3. Create one automation from the blueprint per valve. Select the room sensor, the valve's climate entity, and the switch and number belonging to that same valve. Optionally select a window contact whose `off` state means closed. Leave the existing window automations in control of HVAC mode. Disable any older external-temperature sync automation for this valve before enabling the new one.
4. Choose a refresh interval from 1 to 90 minutes and, optionally, an offset from 0 to 59. The default offset is 7, preserving the original hourly schedule in time zones with whole-hour UTC offsets. Regular writes use elapsed UTC minutes and an offset modulo the interval, so cadence survives clock changes and hour boundaries. Choose offsets with different remainders for valves on the same interval. Event-driven writes can still coincide.

For an existing automation, you can overwrite the previously installed blueprint **at its existing Home Assistant path** and reload automations. All existing input keys are supported, including `hourly_refresh_minute` for the offset. If you install at the new path, update the automation's `use_blueprint.path` or recreate the automation and disable the old instance.

### Example `automations.yaml` entries

These use illustrative entity IDs. Replace them with the entities from your installation and adjust the blueprint path if Home Assistant imported it under a different directory.

```yaml
- id: sonoff_trvzb_entrance_external_temperature
  alias: Entrance radiator external temperature
  use_blueprint:
    path: sonoff/sonoff_trvzb_zha_external_temperature_sync.yaml
    input:
      room_temperature: sensor.entrance_room_temperature
      trv_climate: climate.entrance_radiator_thermostat
      trv_external_sensor_switch: switch.entrance_radiator_external_temperature_sensor
      trv_external_temperature: number.entrance_radiator_external_temperature_sensor_value
      optional_window_contact: binary_sensor.entrance_skylight
      refresh_interval_minutes: 60
      hourly_refresh_minute: 7

- id: sonoff_trvzb_corridor_external_temperature
  alias: Corridor radiator external temperature
  use_blueprint:
    path: sonoff/sonoff_trvzb_zha_external_temperature_sync.yaml
    input:
      room_temperature: sensor.corridor_room_temperature
      trv_climate: climate.corridor_radiator_thermostat
      trv_external_sensor_switch: switch.corridor_radiator_external_temperature_sensor
      trv_external_temperature: number.corridor_radiator_external_temperature_sensor_value
      refresh_interval_minutes: 60
      hourly_refresh_minute: 37
```

### Test on one valve

1. **Normal operation.** With the valve in `heat`, the external sensor switch on, and any selected contact closed, wait for a room temperature change. In the automation trace, check the `number.set_value` action and compare the valve's reported current temperature with the room sensor. With a stable room reading, check that a scheduled write still dispatches.
2. **Fallback and keepalive.** In a supervised test with the valve in `heat`, pause external-temperature sync and observe the valve's reported current temperature and HVAC action for just over two hours from the last applied update. Resume sync, wait for the next scheduled write, and verify that the valve again tracks room temperature. Then observe for over two hours with hourly sync enabled and check the traces and valve history for a fallback.
3. **Manual off and window.** Switch the valve to `off`, then wait for a source change and a scheduled refresh. Confirm no dispatch and that the valve remains off. Return it to `heat`, open the configured contact, and check that dispatch stops even before the window automation switches the valve off. Close the contact and verify sync resumes once the contact is closed and the valve is in `heat`. If the window automation restores `off`, confirm the valve stays off.
4. **Startup and reconnect.** Restart Home Assistant while the valve is in `heat` and the sensor switch is on. Expect a startup write if everything is available, or a write when the final required entity becomes ready. Repeat with HVAC `off` and confirm no write. If an entity goes unavailable and later recovers, check the readiness trace.
5. **Second instance.** Verify scheduled actions occur at the two different offsets and each instance targets its own number.

### Observed behavior and limits

- In the supplied entrance history, the valve reported 21.2 °C at 03:46:08 CET and 25.5 °C at 05:46:18 CET on 17 November 2026, while the room sensor stayed at 21.2 °C. This is consistent with a roughly two-hour fallback on that valve. The reported same-value recovery and off-mode reactivation were observations from the installation. Whether and when they occur depends on firmware and device behavior.
- Home Assistant checks its live state immediately before dispatch. A concurrent physical or automation mode change can occur between the check and Zigbee delivery. A dispatched command cannot be recalled. The configured window contact adds protection while a separate window automation is still waiting to switch HVAC off.
- Each automation serializes writes and permits one waiting run. Under a burst, additional triggers may be dropped. Waiting runs re-read the current temperature and safety states. Failed Zigbee writes can be retried by subsequent events or scheduled refreshes. A trace or updated number state alone cannot verify that the valve is using the reading.
- A numeric state can remain cached when a sensor stops reporting. This blueprint relies on Home Assistant's entity availability and cannot detect every stale reading or missed report. It blocks writes if either temperature entity lacks a numeric Celsius state, or the requested value falls outside the target number's range. A target that stays `unknown` needs investigation in ZHA before the blueprint can sync.

### Changelog

- Added a configurable 60-minute keepalive with a per-valve offset and retained the original input keys.
- Added readiness detection for delayed startup, reconnects, the external sensor switch, HVAC mode changes, and the optional window contact.
- Added target availability, Celsius and range checks. Preserved the final heat, switch and window checks before the only service action, `number.set_value`.
- Replaced restart mode with a bounded queue to avoid canceling a running Zigbee call when a new trigger arrives. The [original blueprint](archive/sonoff_trvzb_zha_safe_external_temperature_sync.original.yaml) is archived for reference.
