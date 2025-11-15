---
description: "Garmin Forerunner 945 LTE – Development Profile"
applyTo: "**/*.mc"
device: "fr945lte"
---

# Garmin FR945 LTE Development Profile

## Hardware & Display

* **Screen:** 240×240 px, round, Memory-In-Pixel (64 colors)
* **UI Guidance:** High-contrast, round-safe layouts; update at 1 Hz or less to save battery

## Button Layout (with Locations)

| Button         | Location    | Recommended Use                  | Simulator Key |
| -------------- | ----------- | -------------------------------- | ------------- |
| **LIGHT**      | Upper-left  | Backlight / controls menu        | Ctrl + L      |
| **UP**         | Mid-left    | Navigate up / scroll             | ↑             |
| **DOWN**       | Lower-left  | Navigate down / scroll           | ↓             |
| **START/STOP** | Upper-right | Confirm / Start / Pause / Resume | Enter         |
| **BACK/LAP**   | Lower-right | Cancel / Exit / Lap              | Esc           |

> Suggested mappings: START = Start/Pause/Resume, BACK = Exit, UP/DOWN = Navigate, long-press START = Menu (Reorder / Delete / Quick Add), quick-minute addition via dedicated action/menu.

## App Setup

* **App type:** `watch-app`
* **Manifest essentials:**

```xml
<iq:application type="watch-app" minSdkVersion="3.2.0">
  <iq:products><iq:product id="fr945lte"/></iq:products>
</iq:application>
```

* Request **only needed permissions** (e.g. `background`, `communications`, `sensor`).

## Coding Patterns

* **Entry:** `Application.AppBase` → implement `onStart()`, `getInitialView()`, `onStop()`
* **Views:** subclass `WatchUi.View`, override `onUpdate(dc)` & `onKey(key,state)`
* Use **1 Hz Timer** + `Time.now()` for elapsed calculations
* Trigger haptics with `Attention.vibrate()`
* Persist state with `Application.Properties` or `Storage`
* Feature-gate optional APIs (extended keys) with `try/catch`

## Build & Signing

* Generate a **developer key** once (`monkeyc --create-key` or VS Code > Generate Developer Key)
* Pass key to builds (`-y key.der`); required for sideload + store submission
* Test in simulator: `monkeydo myapp.prg fr945lte`

## QA Checklist

* [ ] UI fits 240×240 round display
* [ ] Buttons mapped to Garmin conventions
* [ ] Single timer tick, minimal redraws
* [ ] Haptics throttled (no spam)
* [ ] Minimal permissions in manifest
* [ ] Runs on simulator & sideloaded watch

## Code Skeleton Template

### Basic App Structure

```monkeyc
using Toybox.Application as App;
using Toybox.WatchUi;

class MyApp extends App.AppBase {
    function initialize() {
        AppBase.initialize();
    }

    function onStart(state as Lang.Dictionary?) as Void {
        // App startup logic
    }

    function getInitialView() as [WatchUi.Views] or [WatchUi.Views, WatchUi.InputDelegates] {
        return [new MyView(), new MyDelegate()];
    }

    function onStop(state as Lang.Dictionary?) as Void {
        // App shutdown logic
    }
}

class MyView extends WatchUi.View {
    function initialize() {
        View.initialize();
    }

    function onUpdate(dc as Toybox.Graphics.Dc) as Void {
        // Clear screen
        dc.setColor(Graphics.COLOR_BLACK, Graphics.COLOR_BLACK);
        dc.clear();
        
        // Draw content (240x240 round display)
        dc.setColor(Graphics.COLOR_WHITE, Graphics.COLOR_TRANSPARENT);
        dc.drawText(120, 120, Graphics.FONT_LARGE, "Hello FR945!", Graphics.TEXT_JUSTIFY_CENTER | Graphics.TEXT_JUSTIFY_VCENTER);
    }
}

class MyDelegate extends WatchUi.BehaviorDelegate {
    private var _view as MyView;

    function initialize(view as MyView) {
        BehaviorDelegate.initialize();
        _view = view;
    }

    function onKey(keyEvent as WatchUi.KeyEvent) as Lang.Boolean {
        var key = keyEvent.getKey();
        
        switch (key) {
            case WatchUi.KEY_UP:        // Mid-left button
                // Handle up navigation
                return true;
                
            case WatchUi.KEY_DOWN:      // Lower-left button  
                // Handle down navigation
                return true;
                
            case WatchUi.KEY_ENTER:     // Upper-right (START/STOP)
                // Handle confirm/start action
                return true;
                
            case WatchUi.KEY_ESC:       // Lower-right (BACK/LAP)
                // Handle exit/back action
                WatchUi.popView(WatchUi.SLIDE_IMMEDIATE);
                return true;
        }
        
        return BehaviorDelegate.onKey(keyEvent);
    }
}
```

### Timer-Based Updates (1 Hz Pattern)

```monkeyc
using Toybox.Timer;

class TimedView extends WatchUi.View {
    private var _timer as Timer.Timer?;
    
    function initialize() {
        View.initialize();
        _timer = new Timer.Timer();
        _timer.start(method(:onTimer), 1000, true); // 1 Hz
    }
    
    function onTimer() as Void {
        WatchUi.requestUpdate(); // Trigger onUpdate()
    }
    
    function onHide() as Void {
        if (_timer != null) {
            _timer.stop();
        }
    }
}
```

---

**Perfect for AI context** - provides button mappings, display specs, coding patterns, and ready-to-use templates for FR945 LTE development.