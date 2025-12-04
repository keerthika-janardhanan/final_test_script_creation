# Recorder Troubleshooting - Empty Actions Array

## Issue Analysis

Your recorder metadata shows:
```json
"actions": []  // ← EMPTY - No user interactions captured
"pageContextEvents": [...]  // ← 6 events captured successfully
```

**Status:**
- ✅ Script injection working (pageContextEvents captured)
- ✅ Navigation tracking working (OAuth flow tracked)
- ✅ DOM snapshots working (dom/ directory populated)
- ✅ Screenshots working (screenshots/ directory populated)
- ❌ **User interactions NOT captured** (clicks, inputs, keypresses)

## Root Causes

### 1. Content Security Policy (CSP) Blocking
Oracle IDCS (Identity Cloud Service) pages have extremely strict CSP that blocks:
- Cross-origin script execution
- Playwright binding communication
- `eval()` and dynamic code execution

**Evidence:**
```json
"bypassCsp": true  // Enabled but insufficient for Oracle Cloud
"pageUrl": "https://idcs-03948457336f43a581f13551f128012c.identity.oraclecloud.com/..."
```

Oracle Cloud infrastructure uses multiple layers of security that bypass-csp alone cannot overcome.

### 2. Cross-Origin Communication Failure
The OAuth redirect chain breaks binding context:
```
Page 1: ecqg-test.fa.us2.oraclecloud.com
  ↓ redirect
Page 2: idcs-...oraclecloud.com/oauth2/v1/authorize
  ↓ redirect  
Page 3: idcs-...oraclecloud.com/ui/v1/signin  ← Login page (where you interact)
```

Each redirect creates a new page context. If bindings aren't re-established on Page 3, actions are lost.

### 3. Page Not Fully Loaded
```json
"pageTitle": "",     // Empty - page structure incomplete
"headings": []       // Empty - DOM not accessible
"mainHeading": ""    // Empty - no content detected
```

This indicates:
- JavaScript may not have access to the DOM
- Page security restrictions preventing introspection
- Script injection timing issue (injected after page locked down)

### 4. Playwright Binding Blocked
The communication channel from browser → Python is broken:

```javascript
// Browser side (should work):
window.pythonRecorderCapture(actionData)  // ← This function likely doesn't exist

// Python side (never receives):
def _enqueue_action(source, payload):  // ← Never called
    pending_actions.append(payload)
```

## Diagnostic Checklist

### Test 1: Verify Bindings Are Exposed

**In Browser DevTools Console (during recording):**
```javascript
// Test 1: Check script injection
window.__pyRecInstalled
// Expected: true
// If undefined: Script injection failed

// Test 2: Check binding function
typeof window.pythonRecorderCapture
// Expected: "function"  
// If undefined: Binding communication blocked by CSP

// Test 3: Check queue system
window.capQ
// Expected: Array (may be empty)
// If undefined: Script not executing properly

// Test 4: Manual action test
const testAction = {
    action: 'click',
    pageUrl: window.location.href,
    pageTitle: document.title,
    timestamp: Date.now(),
    element: {tag: 'button'},
    extra: {}
};

// Try direct binding call
if (typeof window.pythonRecorderCapture === 'function') {
    window.pythonRecorderCapture(testAction);
    console.log('✅ Binding works - check Python logs');
} else {
    console.log('❌ Binding missing - CSP blocking');
}
```

### Test 2: Check Console Logs

**In Browser Console, filter for:**
```
[recorder]        // Should see action logs
[recorder-json]   // Fallback mechanism logs
```

**Expected output when clicking:**
```
[recorder] action click button https://...
```

**If nothing appears:** Event listeners not registered.

### Test 3: Check Python Output

**Terminal should show:**
```
[recorder][console] debug: [recorder] action click button ...
[recorder] Recorded X actions.
```

**If missing:** Binding communication completely broken.

### Test 4: Network Errors

**Check for:**
```
[recorder][requestfailed] ... (URL) -> (error)
```

Network failures during OAuth can interrupt binding setup.

## Solutions

### Solution 1: Enhanced Browser Launch Arguments (CRITICAL)

Oracle Cloud requires disabling web security entirely. Modify `run_playwright_recorder_v2.py`:

**Location:** Around line 600-700 in `_build_context()` function

