---
name: viz-verify
description: Visual verification workflow. Screenshot → analyze → map
  to code → fix → re-capture → compare. Pure procedure — no gates.
category: lambda-skills
version: 0.3.0
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

3. **Analyze.** `vision_analyze` — identify every visual problem.

4. **Map to code.** `search_files` for visible text/labels to find widget
   creation sites. `read_file` for layout context.

5. **Fix.** Apply GUI edits. Stay within scope — no side fixes.

6. **Re-capture.** Re-launch GUI; fresh screenshot.

7. **Compare.** `vision_analyze` with: "Compare to the before screenshot
   (from step 1). Is [issue] resolved? Any new rendering problems?"

8. **If unresolved** → return to step 4 (fresh analysis).

## Pitfalls

### Screenshot of Wrong Window

`import -window root` captures the focused window. If the GUI crashed
silently on launch, the screenshot shows your terminal or browser. The
symptoms are: `vision_analyze` describes a terminal session or GitHub
page and says "no dashboard visible." The fix is NOT window management
tools — the GUI simply isn't running. Check the process status, fix
the crash, and relaunch.

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

---

Viz-verify falls silent.
