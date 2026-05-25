---
name: google-recorder-download
version: 1.0.0
description: |
  Download all recordings from Google Recorder (recorder.google.com) to a local
  folder. Handles: shadow DOM navigation, "Load more" pagination, per-recording
  ⋮→Download→confirm flow, fire-and-forget async loop (no CDP timeout), title-
  based dedup against already-downloaded files, generation-guard cancellation,
  move from ~/Downloads to target folder, and failure retry. Invoke when user
  says "download my Google Recorder recordings", "export Recorder", or shares a
  recorder.google.com URL asking to download/backup.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - mcp__Claude_in_Chrome__javascript_tool
  - mcp__Claude_in_Chrome__navigate
  - mcp__Claude_in_Chrome__get_page_text
  - mcp__Claude_in_Chrome__read_page
  - mcp__Claude_in_Chrome__screenshot
  - mcp__Claude_in_Chrome__list_connected_browsers
  - mcp__Claude_in_Chrome__select_browser
  - mcp__Claude_in_Chrome__tabs_context_mcp
  - AskUserQuestion
---

# Google Recorder — Bulk Download Skill

You are automating the bulk download of every recording from Google Recorder's
web app to a local folder. There is **no official bulk-export API**; every file
must be triggered individually through the UI. This skill encodes the complete
hardened approach developed over a real 418-recording session.

---

## Step 0 — Gather parameters

Ask the user (or infer from their message):

| Parameter | Default | Notes |
|-----------|---------|-------|
| `TARGET`  | `~/Desktop/RECORDINGS` | Absolute path to destination folder |
| `RECORDER_URL` | `https://recorder.google.com` | Use URL they shared, or navigate manually |
| `AUDIO_FORMAT` | `.m4a` | The only option Google Recorder offers |

Create the target folder if it doesn't exist:
```bash
mkdir -p "$TARGET"
```

---

## Step 1 — Connect Chrome and navigate

Load the Chrome MCP tools:
```
ToolSearch: query="select:mcp__Claude_in_Chrome__javascript_tool,mcp__Claude_in_Chrome__navigate,mcp__Claude_in_Chrome__list_connected_browsers,mcp__Claude_in_Chrome__select_browser", max_results=10
```

List connected browsers and select the one with Google Recorder open (or navigate there):
```js
// Check current URL via javascript_tool
window.location.href
```

If not on recorder.google.com, navigate there. The user must be logged in — do not handle authentication.

---

## Step 2 — Seed the done-set from existing files

Run this Python snippet to convert filenames already in `TARGET` back to the
`document.title` format that the recorder uses. Pass the result into the JS
done-set so already-downloaded recordings are skipped.

```python
#!/usr/bin/env python3
import os, sys, json, re

target = sys.argv[1]   # e.g. ~/Desktop/RECORDINGS
titles = []
for f in os.listdir(target):
    if f.lower().endswith('.m4a'):
        n = f[:-4]
        n = re.sub(r'\s*\(\d+\)', '', n).strip()          # strip (1) duplicates
        n = re.sub(r'(\d{1,2})-(\d{2})(\s*(?:AM|PM))', r'\1:\2\3', n)  # dash→colon
        titles.append(n)
print(json.dumps(sorted(set(titles))))
```

Invoke:
```bash
python3 /tmp/seed_done.py "$TARGET"
```

Capture the JSON array for injection into Step 4.

---

## Step 3 — Expand the full recording list ("Load more")

Paste this into `javascript_tool`. It clicks "Load more" until all recordings
are in the DOM and returns the final count. Returns immediately (resolves async
in background) — BUT because it uses a setInterval it's safe to await:

```js
// STEP 3: Expand sidebar — call once before the download loop
// Returns: total number of recorder-sidebar-item elements loaded

(function expandAll() {
  function fastRoots() {
    const acc = [];
    (function w(r) {
      acc.push(r);
      const k = r.querySelectorAll ? r.querySelectorAll('*') : [];
      for (const e of k) {
        if (e.shadowRoot && e.tagName !== 'RECORDER-SIDEBAR-ITEM')
          w(e.shadowRoot);
      }
    })(document);
    return acc;
  }
  function fastFind(sel) {
    const acc = [];
    for (const r of fastRoots()) {
      if (r.querySelectorAll) acc.push(...r.querySelectorAll(sel));
    }
    return acc;
  }

  function clickLoadMore() {
    for (const b of fastFind('button')) {
      const t = (b.textContent || '').trim();
      const lbl = b.getAttribute('aria-label') || '';
      if ((t === 'Load more' || lbl === 'Load more') && b.getBoundingClientRect().width > 0) {
        b.click(); return true;
      }
    }
    return false;
  }

  let clicks = 0;
  const iv = setInterval(() => {
    const did = clickLoadMore();
    if (did) clicks++;
    if (!did || clicks > 250) {
      clearInterval(iv);
    }
  }, 600);

  return 'expandAll started — check count in ~60s with: fastFind("recorder-sidebar-item").length';
})();
```

