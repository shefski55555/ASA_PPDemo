# ASA Custom Stencil Bitwise Manipulation

## Overview

This repository demonstrates how to write custom data to the CustomStencil buffer without breaking the Tek Helmet "ESP" effect.

Important: in the devkit, r.customdepth seems to be set to 1 by default. Use the command r.customdepth 3 to set it to the needed value. (This is done automatically in live from amy testing)

If you need a more general starting point on PP buffs I started this off of Quellcrest's example [here](https://discord.com/channels/153690873186484224/623923912308424714/1175917589495558205)

You can manually give yourself the buff from this demo using this command ```cheat ForceGiveBuff "Buff_dinooverlay" true``` and then just spawn any dino.

---

## The Problem

The Tek Helmet reads the CustomStencil value on each dino mesh to determine outline colour and brightness. This value is not a simple ID — it is a **packed bitfield** encoding multiple pieces of data into a single byte.

Naively calling `Set Custom Depth Stencil Value` on a dino mesh overwrites this packed data entirely. The Tek Helmet's post process material then reads incorrect values, causing wrong outline colours or no outline at all.

---

## The Tek Helmet Bit Layout

The Tek Helmet's ESP Material (ESPMaterialOutline) expects to be able to read the stencil value in the following format.

H = health value (used for the saturation of the outline)
C = category value (used to decide whether the outline is red/green/yellow/white)


```
Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0
  H       H       H       H     FREE     C       C       C
```

| Section | Bits | Range | Description |
|---|---|---|---|
| **Category** | 0-2 | 0-7 | Dino outline colour (Tek Helmet uses 0-4) |
| **Free Bit** | 3 | 0 or 8 | Not read by any known Tek Helmet function |
| **Health** | 4-7 | 0-15 | Outline brightness (scales 0.5-1.0 saturation) |

The Tek Helmet colour categories are:
- `0` = Black (none)
- `1` = Green
- `2` = White
- `3` = Yellow
- `4` = Red
- `5-7` = Cyan (unused — available with cyan outline side effect)

---

## Reconstructing The Stencil Value

I tried reading and storing the stencil value created by the game but ran into an issue getting up to date values. I beleive this is due to it not writing these stencil values if you have a buff active with "Complete Custom Depth Stencil Override" active, which you need to write your own values. Instead I've opted to fully recreate the stencil value for any dino where I am also editing the value.

---

## The Three Blueprint Functions

Inside the "Buff_DinoOverlay" buff there are three Blueprint functions, each encoding custom mod data into a different section of the stencil byte. All three take `Packed Bits`, and `Dino` as inputs so the full stencil value is always correctly reconstructed. Essentially we recreate the "vanilla" stencil value and then edit it.

Each function also contains the logic for overwriting 1 section of the stencil data. Currently the buff sets values based on if a dino is male/female.

The stencil value is a single byte (0-255). Each function's comment documents the recommended min/max values to ensure the combined result stays within range.

### 1. `CreateStencilValueCategoryBits`
Writes custom data into **bits 0-2**, replacing the colour category.

- Side effect: Tek Helmet outline will appear **cyan** (values 5-7 hit the unhandled branch)

### 2. `CreateStencilValueHealthBits`
Writes custom data into **bits 4-7**, replacing the health value.

- Side effect: Tek Helmet outline brightness will reflect your value rather than actual health (personally I'd recommend using values on the larger end of what's recommended (13/14/15) as this will make the outline most saturated)

### 3. `CreateStencilValueFreeBit`
Writes custom data into **bit 3**, the free bit not read by the Tek Helmet. - I'm not sure if it's used elsewhere but for the Tek Helmet it seems to be unused.

- Side effect: **none known**
- Limitation: binary only (on/off)

---

## The Material

The post process material **PP_DinoOverlay** contains the corresponding material functions for reading each of the three bit sections from the CustomStencil buffer and returning a value your material graph can use to drive visual effects.

Note that `SceneTexture:CustomDepth` and `SceneTexture:CustomStencil` must be connected as input pins on any custom node that samples the stencil — this registers the scene textures with the shader compiler and is required for `SceneTextureLookup` to be available.


For example, this is the function for health bits.

```hlsl
{
float2 UV = GetDefaultSceneTextureUV(Parameters, PPI_CustomDepth);
int TrueValue = (int)(StencilValue * 255.9999f);
int StateBits = (TrueValue >> 4) & 0x0F;

if(StateBits == 13) return 5.0f;
if(StateBits == 14) return 6.0f;

return 0.0f;
}
```

`if(StateBits == 13) return 5.0f;` -> this checks for whatever value you passed into the stencil from the function in the buff and outputs 5.0/6.0. This value is then used by the if node to decide the colour of the overlay. 5.0 and 6.0 can be any number you want, add more if statements if you want more options.