**Add these launch arguments:**
```python
def _build_context(
    playwright: Playwright,
    browser_name: str,
    headless: bool,
    slow_mo: Optional[int],
    har_path: Optional[Path],
    ignore_https_errors: bool,
    user_agent: Optional[str],
    proxy_server: Optional[str] = None,
    launch_args: Optional[List[str]] = None,
    bypass_csp: bool = False,
) -> BrowserContext:
    name = normalize_browser_name(browser_name, SUPPORTED_BROWSERS)
    factory = getattr(playwright, name)
    
    # Enhanced launch args for Oracle Cloud / strict CSP sites
    enhanced_args = [
        '--disable-web-security',              # Bypass CORS/CSP
        '--disable-site-isolation-trials',     # Allow cross-origin access
        '--disable-features=IsolateOrigins,site-per-process',  # Disable isolation
        '--allow-running-insecure-content',    # Allow mixed content
        '--ignore-certificate-errors',         # Ignore cert issues
    ]
    
    if launch_args:
        enhanced_args.extend(launch_args)
    
    launch_kwargs: Dict[str, Any] = {
        "headless": headless,
        "slow_mo": slow_mo,
        "args": enhanced_args  # ← Add this
    }
    
    # ... rest of function
```

**Then run:**
```bash
python -m app.run_playwright_recorder_v2 \
    --url "https://ecqg-test.fa.us2.oraclecloud.com/" \
    --session-name "oracle-enhanced" \
    --bypass-csp \
    --capture-dom \
    --capture-screenshots \
    --slow-mo 500 \
    --timeout 180
```

### Solution 2: Add Explicit Binding Verification

Add this after navigation completes (around line 950 in main()):

```python
# After page.goto() and page.wait_for_load_state()

# Explicitly verify bindings are ready
print("[recorder] Verifying bindings...")
bindings_ready = False
for attempt in range(20):  # Try for up to 10 seconds
    try:
        result = page.evaluate("""
            () => {
                return {
                    installed: window.__pyRecInstalled === true,
                    hasCapture: typeof window.pythonRecorderCapture === 'function',
                    hasContext: typeof window.pythonRecorderPageContext === 'function',
                    queueSize: window.capQ ? window.capQ.length : -1
                };
            }
        """)
        
        if result['installed'] and result['hasCapture']:
            print(f"[recorder] ✅ Bindings ready: {result}")
            bindings_ready = True
            break
        else:
            print(f"[recorder] ⏳ Attempt {attempt + 1}/20: {result}")
        
        time.sleep(0.5)
    except Exception as e:
        print(f"[recorder] ⚠️  Binding check failed: {e}")
        time.sleep(0.5)

if not bindings_ready:
    print("[recorder] ❌ WARNING: Bindings never became ready!")
    print("[recorder] Actions may not be captured. Consider:")
    print("  1. Using --disable-web-security flag")
    print("  2. Recording after manual login")
    print("  3. Using a less restrictive page")
```

### Solution 3: Use Console Fallback Exclusively

If bindings are completely blocked, rely on console fallback. Add this diagnostic:

```python
# In _on_console_with_fallback function (around line 1100)

def _on_console_with_fallback(msg: ConsoleMessage) -> None:
    try:
        text = msg.text
        msg_type = msg.type
        
        # Always log console for debugging
        sys.stderr.write(f"[recorder][console][{msg_type}] {text}\n")
        
        # Check if our recorder script is logging
        if '[recorder]' in text:
            sys.stderr.write(f"[recorder] ✅ Recorder script is active\n")
        
        # Enhanced fallback detection
        if session and isinstance(text, str) and msg_type == "debug":
            # JSON payload fallback
            if text.startswith("[recorder-json]"):
                try:
                    jtxt = text[len("[recorder-json]"):].strip()
                    payload = json.loads(jtxt)
                    if isinstance(payload, dict):
                        ex = payload.get("extra") or {}
                        ex["fromConsole"] = True
                        payload["extra"] = ex
                        print(f"[recorder] 📝 Captured via console: {payload.get('action')}")
                        session.add_action(payload, runtime_page=None)
                        return
                except Exception as parse_err:
                    sys.stderr.write(f"[recorder] Failed to parse JSON: {parse_err}\n")
            
            # Simple text fallback
            if text.startswith("[recorder] action"):
                parts = text.split()
                if len(parts) >= 5:
                    act = parts[2]
                    tag = parts[3]
                    url = parts[4]
                    fallback = {
                        "action": act,
                        "pageUrl": url,
                        "element": {"tag": tag},
                        "extra": {"fromConsole": True}
                    }
                    print(f"[recorder] 📝 Captured via console fallback: {act} on {tag}")
                    session.add_action(fallback, runtime_page=None)
    except Exception as e:
        sys.stderr.write(f"[recorder] Console handler error: {e}\n")
```

### Solution 4: Test on Simple Page First

**Before debugging Oracle Cloud, verify recorder works:**

```bash
# Test 1: Simple static page
python -m app.run_playwright_recorder_v2 \
    --url "https://example.com" \
    --session-name "test-simple" \
    --capture-dom \
    --timeout 30

# Test 2: Form page
python -m app.run_playwright_recorder_v2 \
    --url "https://www.w3schools.com/html/html_forms.asp" \
    --session-name "test-forms" \
    --capture-dom \
    --timeout 60
```

