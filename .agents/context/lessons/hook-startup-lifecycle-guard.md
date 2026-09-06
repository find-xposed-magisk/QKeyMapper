---
name: hook-startup-lifecycle-guard
description: Global Windows hook thread receiving OS input before main GUI window finishes construction causes null pointer dereference crashes; fix with atomic lifecycle guard and pass-through early returns.
metadata:
  type: pattern
---

# Global Windows Hook Startup Lifecycle Guard Pattern

**Pattern**: In multithreaded desktop apps that register global OS hooks (`WH_MOUSE_LL`, `WH_KEYBOARD_LL`, or input filter drivers like Interception), background threads may start receiving OS input events before the main GUI window finishes construction and assigns its singleton pointer (`m_instance`), causing sporadic null pointer dereference crashes (e.g. `0xC0000005` reading `[rcx+0x30]`).

**Root cause**:
1. Global hooks installed via `SetWindowsHookEx` start receiving OS-wide input events immediately.
2. If the hook thread is started prior to or concurrently with the main window's constructor, any user input (mouse move, click, keystroke) dispatches to the hook procedure on the worker thread.
3. Hook procedures or hotkey detection functions query the main window singleton (`QKeyMapper::getInstance()`).
4. During window construction, the singleton is either still `nullptr` or points to an object that has not completed field initialization, resulting in immediate access violations.

**Detection signal**:
- WinDbg dump shows `0xC0000005 (Access violation)` reading low offset address like `0x0000000000000030` (`NullClassPtr Read`).
- Faulting thread is the hook/worker thread, NOT the main GUI thread.
- Faulting instruction is typically accessing a member of the main window singleton (e.g. `cmp dword ptr [rcx+30h], 0` where `rcx` is 0).
- Bug is sporadic: occurs only when user touches mouse or keyboard within the ~50ms window right after double-clicking to launch the app.

## The Solution: Two-tier Lifecycle Guard (Zero-risk to Thread Ordering)

Do NOT reorder thread startup if threads have intricate interactions with drivers or COM. Instead, use an atomic lifecycle state guard:

### 1. Atomic Lifecycle Flag in Main Window
```cpp
// Header
static QAtomicInt s_AtomicIsInitialized;
static bool isInitialized() {
    return (m_instance != Q_NULLPTR) && (QKEYMAPPER_ATOMIC_LOAD_RELAXED(s_AtomicIsInitialized) != 0) && (!s_isDestructing);
}

// Constructor (last line after all UI, tables, configs loaded)
QKEYMAPPER_ATOMIC_STORE_RELAXED(s_AtomicIsInitialized, 1);

// Destructor (first line)
QKEYMAPPER_ATOMIC_STORE_RELAXED(s_AtomicIsInitialized, 0);
```

### 2. Immediate Pass-Through Early Return in Hook Procedures
```cpp
LRESULT CALLBACK LowLevelMouseHookProc(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode != HC_ACTION) return CallNextHookEx(Q_NULLPTR, nCode, wParam, lParam);
    if (!QKeyMapper::isInitialized()) {
        return CallNextHookEx(Q_NULLPTR, nCode, wParam, lParam); // Transparent pass-through
    }
    // Normal hook processing...
}
```

### 3. Null-Check Defense in All Hotkey / State Helpers
In functions like `detectMappingSwitchKey`:
```cpp
QKeyMapper *keyMapper = QKeyMapper::getInstance();
if (keyMapper == Q_NULLPTR || !QKeyMapper::isInitialized()) {
    return false;
}
```

**Why**: Reordering thread startup can introduce hidden regressions in driver loops and asynchronous event queues. An atomic lifecycle guard protects both startup and shutdown phases with zero side effects on architecture.