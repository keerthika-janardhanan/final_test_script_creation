# Recorder Internal Configuration - Step Capture Mechanism

## Overview
The recorder uses a sophisticated JavaScript injection system that captures all DOM interactions in real-time. This document explains how every step is captured and what might cause missing steps.

---

## 1. Injection Architecture

### A. Initialization System
```javascript
// The recorder injects this script into every page/frame
PAGE_INJECT_SCRIPT = """
(() => {
    // Prevent double-injection
    if (window.__pyRecInstalled) { return; }
    window.__pyRecInstalled = true;
    
    // Queue system for reliability (handles async delivery)
    const capQ = [];  // Action capture queue
    const ctxQ = [];  // Page context queue
    
    // Retry delivery every 200ms if bindings aren't ready
    setInterval(() => {
        while (capQ.length && deliver('pythonRecorderCapture', capQ[0])) capQ.shift();
        while (ctxQ.length && deliver('pythonRecorderPageContext', ctxQ[0])) ctxQ.shift();
    }, 200);
})();
"""
```

**Key Points:**
- Injected via `context.add_init_script()` - runs BEFORE page loads
- Injected into frames via `frame.add_script_tag()` - handles iframes
- Queue system ensures no events are lost if Python bindings are delayed
- Retry mechanism handles CSP (Content Security Policy) restrictions

---

## 2. Event Capture System

### A. DOM Events Being Captured

```javascript
// CAPTURED EVENTS (with capture: true for early interception)
document.addEventListener('click', handler, true);           // All clicks
document.addEventListener('dblclick', handler, true);        // Double clicks
document.addEventListener('contextmenu', handler, true);     // Right-clicks
document.addEventListener('submit', handler, true);          // Form submissions
document.addEventListener('change', handler, true);          // Input changes (select, checkbox, radio)
document.addEventListener('input', handler, true);           // Text input (real-time typing)
document.addEventListener('keydown', handler, true);         // Special keys (Enter, Tab, Esc, Arrows)
document.addEventListener('keyup', handler, true);           // Key releases
document.addEventListener('wheel', handler, {capture: true, passive: true}); // Scroll events (throttled)
```

**Capture Phase (`true`):**
- Events are intercepted BEFORE they reach the target element
- Prevents event.stopPropagation() from blocking recording
- Essential for complex SPA frameworks (React, Angular, Vue)

### B. Page Context Events

```javascript
// NAVIGATION & LIFECYCLE EVENTS
document.addEventListener('DOMContentLoaded', () => sendCtx('domcontentloaded'));
window.addEventListener('load', () => sendCtx('load'));
window.addEventListener('hashchange', () => sendCtx('hashchange'));

// SPA DETECTION (intercepts history API)
history.pushState = function() {
    const result = originalPushState.apply(this, arguments);
    setTimeout(() => sendCtx('pushstate'), 0);
    return result;
};

history.replaceState = function() {
    const result = originalReplaceState.apply(this, arguments);
    setTimeout(() => sendCtx('replacestate'), 0);
    return result;
};
```

**Why This Matters:**
- Captures SPA route changes (React Router, Vue Router, etc.)
- Tracks page context for multi-page flows
- Detects when UI state changes without full page reload

---

## 3. Element Data Extraction (The "Snapshot")

When an event occurs, the recorder captures a comprehensive snapshot:

```javascript
const snap = raw => {
    const el = normalizeElement(raw);
    return {
        // Basic identification
        tag: el.tagName.toLowerCase(),
        id: el.id || '',
        className: el.className || '',
        
        // Accessibility info (for stable selectors)
        role: roleOf(el),                    // ARIA role or inferred role
        name: accName(el),                   // Accessible name
        ariaLabel: el.getAttribute('aria-label'),
        labels: labelTexts(el),              // <label> associations
        
        // Text content
        title: el.getAttribute('title'),
        placeholder: el.getAttribute('placeholder'),
        text: el.textContent.trim().slice(0,150),
        
        // Attributes (id, name, type, value, class, href, src, etc.)
        attributes: extractAttributes(el),
        dataset: extractDataAttributes(el),  // data-* attributes
        
        // Multiple selector strategies
        xpath: generateXPath(el),            // Full XPath
        cssPath: generateCSSPath(el),        // CSS path with nth-child
        stableSelector: makeStableSelector(el), // Best stable selector
        
        // Playwright-specific locators
        playwright: {
            byRole: { role, name },          // getByRole("button", { name: "Submit" })
            byLabel: labels[0],              // getByLabel("Email")
            byText: el.innerText.trim()      // getByText("Click here")
        },
        
        // Visual context
        rect: el.getBoundingClientRect(),    // Position & size
        styles: getComputedStyle(el),        // CSS properties
        
        // DOM relationships
        ancestors: ancestorChain(el),        // Parent chain with indexes
        relations: {
            parent: parentInfo,
            previousSibling: prevInfo,
            nextSibling: nextInfo,
            siblingIndex: index,
            siblings: siblingsInfo
        },
        
        // Contextual heading (for section identification)
        nearestHeading: findNearestHeading(el)
    };
};
```