**Then check:**
```bash
# PowerShell
cat recordings\test-simple\metadata.json | Select-String '"actions"'
```

**Expected:** Should see action objects, not empty array.

**If empty on simple pages too:** Core recorder issue, not CSP-related.

### Solution 5: Record After Manual Login

For Oracle Fusion apps, bypass the login screen entirely:

**Steps:**
1. Manually open browser
2. Log in to Oracle Fusion
3. Copy the post-login URL (e.g., `https://ecqg-test.fa.us2.oraclecloud.com/fscmUI/faces/FndOverview`)
4. Start recorder with that URL:

```bash
python -m app.run_playwright_recorder_v2 \
    --url "https://ecqg-test.fa.us2.oraclecloud.com/fscmUI/faces/FndOverview" \
    --session-name "post-login-recording" \
    --bypass-csp \
    --capture-dom \
    --capture-screenshots \
    --slow-mo 500 \
    --timeout 300
```

This avoids the OAuth redirect chain that breaks bindings.

### Solution 6: Use Authenticated Context

Create a script to handle authentication first:

```python
# auth_and_record.py
from playwright.sync_api import sync_playwright
import time
import json

with sync_playwright() as p:
    browser = p.chromium.launch(
        headless=False,
        args=[
            '--disable-web-security',
            '--disable-site-isolation-trials',
        ]
    )
    
    context = browser.new_context(
        bypass_csp=True,
        ignore_https_errors=True
    )
    
    page = context.new_page()
    
    # Step 1: Manual login
    print("Navigate to Oracle Cloud and log in manually...")
    page.goto("https://ecqg-test.fa.us2.oraclecloud.com/")
    input("Press Enter after logging in...")
    
    # Step 2: Save authenticated state
    context.storage_state(path="oracle_auth.json")
    print("✅ Authentication state saved")
    
    # Step 3: Now use this state with recorder
    browser.close()

# Then modify recorder to use saved auth:
# context = browser.new_context(storage_state="oracle_auth.json")
```

## Verification Steps After Fix

### 1. Check Binding Status
During recording, open DevTools and run:
```javascript
{
    installed: window.__pyRecInstalled,
    capture: typeof window.pythonRecorderCapture,
    queue: window.capQ?.length || 0
}
```

Should return:
```javascript
{installed: true, capture: "function", queue: 0}
```

### 2. Test Manual Action
Click something, then in DevTools:
```javascript
window.capQ  // Should have entries
```

### 3. Check metadata.json
After recording:
```bash
# PowerShell
$metadata = Get-Content recordings\<session>\metadata.json | ConvertFrom-Json
$metadata.actions.Length  # Should be > 0
$metadata.actions | ForEach-Object { "$($_.action): $($_.element.tag)" }
```

Expected output:
```
click: button
change: input
press: input
```

### 4. Verify Artifacts
```bash
ls recordings\<session>\
```

Should see:
```
metadata.json          ✅
dom\                   ✅  
screenshots\           ✅
network.har            ❌ (optional)
trace.zip              ❌ (optional)
```

## Common Patterns for Empty Actions

### Pattern 1: CSP Blocking
```json
"actions": []
"warnings": []  // No warnings reported
"bypassCsp": true
```
**Fix:** Solution 1 (enhanced launch args)

### Pattern 2: Bindings Never Ready
```json
"actions": []
"pageContextEvents": [{"trigger": "init"}, ...]  // Multiple init events
```
**Fix:** Solution 2 (explicit binding verification)

### Pattern 3: Console Fallback Not Working
```json
"actions": []
"pageContextEvents": [...]  // Many events
```
**Fix:** Solution 3 (enhanced console fallback)

### Pattern 4: Page Security Too Strict
```json
"actions": []
"pageTitle": ""
"headings": []
```
**Fix:** Solution 5 (record after login)

## Oracle Cloud Specific Issues

### Issue: OAuth Redirect Chain
Oracle Fusion uses multi-step OAuth:
1. Initial page load
2. Redirect to IDCS OAuth endpoint
3. Redirect to IDCS login UI
4. Post-auth redirect back to app

**Each redirect can break bindings.**

**Solution:** Start recording AFTER authentication completes.

### Issue: ADF Framework
Oracle uses ADF (Application Development Framework) which:
- Heavily uses iframes
- Has custom event handling that calls stopPropagation()
- Uses shadow DOM in some components
- Lazy-loads content dynamically

**Solution:** Add frame handlers and use slower timing:
```bash
--slow-mo 1000  # 1 second between actions
```

