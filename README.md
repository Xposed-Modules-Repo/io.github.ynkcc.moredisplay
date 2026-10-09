# MoreDisplay

MoreDisplay is an Xposed/LSPosed module that runs **only inside `system_server`**. It manages
virtual displays from the system side and lets you decide, per app, which displays that app can
see, which display it records, and which display its accessibility gestures land on.

It never hooks the target app. Every decision and rewrite happens in `system_server`, so the
module scope stays a single package.

## Features

| Feature | What it does |
| --- | --- |
| Managed displays | Create / list / remove virtual displays from `system_server` (uid 1000), including flags such as `PUBLIC`, `TRUSTED`, `OWN_FOCUS`. |
| Launch bypass | Relaxes `ActivityTaskSupervisor#isCallerAllowedToLaunchOnDisplay` so apps can launch themselves onto a managed display. |
| Per-uid visibility | Hide a display from a specific app (blacklist) or show it only to that app (whitelist). Covers the list API, the by-id lookup API and display add/remove events. |
| Recording redirection | When a target app starts a `MediaProjection` capture, the virtual display's mirror source is switched from display 0 to a managed display. |
| Accessibility redirection | `dispatchGesture` calls from a configured accessibility service are rewritten so the gesture is injected into a managed display instead of the default one. |

## Requirements

- LSPosed 1.9.3+ (LibXposed API 102).
- Android 10 (API 29) or above. Developed and verified on Android 16 (API 36).
- Recommended scope: **`system`** (system server) only. Nothing else needs to be checked.

## How it works

The app talks to the in-system-server engine over a channel that needs no new SELinux rules and no
root: the module hooks `ContentProvider$Transport#call` and answers only the method
`moredisplay.rpc`; every other call is passed through untouched. Everything else is plain reflection
against `DisplayManagerService`, matched dynamically by method name and parameter shape so ROM
differences degrade to a log line instead of a boot loop.

## Usage

Enable the module in LSPosed, reboot, then either use the app UI or drive it over adb:

```sh
# create a managed display (flags=1 -> PUBLIC)
adb shell am start -n io.github.ynkcc.moredisplay/.CommandActivity \
  --es action create --ei width 1280 --ei height 720 --ei dpi 240 --ei flags 1

# list / remove
adb shell am start -n io.github.ynkcc.moredisplay/.CommandActivity --es action list
adb shell am start -n io.github.ynkcc.moredisplay/.CommandActivity --es action remove --ei display_id 2

# hide display 2 from uid 10123
adb shell am start -n io.github.ynkcc.moredisplay/.CommandActivity \
  --es action policy-set --ei uid 10123 --es mode hide --eia display_ids 2

# send that app's recordings / gestures to display 2
adb shell am start -n io.github.ynkcc.moredisplay/.CommandActivity \
  --es action policy-set --ei uid 10123 --es mode all --ei record_display_id 2

adb shell am start -n io.github.ynkcc.moredisplay/.CommandActivity --es action policy-list
```

## Notes and limitations

- Display 0 is never filtered; hiding it breaks the target app's `Context` creation.
- Policies are keyed by uid (package name is display-only). On multi-user devices compute
  `uid = userId * 100000 + appId`.
- The engine lives in `system_server` memory, so policies take effect without a reboot, but a
  client process may still hold a cached `DisplayInfo` until it restarts.
- Recording redirection only rewrites the mirror source of displays that really mirror display 0
  with `AUTO_MIRROR`; `OWN_CONTENT_ONLY` virtual displays are left alone.
- Accessibility redirection only rewrites gestures aimed at the default display.

## Links

- Source: https://github.com/ynkcc/MoreDisplay