---

## 4. Selector Generation Strategy

### Priority Order (for reliable playback):

1. **ID** (if unique): `#login-button`
2. **Data attributes**: `[data-testid="submit"]`, `[data-test="login"]`
3. **Name attribute** (if unique): `[name="username"]`
4. **Playwright role+name**: `getByRole("button", { name: "Submit" })`
5. **Label association**: `getByLabel("Email Address")`
6. **Text content**: `getByText("Click here")`
7. **CSS path**: `div.container > form:nth-child(2) > input.email`
8. **XPath** (fallback): `/html/body/div[1]/form[1]/input[3]`

### Stable Selector Logic:
```javascript
const makeStableSelector = el => {
    // 1. Prefer ID if unique
    const byId = cssId(el);
    if (byId && isUnique(byId)) return byId;
    
    // 2. Prefer data-testid, data-test, data-qa
    const byData = cssData(el);
    if (byData) return byData;
    
    // 3. Prefer name attribute if unique
    const byName = nameAttrSel(el);
    if (byName) return byName;
    
    // 4. Playwright-style role+name
    const r = roleOf(el);
    const n = accName(el);
    if (r && n) return `getByRole("${r}", { name: "${n}" })`;
    
    // 5. Label association
    const labs = labelTexts(el);
    if (labs.length) return `getByLabel("${labs[0]}")`;
    
    // 6. Text content
    const t = el.innerText.trim();
    if (t) return `getByText("${t.slice(0,80)}")`;
    
    // 7. CSS path (structural)
    const cssp = cssPath(el);
    if (cssp) return cssp;
    
    // 8. XPath (last resort)
    return generateXPath(el);
};
```

---

## 5. Sensitive Data Masking

```javascript
const isSensitive = (el) => {
    const idn = (el.name || el.id || '').toLowerCase();
    const type = el.type.toLowerCase();
    
    // Password fields
    if (type === 'password') return true;
    
    // Pattern matching
    return /password|pwd|otp|token|secret|pin/.test(idn);
};

// Usage in change/input events
document.addEventListener('change', e => {
    const target = e.target;
    const masked = isSensitive(target);
    const value = masked ? '<masked>' : target.value;
    send('change', target, { value, valueMasked: masked });
});
```

---

## 6. Communication Pipeline

### Frontend (Browser) → Backend (Python)

```
┌─────────────────────────────────────────┐
│ Browser DOM Event (click, change, etc) │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ JavaScript snap() - Extract all data    │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ send() - Create payload with metadata   │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Queue (capQ) - Buffer for reliability   │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Playwright Binding (every 200ms retry)  │
│ window.pythonRecorderCapture(payload)   │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Python: _enqueue_action(payload)        │
│ Thread-safe queue (deque)               │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Main thread poll: session.add_action()  │
│ Captures artifacts (DOM/screenshot)     │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ metadata.json - Persisted to disk       │
└─────────────────────────────────────────┘
```

### Fallback Mechanism (Console Logging)

If Playwright bindings fail (CSP restrictions, cross-origin iframes):

```javascript
// Primary: Binding
sendCap(payload);

// Fallback: Console log with JSON
console.debug('[recorder-json] ' + JSON.stringify(payload));
```

Python-side detection:
```python
def _on_console_with_fallback(msg: ConsoleMessage):
    text = msg.text
    if text.startswith('[recorder-json]'):
        try:
            payload = json.loads(text[len('[recorder-json]'):])
            payload['extra']['fromConsole'] = True  # Mark as degraded
            session.add_action(payload)
        except:
            pass
```

---

## 7. Common Issues Causing Missing Steps

### Issue 1: CSP (Content Security Policy) Restrictions
**Symptom:** No steps captured at all
**Cause:** Website blocks script injection
**Solution:** Use `--bypass-csp` flag
```bash
python -m app.run_playwright_recorder_v2 --url "https://..." --bypass-csp
```