After ~60–90 seconds (depending on how many "Load more" pages exist), verify:
```js
// STEP 3b: Verify full count loaded
(function() {
  function fastRoots() {
    const acc = [];
    (function w(r) {
      acc.push(r);
      const k = r.querySelectorAll ? r.querySelectorAll('*') : [];
      for (const e of k) {
        if (e.shadowRoot && e.tagName !== 'RECORDER-SIDEBAR-ITEM')
          w(e.shadowRoot);
      }
    })(document);
    return acc;
  }
  const items = [];
  for (const r of fastRoots()) {
    if (r.querySelectorAll) items.push(...r.querySelectorAll('recorder-sidebar-item'));
  }
  return items.length + ' items in sidebar';
})();
```

---

## Step 4 — Launch the fire-and-forget download loop

**CRITICAL**: This JS evaluates to a short JSON string immediately. The actual
download loop runs as a background async IIFE stored in `window.__loop`. This
avoids the CDP 45-second timeout that kills awaited long-running JS.

Replace `DONE_SET_JSON` with the JSON array from Step 2.

```js
// STEP 4: Fire-and-forget download loop
// Replace DONE_SET_JSON with the array from Step 2, e.g.: ["May 24 at 9:27 AM","Karan"]

(function launchLoop() {
  // ── Element finders ───────────────────────────────────────────────────────
  function fastRoots() {
    const acc = [];
    (function w(r) {
      acc.push(r);
      const k = r.querySelectorAll ? r.querySelectorAll('*') : [];
      for (const e of k) {
        if (e.shadowRoot && e.tagName !== 'RECORDER-SIDEBAR-ITEM')
          w(e.shadowRoot);
      }
    })(document);
    return acc;
  }
  function fastFind(sel) {
    const acc = [];
    for (const r of fastRoots()) {
      if (r.querySelectorAll) acc.push(...r.querySelectorAll(sel));
    }
    return acc;
  }
  function title() {
    return (document.title || '').replace(/\s*-\s*Recorder$/, '').trim();
  }
  function menuBtn() {
    return fastFind('button').find(b =>
      (b.getAttribute('aria-label') || '') === 'Settings' &&
      (b.textContent || '').includes('more_vert')
    );
  }
  function dlItem() {
    return fastFind('mwc-list-item,[role=menuitem]').find(b => {
      const tt = (b.textContent || '').trim();
      const r = b.getBoundingClientRect();
      return /^Download$/i.test(tt) && r.width > 0 && r.y < 200;
    });
  }
  function confirmBtn() {
    return fastFind('button,mwc-button').find(b => {
      const tt = (b.textContent || '').trim();
      const r = b.getBoundingClientRect();
      return /^Download$/i.test(tt) && r.width > 0 && r.y > 300;
    });
  }
  function cancelBtn() {
    return fastFind('button,mwc-button').find(b => {
      const tt = (b.textContent || '').trim();
      const r = b.getBoundingClientRect();
      return /^Cancel$/i.test(tt) && r.width > 0;
    });
  }

  // ── Helpers ───────────────────────────────────────────────────────────────
  function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }
  function poll(fn, timeoutMs) {
    return new Promise(res => {
      const t0 = Date.now();
      const iv = setInterval(() => {
        const v = fn();
        if (v) { clearInterval(iv); res(v); return; }
        if (Date.now() - t0 > timeoutMs) { clearInterval(iv); res(null); }
      }, 80);
    });
  }

  // ── Per-recording download ────────────────────────────────────────────────
  async function dlOne(items, i, prev) {
    items[i].scrollIntoView({ block: 'center' });
    items[i].click();

    // Wait until title changed AND toolbar is rendered (up to 8s for long recordings)
    const t0 = Date.now();
    while (Date.now() - t0 < 8000) {
      if (title() !== prev && menuBtn()) break;
      await sleep(100);
    }
    await sleep(250);

    const cur = title();
    window.__dl.lastTitle = cur;

    if (window.__dl.done[cur]) return { title: cur, skip: true };

    if (!menuBtn()) {
      await sleep(1500);
      if (!menuBtn()) return { title: cur, err: 'not loaded' };
    }

    // Open ⋮ menu, find Download item (retry up to 4×)
    let item = null;
    for (let a = 0; a < 4 && !item; a++) {
      const mb = menuBtn();
      if (!mb) { await sleep(250); continue; }
      mb.click();
      item = await poll(dlItem, 1300);
    }
    if (!item) {
      const c = cancelBtn(); if (c) c.click();
      return { title: cur, err: 'no dlitem' };
    }
    item.click();

    // Confirm format dialog
    const cf = await poll(confirmBtn, 2200);
    if (!cf) {
      const c = cancelBtn(); if (c) c.click();
      return { title: cur, err: 'no confirm' };
    }
    cf.click();
    await sleep(300);

    window.__dl.done[cur] = true;
    window.__dl.ok++;
    return { title: cur, ok: true };
  }

  // ── Initialise state ──────────────────────────────────────────────────────
  if (!window.__gen) window.__gen = 0;
  window.__gen++;
  const myGen = window.__gen;

  // Seed done-set from already-downloaded files
  const SEED = DONE_SET_JSON;  // ← replace with array from Step 2
  if (!window.__dl || window.__dl.gen !== myGen) {
    const done = {};
    for (const t of SEED) done[t] = true;
    window.__dl = { done, cursor: 0, failed: [], ok: 0, phase: 'running', gen: myGen, lastTitle: null, beat: Date.now() };
  }

  // ── Fire-and-forget loop ──────────────────────────────────────────────────
  window.__loop = (async () => {
    let prev = title();
    const items = fastFind('recorder-sidebar-item');

    while (window.__dl.cursor < items.length) {
      if (window.__gen !== myGen) { window.__dl.phase = 'cancelled'; return; }
      window.__dl.beat = Date.now();

      const i = window.__dl.cursor;
      let r;
      try {
        r = await dlOne(items, i, prev);
      } catch (e) {
        r = { err: 'ex:' + (e && e.message || String(e)) };
        try { const c = cancelBtn(); if (c) c.click(); } catch (_) {}
      }

      prev = (r && r.title) || prev;
      if (r && r.err) window.__dl.failed.push({ i, t: r.title || null, e: r.err });
      window.__dl.cursor = i + 1;
      await sleep(50);
    }

    window.__dl.phase = 'done';
  })();

  return JSON.stringify({ launched: true, gen: myGen, seed: SEED.length, items: 'call fastFind("recorder-sidebar-item").length to verify' });
})();
```

