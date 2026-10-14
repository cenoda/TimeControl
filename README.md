# TimeControl
Speeds up or slows down Mindustry.

This fork maintains the original JavaScript mod for Mindustry v8, build 160 or
newer. It has been tested against build 160.7.

Supports `x1/256` ~ `x256`. Simply drag the slider to adjust the speed.
Click/tap on 'x#' to shrink the ui. Click/tap on the button to adjust the speed, hold to expand back to full size.

The controls are disabled in multiplayer. The mod changes the duration of each
game update; very high speeds can therefore skip conveyor movement, crafting
cycles, or other simulation work. Use high multipliers with care.

## Install

Find **Time Control** in Mindustry's mod browser, or open
**Mods > Import Mod > Import from GitHub** and enter:

```text
cenoda/TimeControl
```

Restart Mindustry when prompted.

## Changes from upstream

- Targets Mindustry v8 build 160 and newer.
- Avoids mutating Mindustry's shared button style.
- Hides the controls whenever multiplayer networking is active.
- Reapplies the selected speed after loading a world.

Original mod by [sk7725](https://github.com/sk7725/TimeControl), licensed under
Apache-2.0.