### Issue 2: Shadow DOM
**Symptom:** Clicks inside web components not captured
**Cause:** Events trapped inside shadow root
**Solution:** Uses `composedPath()` to pierce shadow DOM
```javascript
const targetOf = e => {
    if (e.composedPath) {
        const path = e.composedPath();
        return path[0];  // Get actual target through shadow boundary
    }
    return e.target;
};
```

### Issue 3: Event.stopPropagation() in App Code
**Symptom:** Some clicks/changes not captured
**Cause:** App code prevents event bubbling
**Solution:** Use capture phase (`true` flag)
```javascript
document.addEventListener('click', handler, true);  // Capture BEFORE target
```

### Issue 4: Delayed Bindings
**Symptom:** First few actions missing
**Cause:** Python bindings not ready when page loads
**Solution:** Queue + retry system (already implemented)
```javascript
setInterval(() => {
    while (capQ.length && deliver('pythonRecorderCapture', capQ[0])) 
        capQ.shift();
}, 200);
```

### Issue 5: Fast Navigation
**Symptom:** Actions lost during page transitions
**Cause:** Page unload before actions are sent
**Solution:** Final aggressive drain on shutdown
```python
# Final drain before context close (up to 2s)
end_deadline = time.time() + 2.0
while time.time() < end_deadline:
    while pending_actions:
        action = pending_actions.popleft()
        session.add_action(action)
    time.sleep(0.05)
```

### Issue 6: iframes
**Symptom:** Actions inside iframes not captured
**Cause:** Script not injected into frames
**Solution:** Frame attachment handler
```python
def _on_frame_attached(frame: Frame):
    frame.add_script_tag(content=PAGE_INJECT_SCRIPT)

page.on("frameattached", _on_frame_attached)
```

### Issue 7: Popups/New Windows
**Symptom:** Actions in popups not captured
**Cause:** New page context not instrumented
**Solution:** Popup handler
```python
def _on_popup(popup_page: Page):
    popup_page.add_init_script(PAGE_INJECT_SCRIPT)
    popup_page.on("console", _on_console_with_fallback)
    # ... register handlers

page.on("popup", _on_popup)
context.on("page", _on_popup)  # Also catch window.open()
```

---

## 8. Debugging Missing Steps

### Enable Verbose Logging

1. **Browser Console:**
   - Open DevTools → Console
   - Filter for `[recorder]`
   - Should see: `[recorder] action click button https://...`

2. **Check Binding Status:**
   ```javascript
   // In browser console:
   window.__pyRecInstalled  // Should be true
   typeof window.pythonRecorderCapture  // Should be "function"
   ```

3. **Python-side Logs:**
   ```python
   # Check stderr output for:
   [recorder][console] debug: [recorder] action click ...
   [recorder][requestfailed] ... (network errors)
   [recorder][framenavigated] ... (navigation tracking)
   ```

### Diagnostic Script

Add to `run_playwright_recorder_v2.py`:
```python
# After line 950 (in main recording loop)
if len(pending_actions) > 0 or len(pending_ctx) > 0:
    print(f"[DEBUG] Queue sizes: actions={len(pending_actions)}, ctx={len(pending_ctx)}")
```

### Check Recorded Metadata

```python
# After recording:
import json
with open("recordings/<session>/metadata.json") as f:
    data = json.load(f)
    print(f"Total actions captured: {len(data['actions'])}")
    for action in data['actions']:
        print(f"  - {action['action']}: {action['element']['tag']}")
```

---

## 9. Configuration Options

### Recommended for Maximum Capture:

```bash
python -m app.run_playwright_recorder_v2 \
    --url "https://your-app.com" \
    --session-name "my-test-flow" \
    --browser chromium \
    --capture-dom \              # Capture full HTML at each step
    --capture-screenshots \       # Capture screenshots
    --bypass-csp \               # Bypass Content Security Policy
    --ignore-https-errors \      # Ignore SSL errors
    --timeout 120                # 2 minutes recording time
```

### Performance Trade-offs:

| Option | Impact | When to Use |
|--------|--------|-------------|
| `--capture-dom` | +Large files, +Debugging | Complex apps, need DOM forensics |
| `--capture-screenshots` | +Large files, +Visual validation | Visual regression, debugging |
| `--bypass-csp` | Improved capture | Sites with strict CSP |
| `--slow-mo 500` | Slower, more reliable | Fast SPAs, race conditions |

---

## 10. Verification Checklist

After recording, verify:

- [ ] **metadata.json exists** in `recordings/<session>/`
- [ ] **Actions array has entries:** `len(data['actions']) > 0`
- [ ] **Actions have selectors:** Each action has `element.stableSelector`
- [ ] **Actions have types:** `action['action']` is 'click', 'fill', or 'press'
- [ ] **Pages array populated:** For multi-page flows
- [ ] **No warnings array:** Check `data['warnings']` for issues
- [ ] **Artifacts exist:** HAR, trace, DOM, screenshots (if enabled)

---

## 11. Refinement Process (recorder_auto_ingest.py)

After recording, `auto_refine_and_ingest()` processes the raw metadata:

```python
# 1. Load metadata.json
metadata = json.load(open("metadata.json"))

# 2. Convert raw actions to refined steps
for action in metadata['actions']:
    # Extract action type
    action_type = action['action']  # 'click', 'change', 'press', etc.
    
    # Map to test step
    if action_type == 'change':
        step = {'action': 'fill', 'value': action['extra']['value']}
    elif action_type == 'click':
        step = {'action': 'click', 'value': ''}
    elif action_type == 'press':
        step = {'action': 'press', 'value': action['extra']['key']}
    
    # Extract best selector
    selectors = action['selectorStrategies']
    step['selector'] = (
        selectors['aria'] or          # getByRole/getByLabel
        selectors['playwright'] or     # Playwright-style
        selectors['css'] or            # CSS selector
        selectors['xpath']             # XPath fallback
    )
    
    refined_steps.append(step)

# 3. Enrich with context
for step in refined_steps:
    step['navigation'] = describe_navigation(step)
    step['data'] = extract_data_label(step)
    step['expected'] = infer_expected_result(step)

# 4. Save refined flow
output = {
    'flow_name': flow_name,
    'steps': refined_steps,
    'elements': unique_elements
}
json.dump(output, open(f"{flow_name}.refined.json", "w"))

# 5. Ingest to vector DB
ingest_refined_file(output_path, flow_name)
```

**Common Refinement Issues:**

1. **No steps generated:** Raw metadata had no valid actions
2. **Missing selectors:** Elements had no stable identifier
3. **Degraded actions:** Console fallback used (incomplete data)

---

## 12. Best Practices

### For Reliable Recording:

1. **Wait for page load:** Don't interact immediately
2. **Use visible elements:** Hidden elements may not capture properly
3. **Avoid rapid clicks:** Give 500ms between actions
4. **Test with --slow-mo:** Helps with timing issues
5. **Check console:** Look for `[recorder]` logs in DevTools
6. **Use proper selectors:** Add `data-testid` to important elements

### For Your New Recorder:

If your new recorder is missing steps, check:

1. **Are events being registered?**
   ```javascript
   // Add console.log to verify
   document.addEventListener('click', e => {
       console.log('CLICK CAPTURED:', e.target);
       // ... your handler
   }, true);  // ← Must be true for capture phase
   ```

2. **Is the script injected early enough?**
   ```javascript
   // Check injection timing
   if (document.readyState === 'loading') {
       // Good - injected before DOM ready
   } else {
       // Bad - may miss early events
   }
   ```

3. **Are you using the capture phase?**
   ```javascript
   addEventListener('click', handler, true);  // ✅ Correct
   addEventListener('click', handler, false); // ❌ May miss events
   ```

4. **Is there a queue/buffer system?**
   - Events may come faster than you can process
   - Queue prevents loss during async operations

5. **Do you handle iframes?**
   - Each frame needs separate instrumentation
   - Use frame attachment handlers

6. **Do you handle popups?**
   - New windows need separate instrumentation
   - Listen for window.open() / popup events

---

## Summary

The recorder's reliability comes from:

1. **Early injection** - Script runs before page loads
2. **Capture phase** - Intercepts events before apps can block them
3. **Queue system** - Buffers events during async operations
4. **Multiple fallbacks** - Console logs if bindings fail
5. **Comprehensive snapshots** - Captures 30+ element properties
6. **Multiple selector strategies** - 8 fallback levels for reliability
7. **Frame/popup handling** - Instruments all page contexts
8. **Aggressive drain** - Doesn't lose events on shutdown
9. **SPA detection** - Tracks history API changes
10. **Shadow DOM piercing** - Uses composedPath() to reach actual targets

If your recorder is missing steps, the most likely issues are:
- Missing capture phase flag (`true`)
- No queue/buffer system
- Script injected too late
- Not handling iframes/popups
- CSP blocking injection