---

## Step 5 — Snapshot Downloads folder baseline

Before the loop starts downloading (or right after launching it):

```bash
ls -1 ~/Downloads > /tmp/dl_baseline.txt
echo "Baseline: $(wc -l < /tmp/dl_baseline.txt) files"
```

---

## Step 6 — Poll status

Run this JS every 60–120 seconds to check progress:

```js
JSON.stringify({
  phase:     window.__dl ? window.__dl.phase    : 'not started',
  cursor:    window.__dl ? window.__dl.cursor   : 0,
  ok:        window.__dl ? window.__dl.ok       : 0,
  failedN:   window.__dl ? window.__dl.failed.length : 0,
  last:      window.__dl ? window.__dl.lastTitle : null,
  beatAgoMs: window.__dl ? Date.now() - window.__dl.beat : null
});
```

**Healthy heartbeat**: `beatAgoMs` should be < 15000 (15s). If it's > 30s and
`phase` is still `'running'`, the loop may be stalled — see Troubleshooting.

**Done condition**: `phase === 'done'` → proceed to Step 7.

---

## Step 7 — Move files from ~/Downloads to TARGET

```bash
TARGET="$HOME/Desktop/RECORDINGS"
DL="$HOME/Downloads"

# Move all new .m4a files added since baseline
comm -13 <(sort /tmp/dl_baseline.txt) <(ls -1 "$DL" | sort) \
  | grep -i '\.m4a$' \
  | while IFS= read -r f; do
      [ -n "$f" ] && mv -- "$DL/$f" "$TARGET/$f"
    done

echo "Files now in TARGET: $(ls -1 "$TARGET"/*.m4a 2>/dev/null | wc -l)"
```

