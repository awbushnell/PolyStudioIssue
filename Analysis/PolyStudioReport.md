# Troubleshooting Report: WPF HwndHost Crash — Desktop Heap Exhaustion by HP Poly Studio

**Date:** February 18, 2026
**Affected Application:** Any WPF application using `HwndHost` (reproduced with WPFHostingWin32Control sample; also affects Autodesk Revit)
**External Cause:** HP Poly Studio (Electron/Node.js application)
**Error:** `System.ComponentModel.Win32Exception` — "Not enough quota is available to process this command" (Win32 error 1816)

---

## Symptom

When HP Poly Studio is running, setting `IsEnabled = false` on a WPF `HwndHost` subclass throws a `Win32Exception` with error code 1816. The crash occurs because WPF's internal `HwndHost.OnEnabledChanged` callback calls `EnableWindow` on the native window handle, and the kernel rejects the call due to shared desktop heap exhaustion.

The crash does **not** occur when Poly Studio is not running.

### Managed Stack Trace

```
WPFHostingWin32Control.MainWindow.ToggleListEnabledState(Object, EventArgs)
  → System.Windows.DependencyObject.SetValue(DependencyProperty, Object)
    → System.Windows.DependencyObject.SetValueCommon(...)
      → System.Windows.DependencyObject.UpdateEffectiveValue(...)
        → System.Windows.FrameworkElement.OnPropertyChanged(...)
          → System.Windows.UIElement.OnIsEnabledChanged(...)
            → System.Windows.Interop.HwndHost.OnEnabledChanged(...)
              → MS.Win32.UnsafeNativeMethods.EnableWindow(HandleRef, Boolean)
```

---

## Root Cause

### Summary

HP Poly Studio registers **15 global WinEvent hooks** (via `SetWinEventHook`) targeting **all processes** on the desktop, including **7 redundant hooks** for `EVENT_OBJECT_LOCATIONCHANGE` — the highest-frequency accessibility event in Windows. Its Node.js/V8 runtime then enters **7-8 second JavaScript execution pauses** without pumping Win32 messages. During these pauses, the Windows kernel queues thousands of pending WinEvent callback notifications in the **shared desktop heap**, exhausting it and causing any process that needs a kernel USER object allocation to fail with error 1816.

### Detailed Mechanism

#### 1. Poly Studio Registers Excessive Global Hooks

Poly Studio (`Poly Studio.exe`) is an Electron/Chromium application built on Node.js. Upon startup, it registers 15 `WINEVENT_OUTOFCONTEXT` hooks with `idProcess = 0` (all processes):

| Event ID(s) | Event Name | Hook Count |
|-------------|-----------|:----------:|
| `0x3` | `EVENT_SYSTEM_FOREGROUND` | 1 |
| `0x9` | `EVENT_SYSTEM_CAPTUREEND` | 1 |
| `0xa`–`0xb` | `EVENT_SYSTEM_MOVESIZESTART/END` | 1 |
| `0x16`–`0x17` | `EVENT_SYSTEM_MINIMIZESTART/END` | 1 |
| `0x8002`–`0x8003` | `EVENT_OBJECT_SHOW/HIDE` | 1 |
| `0x800a` | `EVENT_OBJECT_STATECHANGE` | 1 |
| `0x8017`–`0x8018` | `EVENT_OBJECT_CLOAKED/UNCLOAKED` | 1 |
| **`0x800b`** | **`EVENT_OBJECT_LOCATIONCHANGE`** | **7** |

The 7 duplicate `EVENT_OBJECT_LOCATIONCHANGE` hooks are almost certainly a bug — one hook would suffice. This event fires for **every** window move, resize, scroll, animation, tooltip, and compositor update across the entire desktop.

The hooks are installed from a timer callback inside Node.js's `uv_run` event loop:

```
USER32!SetWinEventHook
  Poly_Studio!Cr_z_adler32_combine+0xa1f     (native addon)
    Poly_Studio!uv_loop_fork+0x29aa
      Poly_Studio!uv_timer_get_due_in+0x5f9
        Poly_Studio!uv_run+0x596              (Node.js event loop)
```

#### 2. Node.js Fails to Pump Win32 Messages

For `WINEVENT_OUTOFCONTEXT` hooks, the Windows kernel delivers event callbacks to the hook owner's thread only when that thread is in a waitable state (calling `PeekMessage`, `GetMessage`, or `MsgWaitForMultipleObjects`). If the thread is busy, the kernel **queues** the pending callbacks in the **shared desktop heap**.

Poly Studio's Node.js event loop uses `PeekMessage` (non-blocking) to integrate with Win32, but has **7-8 second gaps** where V8 executes JavaScript without calling `PeekMessage`:

| Time (seconds) | PeekMessage calls | Status |
|:--------------|:-----------------:|--------|
| :30–:31 | 5 | Hooks just installed |
| **:32–:38** | **1 in 7 seconds** | **V8 executing JavaScript** |
| :39–:40 | 53 | Brief resume |
| **:44–:49** | **0 for 6 seconds** | **V8 executing JavaScript** |
| :54–:56 | 76 | Resume |
| :57–:59 | 326 | Callback flood on drain |

#### 3. Desktop Heap Fills Up During Pump Gaps

During a typical 8-second non-pumping gap:

- `EVENT_OBJECT_LOCATIONCHANGE` fires ~75+ times/second on a busy desktop
- With **7 duplicate hooks**, each event generates **7 queued kernel callbacks**
- Over 8 seconds: **~4,200 pending callbacks** queued in the shared desktop heap
- Each pending callback structure: ~200–400 bytes
- **Total: ~840 KB – 1.7 MB** — exceeding the default 768 KB desktop heap

#### 4. WPF App Crashes During Exhaustion Window

When the user clicks a button in the WPF app while the desktop heap is full:

```
[Poly Studio]  V8 executing JavaScript — no PeekMessage for ~8 sec
       ↓
[Kernel]       Queues ~4,200 WinEvent callbacks in shared desktop heap
       ↓
[Desktop Heap] FULL — no contiguous blocks available
       ↓
[WPF App]      User clicks "Toggle List Enabled"
       ↓
[WPF App]      _listControl.IsEnabled = false
       ↓
[WPF Runtime]  HwndHost.OnEnabledChanged → EnableWindow(hwnd, FALSE)
       ↓
[Kernel]       NtUserEnableWindow → allocation fails → error 1816
       ↓
[WPF App]      Win32Exception: "Not enough quota is available"
```

### Why It's Intermittent

The crash only occurs if the button is clicked **during one of Poly Studio's non-pumping windows** (~7-8 seconds). When Node.js briefly pumps messages (~1-3 seconds), the callback queue drains and the desktop heap recovers. This creates a repeating cycle of ~8 seconds "crash window" followed by ~2 seconds "safe window."

---

## Evidence

### TTD Trace Analysis — Poly Studio Process

| Metric | Value |
|--------|-------|
| Executable | `C:\Program Files\Poly\Poly Studio\Poly Studio.exe` |
| Framework | Electron/Chromium (Node.js, V8, libuv) |
| Global WinEvent hooks | 15 (7× LOCATIONCHANGE) |
| Hook flags | `WINEVENT_OUTOFCONTEXT`, `idProcess=0` |
| WinEvent callbacks received | 448 in 18 seconds (~25/sec) |
| Max message pump gap | **8.4 seconds** |
| Error 1816 in Poly Studio itself | 150 occurrences |
| `SetWindowsHookEx` (system-wide hooks) | 0 |

### TTD Trace Analysis — WPF Host App

| Metric | Value |
|--------|-------|
| Executable | `WPFHostingWin32Control.exe` (.NET 10) |
| Total error 1816 occurrences | **1,327** |
| First error 1816 | **31 ms** after first window creation |
| Error pattern | Bursty — clusters separated by 5–20 sec gaps |
| Kernel `NtUserCreateWindowEx` failures | 2 (retried successfully by user32) |
| `EnableWindow` crash | 21:24:09.545 — during fresh exhaustion burst |
| TEB `LastErrorValue` at crash | `0x718` (error 1816) |
| WPF app's own USER object footprint | 16 windows, 0 hooks, 0 menus — negligible |

### Cross-Trace Correlation

The bursty error pattern in the WPF trace (clusters of errors separated by multi-second quiet periods) matches the pump cycle observed in the Poly Studio trace (JavaScript execution gaps followed by brief message pump bursts). Both traces show error 1816 originating from the same kernel-level desktop heap allocation failures.

---

## Workaround — System Level


### Restart or Disable Poly Studio

- Restart Poly Studio before launching Revit or WPF applications.
- Disable Poly Studio auto-start if it's not needed for active calls.
- Configure Poly Studio to only run during meetings.

---

## Recommendations for HP/Poly

1. **Fix duplicate hooks.** Seven separate `SetWinEventHook` calls for `EVENT_OBJECT_LOCATIONCHANGE` (0x800b) is unnecessary. A single hook suffices. Each duplicate multiplies the kernel's callback dispatch work and desktop heap consumption.

2. **Pump Win32 messages more frequently.** The Node.js event loop must call `PeekMessage` or `MsgWaitForMultipleObjects` at least every 100–200 ms when global WinEvent hooks are active. An 8-second gap without pumping causes thousands of kernel callbacks to queue in the shared desktop heap, affecting every process in the window station.

3. **Consider `WINEVENT_INCONTEXT` or scoped hooks.** If Poly Studio only needs to monitor specific windows or its own process, use `idProcess` to scope the hooks and reduce system-wide impact.
