# Restart-proof auto-off (Home Assistant blueprint)

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FHalEEfacts%2Fhome-assistant-blueprints%2Fblob%2Fmain%2Frestart_proof_auto_off%2Frestart_proof_auto_off.yaml)

Turns a device off after a set time, **and keeps counting through restarts and updates.**

## Why

The usual way to do "turn off after 4 hours" is:

```yaml
- action: switch.turn_on
- delay: "04:00:00"
- action: switch.turn_off
```

When Home Assistant restarts (an update, a power blip, a config reload), every running `delay` and `wait` is cancelled. The "turn off" step never runs. In my case an electric blanket stayed on for 28 hours.

This blueprint uses a **timer helper with Restore turned on** instead. Timers like that pick up where they left off after a restart, and the blueprint catches up on anything that ran out while Home Assistant was down.

## What it does

| When | What happens |
|---|---|
| The device is turned on any way (app, wall button, voice, another automation) | Starts the timer with your default time |
| You start the timer yourself with a `duration` | Uses that time instead. If the device is off, it turns on |
| The timer runs out | Turns the device off (up to 3 tries, 15 s apart) and writes a logbook line. Optional actions if it still won't turn off |
| The device is turned off early | Cancels the timer |
| Home Assistant starts | Waits 1 minute, then: device on with no timer running → turns it off (or starts a fresh timer, your choice). Device off with a timer running → cancels it. Timer missing "Restore" → writes a warning to the log |

## Setup

1. **Create a timer helper** for the device: Settings → Devices & services → Helpers → Create helper → Timer.
   - Tick **"Restore state and time when Home Assistant starts"**. This is the important part.
   - The duration you set on the helper doesn't matter. The blueprint uses its own "Turn off after" setting.
2. **Import the blueprint** and create an automation from it: pick the device, the timer and the default time.
3. Use one timer helper and one automation per device.

## Turning it on for a custom time

Start the timer instead of turning on the device:

```yaml
action: timer.start
target:
  entity_id: timer.auto_off_space_heater
data:
  duration: "00:45:00"
```

If "Starting the timer turns the device on" is ticked (the default), the device turns on and goes off 45 minutes later. This works from a dashboard button, a script or another automation.

If the device is already on, `timer.start` with a new duration restarts the countdown from that time.

## Options

- **Turn off after:** the time used whenever the device is turned on normally. Default 1 hour.
- **Starting the timer turns the device on:** default on.
- **At startup, turn off a device that has no timer running:** default on (the safe choice). After a restart, a device that's on with nothing counting down most likely ran out of time while Home Assistant was down. Untick this if people often switch the device on at the wall while Home Assistant is offline; it will then get a fresh timer instead.
- **If it won't turn off:** any actions you like, for example a phone notification.

## Supported devices

Simple on/off entities: `switch`, `light`, `fan`, `input_boolean`, `humidifier`, `siren`.

## Why the startup check matters

A restored timer whose time ran out during the restart does fire `timer.finished`, but it fires before automations have loaded, so nothing hears it. In testing, that event went out more than a minute before the automation's start trigger. That's why the startup check looks at the device and timer directly instead of waiting for the event.

## Tested

On Home Assistant 2026.11 with a smart plug: normal turn-on and timeout, custom time via `timer.start`, turned off early, and a real restart while the timer was running. The plug turned off 60 s after startup.

## Limits

- Blueprints can't create helpers, so you have to make the timer yourself.
- If a device is turned on *while Home Assistant is down*, there's no way to tell that apart from "its time ran out while down". The startup option decides which way to go.
- It needs Home Assistant 2024.10 or newer.
