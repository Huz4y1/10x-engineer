Soldering is heating both parts until the solder flows onto them by itself, not melting solder onto a cold joint and hoping.

Temperature

|Solder|Temp|Notes|
|---|---|---|
|Leaded 60/40|320-350 C|Easiest by far, melts low, flows well. Wash your hands|
|Lead-free|350-380 C|Required commercially, harder to work with, duller joints|
|Fine SMD work|300-320 C|Lower, small tip, be quick|
|Big joints, ground planes|380-400 C|Needs the heat, or the joint never wets|

```
  Too cold  -> solder sits in a ball, doesn't wet, "cold joint"
  Too hot   -> flux burns off instantly, tip oxidises black,
               pads lift off the board
  Just right-> solder flows in and pulls itself into a cone
               within about 2 seconds
```

Tinning, which is not optional

```
  A new or cleaned tip must be coated in solder before use.

  1. heat to temperature
  2. wipe on a damp sponge or brass wool
  3. immediately melt fresh solder onto the tip until it's shiny
  4. do this again every few joints, and ALWAYS before switching off

  An untinned tip oxidises, goes dull grey, and stops transferring
  heat entirely. That's when people crank the temperature up and
  make it worse.
```

The actual technique

```
  1. tip touches BOTH the pad and the leg at once
     ┌────────────┐
     │    iron    │
     └──┬──┬──────┘
        │  │
      ──┴──┴──          heat both for ~1 second
      pad  leg

  2. feed solder into the JOINT, not onto the iron
     ┌────────────┐
     │    iron    │   solder ──▶ ●
     └──┬──┬──────┘              │
        │  │                     │
      ──┴──┴─────────────────────┘

  3. it flows and wets both surfaces

  4. remove the solder first, THEN the iron

  5. don't move it while it cools, ~2 seconds

  Total contact time: 2-3 seconds. If it's taking ten, the tip
  is too cold, too dirty, or too small for the joint.
```

Good versus bad

```
  GOOD                       BAD - cold joint

     ╱▔▔╲                        ●●
    ╱ leg ╲                     ●  ●
   ╱  ▏▏   ╲                   ● ▏▏ ●
  ▔▔▔▔▔▔▔▔▔▔▔                 ▔▔▔▔▔▔▔▔
   shiny, concave              dull, ball shaped,
   volcano shape,              sits on top of the
   wets the whole pad          pad without wetting


  BAD - too much solder      BAD - bridge

      ●●●●●                    ▏▏      ▏▏
     ●●●●●●●                 ▔▔▔▔●●●●●●▔▔▔▔
    ●●●●●●●●●                    two pads joined
   ▔▔▔▔▔▔▔▔▔▔▔                   by a blob
   blob hides the joint,
   can't see if it wet
```

|Joint|Looks like|Fix|
|---|---|---|
|Good|Shiny, smooth, concave sides|-|
|Cold|Dull, grainy, ball-shaped|Reheat with fresh solder and flux|
|Too much|A blob you can't see through|Wick some off|
|Bridge|Two pads joined|Add flux, drag the tip across, or wick it|
|Not wetted|Solder on one side only|More heat, more flux|
|Cracked|A ring around the leg|It moved while cooling. Reheat|

The kit that actually matters

|Item|Why|
|---|---|
|Temperature-controlled iron|A fixed 40W "pencil" iron is why people think soldering is hard|
|0.7mm leaded solder with flux core|Thin enough for detail, thick enough for through-hole|
|Flux pen or paste|Fixes almost every wetting problem instantly|
|Brass wool|Cleans the tip without cooling it, better than a wet sponge|
|Desoldering braid|For removing solder|
|Helping hands or a vice|Two hands are already busy|
|Flush cutters|Trimming legs after|
|Isopropyl alcohol|Cleaning off flux residue afterwards|

Desoldering

```
  Braid, for pads and small joints:

    1. lay the braid over the joint
    2. press the iron onto the braid
    3. the solder melts and wicks UP into the braid
    4. lift braid and iron together, or you'll solder the braid on

    A drop of flux on the braid makes it dramatically better.


  Pump, for through-hole components with a lot of solder:

    1. melt the joint fully
    2. slide the pump nozzle over it
    3. fire it while the solder is still liquid

  Multi-leg through-hole parts are genuinely hard to remove.
  Often it's easier to cut the legs, remove the body, then
  take each leg out on its own.
```

Soldering header pins to an ESP32 board

```
  1. push the headers into a breadboard, plastic side down
  2. sit the board on top, so everything is square
  3. solder ONE corner pin, check it's still flat, adjust
  4. solder the opposite corner
  5. then do the rest

  Soldering all of one side first tilts the board permanently.
```

Safety

```
  - the tip is 350 C. It will not hurt for the first half second,
    and that is the dangerous part
  - solder SPITS when flux boils. Eye protection, genuinely
  - flux fumes are an irritant, not lead vapour. Open a window
    or use a small fan blowing the smoke sideways
  - lead is absorbed by ingestion, not fumes. Wash your hands
    before eating and don't chew the solder
  - iron back in the stand every single time. Never on the bench
  - a hot iron looks exactly like a cold one
  - don't solder anything plugged in and powered
```

The gotcha

```
  If solder refuses to flow onto a pad:

    it is almost always FLUX, not heat.

  Flux removes the oxide layer so molten solder can bond to metal.
  Flux-core solder has some, but it burns off in a second, so on
  a reheated joint there's none left. Add fresh flux and the same
  joint that fought you for a minute takes two seconds.

  Second suspect: the joint is bigger than your tip can heat.
  A ground plane sucks heat away faster than a small conical tip
  supplies it. Use a chisel tip and more contact area.
```

---

**Where it's used:** [[20 — EMBEDDED]] · [[Project 008 — Physical Intelligent Engine Monitor]]