### Issue: Session Timeout
Oracle Cloud sessions can timeout during recording.

**Solution:** Use shorter timeout, work in focused flows:
```bash
--timeout 120  # 2 minutes max
```

## Success Criteria

Your recording is successful when metadata.json shows:

```json
{
  "actions": [
    {
      "actionId": "A-001",
      "action": "click",
      "type": "click",
      "element": {
        "tag": "input",
        "id": "username",
        "stableSelector": "#username",
        "xpath": "/html/body/div/form/input[1]"
      },
      "selectorStrategies": {
        "playwright": "#username",
        "css": "#username",
        "xpath": "/html/body/div/form/input[1]"
      }
    },
    {
      "actionId": "A-002",
      "action": "fill",
      "type": "change",
      "element": {
        "tag": "input",
        "type": "text"
      },
      "extra": {
        "value": "testuser",
        "valueMasked": false
      }
    }
  ],
  "pages": [
    {
      "actions": [...]  // Actions associated with page
    }
  ]
}
```

**Key indicators:**
- ✅ `actions.length > 0`
- ✅ Each action has `element.stableSelector`
- ✅ Each action has `selectorStrategies`
- ✅ Actions have correct `action` type (click/fill/press)
- ✅ `pages[0].actions` array populated

## Quick Debug Script

Save as `debug_recorder.py`:

```python
import json
import sys
from pathlib import Path

def analyze_metadata(path: str):
    with open(path) as f:
        data = json.load(f)
    
    print("=" * 60)
    print("RECORDER METADATA ANALYSIS")
    print("=" * 60)
    
    # Basic counts
    actions = data.get("actions", [])
    pages = data.get("pages", [])
    events = data.get("pageContextEvents", [])
    warnings = data.get("warnings", [])
    
    print(f"\n📊 Counts:")
    print(f"  Actions: {len(actions)}")
    print(f"  Pages: {len(pages)}")
    print(f"  Context Events: {len(events)}")
    print(f"  Warnings: {len(warnings)}")
    
    # Actions analysis
    if actions:
        print(f"\n✅ ACTIONS CAPTURED:")
        for i, action in enumerate(actions[:5], 1):
            print(f"  {i}. {action.get('action')} on {action.get('element', {}).get('tag', 'unknown')}")
        if len(actions) > 5:
            print(f"  ... and {len(actions) - 5} more")
    else:
        print(f"\n❌ NO ACTIONS CAPTURED")
        print(f"\n🔍 Diagnostics:")
        
        # Check page state
        if pages:
            page = pages[0]
            print(f"  Page Title: '{page.get('pageTitle', 'N/A')}'")
            print(f"  Page URL: {page.get('pageUrl', 'N/A')}")
            print(f"  Headings: {len(page.get('headings', []))}")
            
            if not page.get('pageTitle'):
                print(f"  ⚠️  Empty page title - DOM access blocked?")
            
            if not page.get('headings'):
                print(f"  ⚠️  No headings - script may not have access to DOM")
        
        # Check warnings
        if warnings:
            print(f"\n⚠️  Warnings:")
            for w in warnings:
                print(f"    - {w}")
        
        # Check CSP
        options = data.get("options", {})
        if options.get("bypassCsp"):
            print(f"\n  ℹ️  CSP bypass enabled but actions still empty")
            print(f"     → Try enhanced launch args (Solution 1)")
        else:
            print(f"\n  ⚠️  CSP bypass NOT enabled")
            print(f"     → Run with --bypass-csp flag")
    
    # Events analysis
    if events:
        print(f"\n📡 Page Context Events:")
        triggers = {}
        for e in events:
            t = e.get("trigger", "unknown")
            triggers[t] = triggers.get(t, 0) + 1
        for trigger, count in triggers.items():
            print(f"  {trigger}: {count}")
    
    print("\n" + "=" * 60)

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python debug_recorder.py <path/to/metadata.json>")
        sys.exit(1)
    
    analyze_metadata(sys.argv[1])
```

**Usage:**
```bash
python debug_recorder.py recordings\testtest\metadata.json
```

## Next Steps

1. **Immediate:** Test with simple page (example.com) to isolate issue
2. **If simple page works:** Apply Solution 1 (enhanced launch args) for Oracle Cloud
3. **If simple page fails:** Core recorder issue - check Python/Playwright installation
4. **After fix:** Verify using debug script above

## Support Information to Provide

If issue persists, provide:

1. **Full terminal output** during recording
2. **Browser console logs** (entire console, not filtered)
3. **Result of binding test** (from Test 1 above)
4. **Output of debug script** on your metadata.json
5. **Python version:** `python --version`
6. **Playwright version:** `pip show playwright`
7. **OS:** Windows version

This will help identify the exact blocking mechanism.
