# Faster Space Station Takeoff

Speeds up the scripted flight that carries your ship out of the landing bay
when you take off from a space station or the space anomaly. Built
natively on Linux with the
[AMUMSS Linux port](https://github.com/SavageCore/AMUMSS/tree/feat/linux-support).

## Usage

Install it and pick how much faster the departure is.

| Option | Factor | Patches |
| --- | --- | --- |
| `10x` | 10x | every value below |
| `5x` | 5x | every value below |
| `2x` | 2x | every value below |

The factor is applied to every base value in `src/patches.tsv`, so one number
scales the whole mod.

It cannot remove the departure itself, only compress it. The ship still lifts
off the pad, turns to face out and hands control back to you.

## What it patches

The game keeps the whole departure in the data of the place you are
leaving, in the `GcOutpostComponentData` block of the dock node. The flight
has several parts, and the game ships them all slow.

| Part | Where | What it governs |
| --- | --- | --- |
| the flight | `TakeOffSpeed` and `TakeOffBoost` in each dock node | how fast the ship flies out of the bay, and how fast it boosts on the way |
| the climb | `TakeOffTime` and `TakeOffAlignTime` | how long the ship spends rising off the pad and turning to face out |
| the handover | `PostTakeOffExtraPlayerSpeed` | how fast the ship is going when you get control back |

Every dock node has its own values, so the mod scales each one in place: the
space stations fly out at 500 and boost at 120, the dock node at 100 and 10,
and the anomaly at 300 and 400.

The flight and the climb are scaled by the same factor, which is what keeps
the two in step. The ship turns to face out over a fixed manoeuvre, so a
faster flight only stays pointed the right way if the climb is shortened by
the same factor.

The handover is capped, so the speed you get control back at cannot run away
from the flight that leads into it.

Two things are left alone on purpose. The height the ship climbs to and the
distance it carries on out, because raising those changes the shape of the
exit rather than its speed. And the extra height NPC ships get after they
leave, which is about the traffic in the bay, not about your ship.

Every value lives in `src/patches.tsv`, which is also what `make verify`
checks the build against. A value cannot be patched without also being
checked, and the two cannot drift apart.

## Compatibility

Modifies the six dock entity files of the space stations and the anomaly.
Nothing else.

It does not touch `GCSPACESHIPGLOBALS.GLOBAL.MBIN` at all, so it has no file
in common with Faster Spaceship Docking Rotation Reloaded and the two cannot
fight over it.

It does share its six entity files with Faster Space Station Landing, but
the two change different values in them: the landing mod writes the approach
and landing speed, this one writes the takeoff values. NMS applies EXML
patches value by value, so both take effect at the same time. A mod manager
will still show a file-level conflict between the two, because it compares
files rather than values.

The entity files live in `NMSARC.Precache.pak`.

The dock node and shared ship settings changes also apply to NPC ships
taking off, so station traffic moves faster too.

## Build (on Linux)

Requires a full AMUMSS install at `~/AMUMSS` (override with
`AMUMSS_HOME=...`) with `MBINCompiler-linux` fetched
(`linux/scripts/fetch_mbincompiler.sh` in the AMUMSS repo).

```sh
make release      # clean rebuild + verify + pack dist/ zip (default)
make verify       # sanity-check the built outputs
make clean        # remove build/ and dist/
make assets       # regenerate the Nexus page images (needs Pillow)
```

`make build` clears this mod's folders out of
`$(AMUMSS_HOME)/CreatedMODS`, temporarily stages the variant scripts into
`$(AMUMSS_HOME)/ModScript` (existing content moved aside and restored), runs one
`buildmod.sh --run-pipeline`, and collects the outputs.

### The demo video

`assets/combine.py` joins the three takeoff recordings into one video with
the speed labelled on every segment, and a timer that runs from the frame the
ship leaves the pad to the frame the player gets control back.

```sh
python3 assets/combine.py            # lossless master, 2560x1440, for upload
```

Both ends of the timer are found in the footage rather than by hand. The
start is the frame the centre console stops saying "MAIN ENGINE IGNITION
STARTED", which is the moment the ship leaves the pad. The end is the frame
the scripted flight stops driving the ship: the flight moves most of the
screen every frame, and once control is back the ship coasts and the picture
barely changes.

The video is re-encoded losslessly, so it is bit identical to the
recordings apart from the overlay. The audio is re-encoded, because the
segments have to be joined and the sources are AAC.

### `make verify` is the load-bearing step

For every variant and every patched file it asserts that the EXML was
produced by *this* run, that each expected property carries the expected
value, that the patch touches nothing else, and that there are no marker
tags.

The freshness check exists because of how AMUMSS fails: a change that stops
matching does not error, it silently applies nothing, creates no file, and
leaves the previous build's file sitting in `CreatedMODS`. The copy then
succeeds and every other check happily validates last week's output.

## Release

`make release` creates `dist/Faster Space Station Takeoff.zip`. It is
a FOMOD installer: `fomod/` sits at the zip root, which is how mod managers
detect an installer, and each option is a folder beside it. Import it into
[Amethyst](https://github.com/ChrisDKN/Amethyst-Mod-Manager)/[Vortex](https://github.com/Nexus-Mods/Vortex)
or upload it to Nexus. For a manual install, copy the folder for the option you want into `GAMEDATA/MODS`.

The zip is named after the mod and carries no version, because a mod manager
names the installed mod after the archive file name and shows the version in a
column of its own. The version travels as the version of the Nexus release
instead.

## Game updates / versioning

The mod version is `VERSION` in the `Makefile`. The game version the script
targets is `GameVersion` in `src/takeoff.lua.in`.

When No Man's Sky updates:

1. Fetch the new matching compiler from the AMUMSS repo (use
   `MBINCOMPILER_TAG=vX` to pin):
   `AMUMSS_HOME=~/AMUMSS linux/scripts/fetch_mbincompiler.sh`
   If the AMUMSS core itself changed, re-run the pipeline once so the
   linux patches re-verify (`apply_linux_patches.sh --verify`).
2. Re-derive the bases in `src/patches.tsv` (see below).
3. Bump `GameVersion` in `src/takeoff.lua.in` and `VERSION` in the `Makefile`.
4. `make release`, install, test.

### Re-deriving the bases

The base values are game data, not constants this mod owns. To re-derive them:

```sh
PAK=~/.local/share/Steam/steamapps/common/"No Man's Sky"/GAMEDATA/PCBANKS/NMSARC.Precache.pak
hgpaktool -O /tmp/probe -f "*spacestation/modularparts/entities/*" -f "*spacestation/spacestation/entities/*" \
  -f "*spacestation/spacestationtypeb/entities/*" -f "*nexus/nexus/entities/*" \
  -f "*nexus/nexusexterior/entities/*" "$PAK"
```

Then decompile each MBIN with `MBINCompiler-linux` and read the values in
`GcOutpostComponentData` (`TakeOffSpeed`, `TakeOffBoost`, `TakeOffTime`,
`TakeOffAlignTime`, `PostTakeOffExtraPlayerSpeed`, `TakeOffHeight` and
`TakeOffFwdDist`). Update the `BASE` column in `src/patches.tsv` to match. If
a property is renamed, `make verify` fails on the missing property rather
than shipping a patch that does nothing.
