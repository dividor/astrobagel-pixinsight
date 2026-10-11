# Driving PixInsight reliably

This is the part that costs the most time if done wrong. The recipe below was worked out on macOS with PixInsight 1.9.x, controlling the app in the background. On other systems the same ideas apply but verify each step with a screenshot.

Use the background app tools (per-window screenshot, click, type, key, menu, drag, list windows). Tool names differ between Claude surfaces; use whichever computer-use tools the session provides.

## One-time setup per session: detach two panels

The Script Editor and Process Console are auto-hide panels on the left edge of the main window. While docked they close unpredictably between and during actions, and keystrokes then land on the image workspace. Detach both first:

1. Click the panel's tab on the left edge (the tab toggles the panel open and closed) and, in the same batch of actions, double-click the panel's title bar. It becomes a separate floating window.
2. List the app's windows to get the two new window ids. Floating windows expose real accessibility text fields, which is what makes the steps below dependable.
3. Drag the floating windows by their title bars toward one side so they do not cover the image windows or each other. In particular the console's command line must not sit underneath the editor.

Floating windows are hidden while PixInsight is not the active application, so they can drop out of the window list. When that happens, use the Execute Script File route below, which does not need them.

## Writing a script

- Write each script as ONE line, statements separated by `;`. Keep a copy and know its length.
- In the floating Script Editor window, set the whole text in a single action: type into the text area in replace mode, allowing the existing content to be overwritten. This is atomic. Typing character by character drops characters and can be cut off midway.
- Setting the text this way does not mark the document as modified, so Save would do nothing. Click in the text area, press `end`, press `space`, then click the Save toolbar button. The status bar must read `File saved`, and the column indicator should equal the script length plus two.
- The very first save of a new script opens a system Save dialog in which typing is blocked. Click Save and accept the default name and location (normally `NewScriptFile1.js` in the user's home folder). Reuse that one file for every script, and tell the user about it at the end.

## Running a script

Preferred: the menu item `Script > Execute Script File...`. It opens PixInsight's own file dialog (window title "Execute Script"). Set the File name field to the script's full path and press Open.

Alternative: in the floating Process Console, set the command-line field to `run -x <full path to script>` and press `return`.

Before running anything a second time, read the console. A command that seemed lost may already have run, and re-running a stretch or curves stage applies it twice.

## Reading results

Take a screenshot of the Process Console window. Have every script print one tagged line per step, for example `console.writeln("SPCC_RESULT "+P.executeOn(v))`. `executeOn` returns false on failure rather than throwing, so always print it.

To list a folder, enter `ls "<path>"` on the console command line.

To compare or inspect images, set a zoom-out factor on the image window from a script (`w.zoomFactor=-8`) and take a screenshot of the main window.

## What does not work in the background

- `Script > Run Script from Editor` is almost always disabled. Do not rely on it.
- `cmd+a` in the docked editor is unreliable, and `shift+cmd+down` cycles image windows instead of selecting text.
- The editor's own in-window menus, pop-up menus and function keys cannot be opened or sent.
- Process tool windows (color calibration graphs, tool dialogs) float over the workspace and are hard to close. Do not open process dialogs; instantiate processes from scripts instead.
- Resizing floating windows by dragging their edges does not work; moving them by the title bar does.

## Safety habits

- Verify the script text (status bar column count) before every run.
- Never press `return` in the console without having just checked what is on the command line.
- If a click or typed text may have gone to the image workspace rather than a text field, check the image and its history, and mention it in the final report.
- If actions keep failing because the user is actively using the computer, pause and tell the user rather than retrying blindly.
