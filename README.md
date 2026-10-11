# PixInsight Processing (no server)

A Claude skill that takes an integrated astrophotography master (for example a
WBPP `masterLight` XISF) from linear data to finished files in PixInsight:
background removal, color calibration, stretch, star separation, noise
reduction, curves, star recombination and export.

It needs nothing installed inside PixInsight and no MCP server. Claude drives
the PixInsight desktop app through its computer-use ability, and does the image
work by writing short PJSR (PixInsight JavaScript) scripts and running them.

This plugin is an independent community project. It is not affiliated with or
endorsed by Pleiades Astrophoto, the makers of PixInsight.

## What you need

- PixInsight 1.9.x, licensed and already running on your computer.
- The Claude desktop app with computer use turned on, and permission for
  Claude to control PixInsight when it asks.
- Optional: the StarNet2 module, for star separation. Without it the skill
  processes the image with stars in place.

Developed and tested on macOS with one-shot-color masters. Windows and Linux
are untested; the scripts are the same but the window handling may differ.

## How to use it

Ask Claude something like:

- "Process the master light in `/path/to/master` in PixInsight."
- "Stretch and color calibrate this integrated image, and save a 16-bit TIFF."
- "Make a starless version too."

Claude picks the master file, works through the stages one at a time, looks at
the image after each, and saves new files next to the source. It reports any
step that failed or needed a fallback.

## What the plugin does on your computer

Described in full so there are no surprises:

- **Controls PixInsight.** It clicks, types and uses menus in PixInsight only.
- **Detaches two panels.** The Script Editor and Process Console are turned
  into floating windows, because that makes them reliable to control.
  Double-click their title bars to dock them again.
- **Writes one script file.** PixInsight saves the working script, by default
  as `NewScriptFile1.js` in your home folder. It is overwritten at each stage
  and is safe to delete afterwards.
- **Writes image files.** Results are saved next to the source master under new
  names (`<name>_processed.xisf`, `.tif`, `.jpg`, and `_starless` variants). It
  never overwrites the source or an earlier result; redo versions get a suffix
  such as `_v2`.
- **Network.** The plugin sends nothing anywhere and downloads nothing.
  PixInsight's own color calibration tool may read its locally installed Gaia
  catalog files.
- **No shell access.** The skill tells Claude not to use the PixInsight
  console's shell escape.

The plugin contains only Markdown instructions: no code that runs on install,
no hooks, no MCP servers.

## Limitations

- Controlling a desktop app in the background is slower and more fragile than
  a script bridge. If you use the computer heavily while Claude works, some
  steps may need retrying.
- The curve and noise-reduction settings are starting points taken from a
  small number of targets (an emission nebula and a large galaxy). Expect to
  ask for adjustments.
- Only stock PixInsight tools plus StarNet2 are used. It does not use
  BlurXTerminator, NoiseXTerminator or StarXTerminator.

## Install

From Claude Code:

```
/plugin marketplace add dividor/astrobagel-pixinsight
/plugin install astrobagel-pixinsight@astrobagel
```

Or download the `.plugin` file from the releases page and add it in Claude
under Customize, Plugins.

## License

MIT. See `LICENSE`.
