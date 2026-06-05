---
name: viz-verify
description: Visual verification workflow. Screenshot → analyze → map
  to code → fix → re-capture → compare. Pure procedure — no gates.
category: lambda-skills
version: 0.4.0
---

NARRATOR
    "Viz-verify, the Director has a screenshot. What do you see?"

# Viz-Verify

I read the screen. Pure procedure — no gates, no approval boundaries.
Gates come from the calling context (e.g., co-loaded two-gate).

## Workflow

1. **Pre-flight.** Run tests before launching the GUI. A crash-on-start
   produces no visible window, and `import -window root` silently
   captures whatever else is on screen (terminal, browser). You'll waste
   cycles analyzing a screenshot of the wrong window. Tests catch the
   crash before the screenshot attempt.

2. **Capture.** Fresh launch always pops to foreground — no need for
   `xdotool` or window management. Launch in background, `sleep 4-6`,
   then capture:
   ```
   DISPLAY=:0 import -window root /tmp/gui_screenshot.png
   ```
   Use programmatic setup scripts (load data, set state, check runs)
   rather than interacting with the GUI manually before capture.
   **If this is a "before" capture for a bug you intend to fix, save
   to a separate path** (e.g. `/tmp/gui_before.png`) so step 6's
   re-capture doesn't overwrite it. The before-screenshot is the
   anchor for the explicit compare in step 7 — losing it means
   re-stashing and re-running the test script later.

3. **Attach + read.** Primary model has native vision. Call
   `vision_analyze(image_path, question="")` to attach the screenshot to
   context — the empty question means "no auxiliary analysis, just
   attach." The agent then **describes what it sees itself** in the
   next response. Delegating to the auxiliary model is wrong: the
   user wants the main agent reading pixels, not a relayed text
   description. If the active model is NOT vision-capable, pass a
   real `question` to get the auxiliary's text analysis.

4. **Map to code.** Use `mcp_codegraph_codegraph_context` for the
   primary call — it returns entry points, related symbols, and
   verbatim bodies in one shot. Fall back to `search_files` for
   visible text/labels and `read_file` for layout context.

5. **Fix.** Apply GUI edits. Stay within scope — no side fixes.

6. **Re-capture.** Re-launch GUI; fresh screenshot to
   `/tmp/gui_screenshot.png` (or a different path than the
   before-capture, so both survive).

7. **Compare.** Attach both screenshots (`vision_analyze` × 2 with
   empty question) and read them yourself. Describe the visual
   diff: which items changed, which stayed the same, any new
   rendering problems. **This comparison IS the visual diff content
   that goes into Gate 2** (when co-loaded with two-gate). The cycle
   is not complete until you have walked through the before/after
   diff explicitly. Do not present Gate 2 on a single
   post-fix screenshot — without the before, the verification
   summary is just "tests pass," which doesn't prove the visual
   bug is fixed.

8. **If unresolved** → return to step 4 (fresh analysis).

## Pitfalls

### Screenshot of Wrong Window — Manifestations

**Black window instead of GUI content:** The GUI window is visible on
screen but shows only a solid black (or empty) rectangle. This means the
test script **created the DPG viewport** (so the OS window is visible) but
**never called `dpg.start_dearpygui()`** (so the render loop isn't
running and nothing is drawn inside the window). Every DPG window that
exists in the DPG context will not render its actual content without
`start_dearpygui()`. Recovery: add `dpg.start_dearpygui()` and use
`dpg.set_frame_callback` for auto-capture/auto-exit.

**Terminal/browser in the frame:** `import -window root` captured the
focused window instead of the GUI. The GUI may have crashed silently on
launch (no viewport created at all), or another window stole focus.
Symptoms: `vision_analyze` describes a terminal session or GitHub page
and says "no dashboard visible." The fix is NOT window management
tools — check the process status, fix the crash, and relaunch.

### Lost "Before" Screenshot

If the before-capture is overwritten before the explicit compare in
step 7, the only recovery is reverting the fix (e.g. `git stash` if the
fix is in a tracked file), re-running the test script to regenerate
the buggy state, capturing again, then unstashing. This costs a full
re-launch cycle and is easy to forget. Prevention: in step 2, save the
before-capture to a path that step 6 won't write to (e.g.
`/tmp/gui_before.png` vs `/tmp/gui_screenshot.png`). If you did
overwrite, the recovery sequence is:
```
git stash
<rerun test script>
cp /tmp/gui_screenshot.png /tmp/gui_before.png
git stash pop
```

### No Window Management Tools

Don't reach for `xdotool`, `xwininfo`, or `wmctrl` — they may not be
installed. Fresh launch is the reliable way to bring the GUI to
foreground.

## Vision Model Note

If the primary model is vision-capable, `vision_analyze` attaches the image
directly to context — the agent reads pixels, not a text relay. If not,
an auxiliary model provides a text description. The workflow is the same
either way, but with a vision-capable primary, step 7 (comparison) can
iterate faster since the agent sees both screenshots directly.

## Fallback: Pixel-Diff When Vision Returns Nothing

Some CLI environments (or model configs) cause `vision_analyze` to return
a brief confirmation like "Image attached natively for the main model"
with **no actual image data the agent can interpret**. The image bytes
are in `/tmp/foo.png` but the model never sees them. In that case, fall
back to programmatic pixel-diff between before/after screenshots:

```python
from PIL import Image
import numpy as np

a = np.asarray(Image.open('/tmp/before.png').convert('RGB'), dtype=np.int16)
b = np.asarray(Image.open('/tmp/after.png').convert('RGB'), dtype=np.int16)
diff = np.abs(a - b).sum(axis=2)
diff_pixels = int((diff > 30).sum())
total = a.shape[0] * a.shape[1]
print(f'Pixels differing: {diff_pixels}/{total} ({100*diff_pixels/total:.2f}%)')
ys, xs = np.where(diff > 30)
print(f'Diff bbox: x=[{xs.min()},{xs.max()}] y=[{ys.min()},{ys.max()}]')
```

Interpretation:
- Diff bbox in the right panel only (x > ~500) = chart updated, left
  panel (process tree) is byte-identical = the fix is in chart state
  only, not in tree rebuild or selection state.
- Diff bbox in the left panel = the change affected tree rendering.
- Diff bbox in the entire image = the change affected layout/global
  state, not what you intended.
- Diff bbox over a single axis region = only that chart updated.

Combine with `dpg.get_item_configuration(tag).get("show")` assertions
on the specific line-series tags (e.g. `line_run_X_proc_Y`) to confirm
*which* series became hidden. This is more decisive than pixel diff
alone and runs in <100ms without ever launching the GUI twice.

---

Viz-verify falls silent.