**Dedup (1) files**: Chrome appends ` (1)` when a same-named file already exists.
After moving, find and review:
```bash
ls "$TARGET"/*\ \(1\)*.m4a 2>/dev/null
# Typically safe to delete — original already present
# rm "$TARGET"/*\ \(1\)*.m4a
```

---

## Step 8 — Retry failures

```js
// Check what failed
JSON.stringify(window.__dl.failed.map(f => ({i: f.i, t: f.t, e: f.e})));
```

For any failures, re-run Step 4 **without** resetting `window.__dl` — the
generation guard will restart the loop from cursor 0 but skip all done items,
effectively only re-attempting items not in `done{}`. Or manually click each
failed recording and use the three-step menu flow.

To force a retry of all failed items, note their titles and remove them from
the done-set:
```js
const toRetry = ['Recording Title Here'];
for (const t of toRetry) delete window.__dl.done[t];
// Then re-increment gen and relaunch Step 4 code (it will re-process undone items)
```

---

## Step 9 — Verify final count

```bash
TARGET="$HOME/Desktop/RECORDINGS"
echo "Total .m4a in target: $(ls -1 "$TARGET"/*.m4a 2>/dev/null | wc -l)"
```

Compare against the total shown in Google Recorder sidebar. If counts match → done.

---

## Troubleshooting

### Loop stalled (`beatAgoMs` > 30s, phase still 'running')

The loop may be stuck waiting on a dialog. Run:
```js
// Cancel any stuck dialog
(function() {
  function fastRoots() {
    const acc = [];
    (function w(r) {
      acc.push(r);
      for (const e of (r.querySelectorAll ? r.querySelectorAll('*') : [])) {
        if (e.shadowRoot && e.tagName !== 'RECORDER-SIDEBAR-ITEM') w(e.shadowRoot);
      }
    })(document);
    return acc;
  }
  function fastFind(sel) {
    const acc = [];
    for (const r of fastRoots()) if (r.querySelectorAll) acc.push(...r.querySelectorAll(sel));
    return acc;
  }
  const c = fastFind('button,mwc-button').find(b => /^Cancel$/i.test((b.textContent||'').trim()) && b.getBoundingClientRect().width > 0);
  if (c) { c.click(); return 'clicked Cancel'; }
  return 'no Cancel button found';
})();
```

Then check status again in 10s. If still stalled, cancel and relaunch:
```js
window.__gen++;  // cancel old loop
```
Then re-run Step 4 (it will resume from `window.__dl.cursor`, skipping done items).

### CDP timeout (evaluate call hangs / returns error after 45s)

This should NOT happen with the fire-and-forget pattern. If it does, verify the
Step 4 code has `return JSON.stringify({launched:true,...})` OUTSIDE the async
IIFE, not inside it.

### "Load more" didn't load all recordings

The interval fires every 600ms. If internet is slow or the page throttles, increase
to 900ms and re-run Step 3. The `clicks > 250` guard prevents infinite loops.

### Wrong recordings getting downloaded (index drift)

The title-based done-set is the authoritative dedup layer. Index drift is harmless
because already-downloaded recordings return `{skip:true}` immediately. If a file
appears as a duplicate `(1)`, delete the `(1)` copy — the original is correct.

### "no dlitem" / "no confirm" errors

The ⋮ menu or confirm dialog didn't appear in time. These items go into
`window.__dl.failed` and are skipped. Retry via Step 8. The root cause is usually
a slow-loading recording — the load-wait in `dlOne` handles most cases but the
8s cap may be too short for very large files (e.g. 2h+ recordings).

---

## Performance Notes

- **Rate**: ~1 recording per 3–5 seconds = ~12–20 per minute = 418 recordings in ~25–35 minutes
- **Shadow DOM**: Google Recorder has 500+ shadow roots; `fastRoots()` skips the 418 `RECORDER-SIDEBAR-ITEM` roots, reducing traversal cost ~6×
- **Parallelism**: Only one recording downloads at a time (Chrome enforces one download dialog)
- **Memory**: `window.__dl.done` grows by one string per recording — negligible
- **Page must stay open**: The browser tab must remain open and active (not backgrounded to the point of being frozen by the browser's tab suspension)
