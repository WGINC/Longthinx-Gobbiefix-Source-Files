# Longthinx gobbiefix: notes on the fixes, with code

*日本語版は [README.ja.md](README.ja.md) をご覧ください。*

## To SQUARE ENIX's Creative Studio 3

If you would like to put any of Dellingr's code or ideas from gobbiefix into FINAL FANTASY XI, please feel free to.
All we ask is that you credit **Dellingr and Geno**. If you can't credit us directly for legal reasons, a
tongue-in-cheek thank-you to the **"Beastmen Alliance"** will do.

The licence ([LICENSE](LICENSE)) says the same in legal terms. It gives Square Enix, its group companies and its
contractors MIT No Attribution terms: you may use, change and ship this code without conditions, so the credit is a
request, not a requirement. Everyone else may use it only under GPL-3.0-only. The licence covers Dellingr's own work
only.
The ideas from other people's mods are named where they are used and in [Credits](#credits). Their code isn't in this
repository.

This document describes what each fix in gobbiefix changes in the PC client and why, with the code that does it.
None of it is your code. gobbiefix changes the running game from outside, so every fix here is written as a hook
around one of the client's functions. With the real source, most of these fixes become a few lines in the function
itself.

### How to reach us

If you work at SQUARE ENIX and would like to get in touch, whether about using any of this or to receive the full
version with every address and offset, please use this repository's private reporting. Open the **Security** tab and
choose **Report a vulnerability** ([direct link](https://github.com/WGINC/Longthinx-Gobbiefix-Source-Files/security/advisories/new)). Only the repository's maintainers can read what you send.
Despite the form's name, you don't need to be reporting a vulnerability: put "Contact" in the title and write freely.

For the full version, please include a SQUARE ENIX company email address in your message. We'll write to it to
confirm it's you, then give your GitHub account read-only access to the private repository.

## How to read this

- **Addresses** are in the FFXiMain.dll build our analysis calls `6A995428`, at its preferred base `0x10000000`. Unless
  a section says otherwise, every address, offset and constant was read from that build's code.
- **Function names** like `Shadow_DrawByMode` or `CXiActor::NameCalc` are names we made up. Your names will differ. The
  addresses identify the functions.
- **The hook pattern.** In the code, `prev_x` is the game's original function, which our hook calls. `orig::X(...)`
  calls one of the game's functions directly. `XI_GAME(a)` turns an address in that build into the address in the
  running process.
- **Code shown.** All code here is gobbiefix's own. Where a fix lives inside a rewritten copy of one of your
  functions, only the lines we added are shown, never the rest of the function. Snippets are abridged with `...` where
  marked.
- **Placeholders.** This public version leaves out the game's data addresses, structure offsets, single-instruction
  sites and patch bytes. Names like `g_shadowSize`, `ACTOR_BOX_CORNERS(actor)` or `RET_IN_RESTART` stand in for them.
  Function addresses are given only where the gobbiefix release already lists them. A full version with every value
  is available to SQUARE ENIX on request.
- **What's known and what isn't.** Each section says where a number came from. Where we don't know something (often
  what the PS2 did exactly), the section says so.

## The fixes at a glance

| # | Fix | On the PC, without it | Where it belongs | Setting (default) | In the game |
|---|---|---|---|---|---|
| 1 | Ground decals | Circles stretched; their size follows the field of view | `BeginEffectOffscreenPass` | `decal_fix` (on) | Confirmed by our tester |
| 2 | Round shadows | Every character's shadow the same size | `Shadow_DrawByMode` | `fitted_shadows` (on) | Seen working; the size scale isn't measured |
| 3 | Name sizes | Names pulse in size as you move | `CXiActor::NameCalc` | `smooth_names` (on) | Seen working |
| 4 | Lamp glows | Glows flicker behind their lamp | The glow's draw call sites | `flare_fix` (on) | Confirmed by our tester |
| 5 | Sphere-mapped metal | Dull next to the PS2 | `CYyModelDt::SkinAndDrawV2` | `sphere_shine` (on) | Not tested in the game |
| 6 | Aura capture size | Jagged auras | `Post_CreateScreenEffectTargets`, `Post_CaptureSceneToTarget` | `aura_size` (1024) | Confirmed at 2048 |
| 7 | Aura spread, soft last copy | Copies bunched on large screens; a hard edge | `Post_DrawRotatingGhostLayers` | `aura_soft` (on) | Used in the confirmed aura runs; not tested on its own |
| 8 | Stronger auras (new) | — | `Post_DrawRotatingGhostLayers` | `aura_boost` (0, off) | Seen working |
| 9 | Aura afterimages (new) | — | `Post_DrawRotatingGhostLayers` | `aura_trail` (0, off) and `aura_trail_*` | Seen working; the latest changes not yet run |
| 10 | Disintegrate | The PS2's break-apart effect missing | — (portfix's method) | `disintegrate` (1) | Both versions confirmed by our tester |
| 11 | Looping sounds | Rain and wind stop for good | `CYySoundElem::UpdateTimerAndPosition` | `sound_retry` (on) | Not confirmed: in the one run measured it never acted |
| 12 | 60 fps limiter | 58.8 fps | `Frame_Render`'s limiter | `precise_pacing` (speedfix.ini, on) | Measured: active; frame-to-frame unevenness 15.2% → 11.1% |
| 13 | Game time and frame rate | Slow below 60 fps, fast above | `Frame_Render`, `Camera_GetFrameScale` | `frame_step` (speedfix.ini, 3) | Measured: game time follows real time from 31 to 184 fps |
| 14 | Even frame cap | — | At Present | `frame_cap` (0, off) | No difference seen on screen at 120 against no cap |
| 15 | SSE on every CPU | x87 paths on CPUs whose vendor string doesn't match | The CPU detection, `CreateProcessorInstance` | `cpu_tier`, `sse_skinning` (speedfix.ini, on) | Seen in the logs: SSE tier with it on, x87 with it off |
| 16 | Occlusion readbacks | The CPU waits for the GPU | The occlusion readback | `occlusion_late` (speedfix.ini, on) | In use; pop-in, its known side effect, seen; its cost not measured in the game |
| 17 | Renderer texture tables | — | d3d8on11 | — | Measured: 0.72–0.88 µs → 0.34–0.36 µs per draw |

- **Setting:** the name in `gobbiefix.ini`, or in `speedfix.ini` where marked. Turning a setting off gives the game's
  own behaviour, so you can compare before and after in the game.
- **In the game:** what has been seen or measured in a real game session. "Confirmed by our tester" means the person
  testing gobbiefix saw the fix working. Measured figures come from gobbiefix's own logs in those sessions.
- **Mode 3** of `frame_step`, the default, goes one step past modes 1 and 2 described in section 13: the five loops
  that run once per whole tick (the camera's follow, the actors' position and turning) also take the fraction.

---

## Visual fixes

### 1. Ground decals: round and at their own size (`decal_fix`)

**Symptom.** Summoning circles and ground-targeted spell circles are stretched into ellipses, and their size changes
with the field of view.

**Cause.**
- `BeginEffectOffscreenPass` binds the 256×256 accumulation target and sets VIEW
  (`D3DXMatrixLookAtRH`, 2 units off the effect) and WORLD (the effect's matrix with a 0.25 scale).
- It sets **no PROJECTION**. The decal is drawn into the square target with the scene's perspective projection, so its
  x and y scales differ by the screen's aspect ratio, and both follow the scene's field of view.
- `EndEffectOffscreenPass` (0x10054F80) then has `DrawProjectedTexture` (0x10182700) map the target onto the ground with
  size 10: `u = (x − px) / 10 + 0.5`.

**The arithmetic.** A point at distance `d` from the effect's centre is drawn at NDC `s · 0.25 · d / 2`, where `s` is
the projection's `_11` or `_22`. It lands on the ground at `5 · NDC = (5s/8) · d`. It lands at `d` itself, one to one,
when **s = 8/5 = 1.6**, in both axes. One player's screen had `_11 = 1.7032` and `_22 = 2.5729`, which gives an
ellipse 1.06 d across and 1.61 d along.

**Fix.** Use a projection with `_11 = _22 = ±1.6` for the offscreen pass, and restore the scene's at the start of
`EndEffectOffscreenPass`. We hook `CMoDX::PushViewMatrix` and act only on its call from Begin (identified by its return address,
`RET_FROM_BEGIN_PUSHVIEW`), because that call comes after the target is bound:

```cpp
static void __thiscall sf_pushview(void *self) {
  prev_pushview(self);
  if ((uintptr_t)__builtin_return_address(0) != RET_FROM_BEGIN_PUSHVIEW || !sf_cfg.decal_fix || sf_decal_set) return;
  void *cdx = CDX_OF(g_pResourceMng);
  float p[16];
  orig::CDx_GetTransformCached(cdx, 3, sf_decal_saved);           // 3 = D3DTS_PROJECTION
  for (int i = 0; i < 16; i++) p[i] = sf_decal_saved[i];
  p[0] = p[0] < 0 ? -1.6f : 1.6f; p[5] = p[5] < 0 ? -1.6f : 1.6f; // _11, _22; depth terms kept
  orig::CDx_SetTransform(cdx, 3, p);
  sf_decal_set = 1;
}
static float * __stdcall sf_effend(float *a1, int a2) {           // EndEffectOffscreenPass
  if (sf_decal_set) { orig::CDx_SetTransform(CDX_OF(g_pResourceMng), 3, sf_decal_saved); sf_decal_set = 0; }
  return prev_effend(a1, a2);
}
```

*In your source:* set that projection in `BeginEffectOffscreenPass`, next to its VIEW and WORLD.

**Not known:** whether the PS2 drew these at exactly their model's size. The PC's constants only say so if the
pipeline was meant to be one to one. Dellingr's earlier fix (dagnyfix, from a prototype by Daleterrence) found 1.529
by eye, within 4% of 1.6.

### 2. Round shadows sized to each character (`shadow_fit`)

**Symptom.** Every character's simple round shadow has the same size: a Galka's is the same as a Tarutaru's.

**Cause.**
- `Shadow_DrawByMode` (0x10033DA0) scales its ground quad (±1 in x and z) by
  `g_shadowSize / 3`.
- That one global is shared by every character. It starts at 1.0, then holds whatever 0x100CBDE0 last wrote from
  *one* character's extent.

**Fix.** Set that global for each character's own call and put it back afterwards. The character's size comes from
the eight world-space corners of its box (`ACTOR_BOX_CORNERS`), which the actor's OnMove writes each tick.
- The footprint's diagonal (the longest x-z distance between corners) doesn't change as the character turns.
- The projected shadow (branch 2) gets twice the number. `DrawProjectedShadow` (0x10184FE0) maps its texture over `s`,
  while the ground quad spreads it over `2s`. Without that factor, a shadow halves in width when its caster comes
  within 20 units.

```cpp
static undefined4 __thiscall sf_shadow(void *self_, float *pos) {
  ...
    if (br <= 2) {                                        // the round-shadow branches
      const float *c = ACTOR_BOX_CORNERS(self_);                                    // the box's 8 corners
      float d2 = 0;
      for (int i = 0; i < 8; i++) for (int j = i + 1; j < 8; j++) {
        float dx = c[4 * i] - c[4 * j], dz = c[4 * i + 2] - c[4 * j + 2], e = dx * dx + dz * dz;
        if (e > d2) d2 = e;
      }
      if (d2 >= 0.05f * 0.05f && d2 <= 20.0f * 20.0f) {   // a box is there
        float d; __asm__("fsqrt" : "=t"(d) : "0"(d2));
        volatile float *g = &g_shadowSize; float keep = *g;
        *g = (br == 2 ? 2.0f : 1.0f) * 3.0f * d * (float)sf_cfg.shadow_fit_scale / 100.0f;
        undefined4 r = prev_shadow(self_, pos);
        *g = keep;
        return r;
      }
    }
  ...
  return prev_shadow(self_, pos);
}
```

**Not known:** the right scale. `shadow_fit_scale = 46` (a half-width of 46% of the footprint's diagonal) is a
starting value, not a measured one. The 0.05–20 guard range was also chosen, not measured.

### 3. Names that shrink smoothly with distance (`name_smooth`)

**Symptom.** Name plates pulse in size as you move. This happens in the unmodified game too.

**Cause.**
- `CXiActor::NameCalc` (0x100830C0) sizes the name from its depth after `_ftol` has truncated
  `z · 4096` to an integer.
- Between the near and far limits (4020 and 4100 in the build we read them from) the
  size is `(4100 − depth) / 80`.
- So a name's size moves in 80 visible steps.

**Fix.** After the game's function, repeat the same projection with the game's own functions in the same order. Only
if its truncation equals the integer the game stored, which checks that it's the same number, replace the size with
the unrounded value:

```cpp
static uint32_t __thiscall sf_namecalc(void *self, undefined4 *pos, int out, float *scale, uint32_t *color) {
  uint32_t r = prev_namecalc(self, pos, out, scale, color);
  ...
  long double zf = (long double)v[2] * 4096.0L;                     // the same depth, before truncation
  if ((int32_t)zf != o[2]) return r;                                // not the game's number: keep its size
  long double sz = ((long double)farz - zf) / (long double)(int32_t)(farz - nearz);
  *scale = (float)(sz < 0 ? 0 : sz > 1 ? 1 : sz);
  return r;
}
```

*In your source:* compute the size from the depth before it's converted to an integer.

### 4. Lamp glows in front of their lamp (`flare_fix`)

**Symptom.** Lamp and lantern glows flicker in and out behind the lamp's own surface.

**Cause and fix.** This one comes from Dellingr's dagnyfix (flarefix v3.1), which found the glow draws by their
signature:
- `DrawPrimitive` from a 36-byte-stride vertex buffer, at most 8 vertices;
- depth test on, depth write off;
- blending `SRCALPHA / ONE`.

dagnyfix moves each vertex's local z by −0.4. gobbiefix makes the same move through WORLD for that one draw, with no
lock and no copy: `WORLD' = T(0, 0, −0.4) × WORLD`.

```cpp
static bool flare_match(D3DPRIMITIVETYPE t, UINT prims) {
  UINT nv = t == D3DPT_TRIANGLELIST ? prims * 3 : t == D3DPT_TRIANGLESTRIP || t == D3DPT_TRIANGLEFAN ? prims + 2 : 0;
  if (!nv || nv > 8 || !g_ssV[0] || g_ssStride[0] != 36 || fvf_rhw(g_curVS)) return false;
  static const D3DRENDERSTATETYPE k[5] = {D3DRS_ZENABLE, D3DRS_ZWRITEENABLE, D3DRS_ALPHABLENDENABLE, D3DRS_SRCBLEND, D3DRS_DESTBLEND};
  static const DWORD want[5] = {D3DZB_TRUE, FALSE, TRUE, D3DBLEND_SRCALPHA, D3DBLEND_ONE};
  for (int i = 0; i < 5; i++) if (g_rs[k[i]] != want[i]) return false;   // (abridged: "state known" checks removed)
  return true;
}
```

*In your source:* the offset belongs in the glow's own draw, at its call sites (which gobbiefix counts but hasn't yet
fixed at the source).

### 5. Sphere-mapped metal at the PS2's shine (`sphere_shine`)

**Symptom.** Sphere-mapped metal looks dull on PC compared with the PS2.

**Cause.**
- The sphere-mapped batch in `CYyModelDt::SkinAndDrawV2` puts the environment texture in stage 1, with
  `MODULATEALPHA_ADDCOLOR` at `TCI_CAMERASPACENORMAL`. It reads the map at `(nx, ny)` and adds it at a 0xFF = 1.0
  scale.
- voetre's portfix measured the PS2 in PCSX2: `u = 0.5 + 0.5·nx`, `v = 0.5 − 0.5·ny`, added at a 0x80 = 1.0 scale,
  which is twice as strong.

**Fix.** gobbiefix's own version of portfix's technique, set up in the game's own batch:
- a texture matrix for the PS2's mapping;
- stage 1 writes `2 × map × alpha` to TEMP;
- stage 2 adds TEMP to the colour.

The states are read first and put back afterwards.

```cpp
static const struct { DWORD st; D3DTEXTURESTAGESTATETYPE ty; DWORD val; } kSmSet[10] = {
  {1, D3DTSS_COLOROP, D3DTOP_MODULATE2X}, {1, D3DTSS_COLORARG1, D3DTA_CURRENT | D3DTA_ALPHAREPLICATE}, {1, D3DTSS_RESULTARG, D3DTA_TEMP},
  {2, D3DTSS_COLOROP, D3DTOP_ADD}, {2, D3DTSS_COLORARG1, D3DTA_CURRENT}, {2, D3DTSS_COLORARG2, D3DTA_TEMP},
  {2, D3DTSS_ALPHAOP, D3DTOP_SELECTARG1}, {2, D3DTSS_ALPHAARG1, D3DTA_TEMP}, {3, D3DTSS_COLOROP, D3DTOP_DISABLE}, {3, D3DTSS_ALPHAOP, D3DTOP_DISABLE}};
// u = 0.5 + 0.5 nx, v = 0.5 - 0.5 ny (the PS2's mapping, as portfix measured it)
static const D3DMATRIX kSmPs2 = {{{0.5f, 0, 0, 0, 0, -0.5f, 0, 0, 0, 0, 0, 0, 0.5f, 0.5f, 0, 1}}};
```

### 6. Smooth auras: a bigger capture (`aura_size`)

**Symptom.** Actor auras look jagged.

**Cause.**
- `Post_CaptureSceneToTarget` (0x10038A80) copies the scene into `g_sceneCaptureTex` as a 256×256 quad.
- `Post_DrawRotatingGhostLayers` (0x10038E10) draws it back at the viewport's size, so the aura is a 256-pixel picture
  blown up.
- Nothing else in the code depends on 256. The target is bound with no depth surface and a viewport from its own
  size (`CDx::PushRenderTargetTexture`, 0x10009F20).

**Fix.** Create the texture larger in `Post_CreateScreenEffectTargets` and draw the capture's quad at that size. Our
added lines, in that function:

```cpp
unsigned short n = xi_aura_size >= 0x100u && xi_aura_size <= 0x1000u ? (unsigned short)xi_aura_size : 0x100u;
g_sceneCaptureTex = (int)orig::CDx_CreateTexture(g_pCDx, n, n, &g_sceneCaptureDepth, 1, 0);
if ( g_sceneCaptureTex == 0 && n != 0x100u )        // not made at that size: the game's
  g_sceneCaptureTex = (int)orig::CDx_CreateTexture(g_pCDx, 0x100u, 0x100u, &g_sceneCaptureDepth, 1, 0);
```

gobbiefix's default is 1024. The stereo twin (`g_sceneCaptureTexEye2`) stays at 256. The idea of enlarging this
texture comes from Krauerlabs's SpectralFix, which Dellingr's dagnyfix ports. gobbiefix makes the change at its source
instead of matching draws from outside.

### 7. Smooth auras: spread and a soft last copy (`aura_soft`)

**Cause.** Two things in `Post_DrawRotatingGhostLayers`:
- The rotated copies are offset by `sin/cos(angle) × radius × 0.0625` **pixels**, the same on every screen, so on a
  large screen the copies bunch up.
- The last, unshifted copy is drawn opaque through an alpha test (`ALPHAREF 0x60`, blending off), which gives it a hard
  edge.

**Fix.** These are SpectralFix's two picture changes (Krauerlabs), made in the game's own function:
- scale the offsets by `render-target width / 1024`, kept between 1 and 16;
- draw the last copy blended at 25%: blending on, alpha test off, alpha = texture × diffuse with the diffuse alpha
  at 64.

Our added lines:

```cpp
if ( xi_aura_soft != 0 )                    // the spread
{
  float sp = (float)RT_WIDTH(orig::CDx_GetRenderTargetTop_v1(g_pCDx_as_v1)) / 1024.0f;
  sp = sp < 1.0f ? 1.0f : sp > 16.0f ? 16.0f : sp;
  fStack_a4 *= sp; fVar11 *= sp;            // the copy's x and y offsets
}
...
if ( xs_comp )                              // the last copy at 25% (states read first, put back after the draw)
{
  xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_ALPHABLENDENABLE, TRUE); xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_SRCBLEND, D3DBLEND_SRCALPHA);
  xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_DESTBLEND, D3DBLEND_INVSRCALPHA); xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_BLENDOP, D3DBLENDOP_ADD);
  xs_d->lpVtbl->SetRenderState(xs_d, D3DRS_ALPHATESTENABLE, FALSE);
  xs_d->lpVtbl->SetTextureStageState(xs_d, 0, D3DTSS_ALPHAOP, D3DTOP_MODULATE); xs_d->lpVtbl->SetTextureStageState(xs_d, 0, D3DTSS_ALPHAARG1, D3DTA_TEXTURE);
  xs_d->lpVtbl->SetTextureStageState(xs_d, 0, D3DTSS_ALPHAARG2, D3DTA_DIFFUSE);
  // each of the quad's four vertex colours gets alpha 0x40 (64)
}
```

### 8. New: stronger auras (`aura_boost`)

This is gobbiefix's own feature, not the PS2's or SpectralFix's. Each notch (0–5) makes the copies' offsets 25% wider
and their alpha 25% stronger:

```cpp
if ( xi_aura_boost != 0 )
{
  float b = 1.0f + 0.25f * (float)xi_aura_boost;
  fStack_a4 *= b; fVar11 *= b;
  uint32_t a = (uStack_a0 >> 24) * (4 + xi_aura_boost) / 4; if (a > 255) a = 255;
  uStack_a0 = (uStack_a0 & 0xFFFFFFu) | (a << 24);
}
```

### 9. New: aura afterimage trails (`aura_trail`)

This is also gobbiefix's own feature. Up to 50 afterimages of the aura follow a moving actor, each a set distance
behind the one ahead. When the actor stands still the trail pulls in to one faint image, so a resting actor doesn't
look messy.

**Where it hooks in.**
- CXiSkeletonActor's draw calls the aura functions with a pointer inside the actor.
- The actor's world position is the first bone's translation in the skinning palette (`g_boneWeightMatrixTable`).
  The palette is still this actor's at that point, because its models were drawn just before.
- `GetPos` hardly changes as the actor moves, so it can't be used.

Each afterimage is a copy of the capture texture, recorded with its world position and VIEW×PROJECTION. It's drawn
where its world position is on screen *now*, scaled by `(P._11 / w_now) / (P._11_then / w_then)`, so it stays put
when the camera moves:

```cpp
float ax, ay, wn, wo;
if (!now_ok || !to_screen(vp, s.pos, rt_w, rt_h, ax, ay, &wn) || !to_screen(s.vp, s.pos, rt_w, rt_h, tx0, ty0, &wo)) continue;
float sc = (p._11 / wn) / (s.p11 / wo);   // (abridged: range checks removed)
float fade = fade_of(k, (int)n, now - s.at, lone, 0.35f, 0.8f) * (float)sf_cfg.aura_trail_alpha / 100.0f;
d->SetTexture(0, s.tex);
quad(d, ax - tx0 * sc, ay - ty0 * sc, ax + (rt_w - tx0) * sc, ay + (rt_h - ty0) * sc, (colour & 0xFFFFFF) | (a << 24));
```

Each afterimage lasts longer the longer the trail is, and fades more steeply from new to old:

```cpp
DWORD life_ms(int n) { return 300u + 60u * (DWORD)n; }
float fade_of(int k, int n, DWORD age, bool lone, float base_lone, float base) {   // k = 1: the newest of n
  if (lone) return base_lone;
  float g = 1.0f + (float)n / 16.0f, x = 1.0f - (float)(k - 1) / (float)(n + 1), a = 1.0f - (float)age / (float)life_ms(n);
  if (a < 0) a = 0;
  return base * powf(x, g) * a;
}
```

A new trail starts when the actor reappears after half a second or more, or when the aura's colour changes by more
than 64 in any channel. Otherwise a new summon, placed where the old one was, would inherit the old one's trail.

An advanced mode (`aura_trail_advanced`) records the CPU-skinned body's posed triangles instead of a picture. Each
afterimage is then redrawn in 3D from the current camera, untextured, additive, depth-tested with no depth write. It
costs a full body draw per afterimage. The timings (300 + 60n ms, the exponent 1 + n/16, 500 ms, 64) are design
choices, not measurements.

### 10. Disintegrate: the PS2's break-apart effect

On the PS2, some models (the Shadow Lord's change of form) break apart into pieces with smoke and haze, and the PC
client doesn't draw this. **voetre's portfix** recreates it: it captures the model's skinned pieces at the device and
draws them as separate fragments. gobbiefix's default (`disintegrate=1`) is portfix's code, used under its MIT licence.
A second version of gobbiefix's own (`disintegrate=2`, which flickers the way the PS2 cutscene does) is built from
portfix's reading of the PS2. Neither version's code is included here: see portfix for the original.

---

## Sound

### 11. Looping sounds that stop for good (`sound_retry`)

**Symptom.** Rain and wind go silent after about a minute, and stay silent until the zone or weather starts them again.
This happens in the unmodified game. With the registry's voice limit (key 0029) at 20 instead of 12, it takes 5–10
minutes.

**Cause.**
- `CYySoundElem::UpdateTimerAndPosition` (0x100364E0) restarts an element with handle 0.
- That stores the handle that `SetVolumeVector` (0x10036F50) returns.
- **0** means "this update's queue is full". The element is kept and asks again next update.
- **−1** means the voice limit refused it (`QueueSoundCmdIfChannelsBelowLimit`, 0x10038330). A negative handle sends
  the element to its destructor. The sound is gone for good.

**Fix.** Answer a refusal at that call the same way as the queue-full case, with 0. Do this only for elements the game
itself keeps when they aren't heard: the element's own "keep when unheard" virtual function
(`VT_KEEP_WHEN_UNHEARD`), the same test the game makes there. `RET_IN_RESTART` is the return address of that call.

```cpp
static undefined4 __thiscall sf_setvol(void *self_, undefined4 a1, undefined4 a2) {   // (abridged: counters removed)
  undefined4 r = prev_setvol(self_, a1, a2);
  if ((uintptr_t)__builtin_return_address(0) != RET_IN_RESTART || !snd_elem) return r;
  void *e = snd_elem;
  if ((int32_t)r < 0 && sf_cfg.sound_retry) {
    typedef uint32_t (__thiscall *KeepFn)(void *);
    if ((uint8_t)((KeepFn)VT_KEEP_WHEN_UNHEARD(e))(e)) return 0;   // kept; it asks again next update
  }
  return r;
}
```

*In your source:* in `UpdateTimerAndPosition`, treat a refused voice like a full queue for elements that persist.

---

## Frame timing

### 12. A precise 60 fps limiter (`precise_pacing`)

**Symptom.** The game runs at 58.8 fps, not 60.

**Cause.**
- The limiter in `Frame_Render` (0x10011A00) loops `Sleep(1)` while `60 / (scale × rate) < floor` (the limiter's minimum ticks per frame).
- The rate is `1000 / ms` from `timeGetTime`, in whole milliseconds (0x10019550).
- With a floor of 1 tick, the wait ends at 17 ms: 1000 / 17 = 58.8.

**Fix.** Time the limiter's own rate calls with `QueryPerformanceCounter`, and replace its `Sleep` with a wait to the
exact moment `v` reaches the floor: a high-resolution waitable timer, then a short spin.

```cpp
// the moment v = 60 / (scale * rate) reaches ticks: rate = 1000 / ms, so ms = ticks * 1000 * scale / 60
double want = ticks * 1000.0 * scale / 60.0 + TIMER_PAUSED_MS(timer) + 0.002;   // + the paused ms, + 2 us
LONGLONG end = g_pp_base + (LONGLONG)(want * (double)g_pp_qpf.QuadPart / 1000.0);
...
if (remain > 0) sf_sleep_until(end, g_pp_qpf.QuadPart);
```

### 13. Game time below and above 60 fps (`frame_step`)

**Cause.**
- `Frame_Render` makes the frame's step:
  `min(max(trunc(average of the last 4 frame times in 1/60 s ticks), floor), 20)`.
- 218 call sites read it through `Camera_GetFrameScale` (0x10014CF0) as `max(step, 1)`.
- The truncation under-counts time below 60 fps: 62–99% of real time in our dumps. With the floor removed above
  60 fps, the step is 0, answered as 1, so the game runs faster than real time.

**Fix.** Keep the same value without truncation, and carry the fraction into whole ticks across frames:

```cpp
double c = xi_step_carry + f;    // f: the untruncated step, floored and capped as the game's
int w = (int)c;                  // whole ticks this frame
xi_step_carry = c - w;           // the fraction carried to the next frame
xi_step_whole = (float)w;
xi_step_frac = (float)f;
```

Mode 1 answers every caller with the whole ticks. Over time they add up to the game's own measure of the frames'
time. Mode 2 also gives the exact fraction to the 14 call sites we proved only accumulate it (every path ends in
`x += f(step)`).

**Not known:** how much code moves things a fixed amount per frame without reading the step. One such function is at
0x1024DE20. That code still runs faster above 60 fps.

### 14. An even frame cap (`frame_cap_fps`)

This paces frames evenly at a chosen rate, at Present: sleep most of the wait on a high-resolution timer, then spin
the last ~1 ms. It re-anchors if a frame is more than one period late. With the game's own step, only 60, 30, 20 and
15 keep game time equal to real time, because the step is whole 1/60 s ticks.

```cpp
LONGLONG period = g_qpf.QuadPart / fps;
if (!g_next || now.QuadPart - g_next > period) g_next = now.QuadPart;   // late by over a frame: re-anchor
LONGLONG remain = g_next - now.QuadPart;
if (remain > 0) {
  double ms = remain * 1000.0 / (double)g_qpf.QuadPart;
  if (ms > 1.5 && g_timer) {                 // sleep most of it on a high-resolution timer, spin the last ~1 ms
    LARGE_INTEGER due; due.QuadPart = -(LONGLONG)((ms - 1.0) * 10000.0);
    if (SetWaitableTimer(g_timer, &due, 0, NULL, NULL, FALSE)) WaitForSingleObject(g_timer, 200);
  }
  LARGE_INTEGER t;
  do { YieldProcessor(); QueryPerformanceCounter(&t); } while (t.QuadPart < g_next);
}
g_next += period;
```

---

## CPU and draw cost

### 15. SSE on every CPU that has it (`cpu_tier`, `sse_skinning`)

**Cause.** Two places check the CPU's vendor string before they check for SSE:
- The math-tier selection has a conditional jump (`VENDOR_TEST_JUMP`) that skips the SSE test unless the vendor
  string matched.
- The CPU detection routine sets the skinning loops' SSE flag only for an "Intel" or "Transmeta" vendor string.

So a CPU with SSE whose vendor string doesn't match, for example the x86 emulation in Windows on ARM when it reports
another vendor, runs the x87 paths. For the skinning loops (0x1002F1D0 and a second one) that includes every AMD CPU.
Which vendor strings the math tier's test accepts isn't covered here.

**Fix** (`cpu_tier`): two NOPs over that `je`, written as one compare-exchange of the aligned dword, before
`CreateProcessorInstance` (0x1006CB50) chooses the tier:

```cpp
volatile LONG *dw = (volatile LONG *)VENDOR_TEST_JUMP;            // test al, al; je <past the SSE test>
const LONG expect = VENDOR_TEST_BYTES, want = VENDOR_TEST_NOPPED;  // the same dword with the je as two NOPs
...
LONG prev = InterlockedCompareExchange(dw, want, expect);
```

*In your source:* test the CPUID feature bit alone, not the vendor string.

### 16. Occlusion readbacks without the wait (`occlusion_late`)

**Cause.** The occlusion test reads its result back from the GPU at once, so the CPU waits for the GPU.

**Fix.**
- Copy each query's picture to a lockable target of its own, and read it at that query's *next* probe, when it's long
  finished.
- The answer is then one probe late: an actor coming into view is drawn from its next probe.
- `occlusion_hold` hides an object only after two "hidden" answers in a row, because the late answer flips visible to
  hidden on 3–4% of probes.
- `occlusion_stagger` re-tests each actor every 8 frames instead of 4.

These rules were chosen by us, not taken from the game.

### 17. The renderer: Direct3D 8 on Direct3D 12 and 11 (d3d8on11)

gobbiefix ships its own Direct3D 8 implementation (`d3d8.dll`), which runs the game's Direct3D 8 calls on Direct3D 12
or 11. One measured gain: building each draw's texture table only from the views the generated pixel shader actually
reads cut that step from 0.72–0.88 µs to 0.34–0.36 µs per draw. The renderer isn't included in this repository.

---

## Not covered in this document

gobbiefix has more settings than the fixes above. These ones aren't described here:

- `skin_merge`: draws a model's runs of triangle-list records as one `DrawIndexedPrimitive`.
- `gpu_skin`: skins characters in d3d8on11's vertex shader instead of on the CPU.
- `lazy_state`, `block_cache`, `vs_dedupe`, `state_cache`, `extended_state_cache`: skip device calls that change
  nothing, or are undone before the next draw.
- `merge_scenes`: one `BeginScene`/`EndScene` pair a frame.
- `batch_up_draws`: merges back-to-back `DrawPrimitiveUP` calls.
- `uncap`: removes the game's minimum wait per frame (needed above 60 fps).
- `smooth_pacing` (`pace_adaptive`) and `step_exact`: other ways of pacing frames and timing the game's step.
- `liminality`: frames between the game's ticks drawn with positions and animations blended.
- `texture_compression`: the threshold for compressing textures as they load.
- d3d8on11's own settings: `swap_buffers`, `mipmaps`, `anisotropy`, `renderer`.
- Measurement only: `motion_stats`, `task_draws`, `null_draws`, `ab_levers`.

## Credits

- **Dellingr:** gobbiefix (the d3d8on11 renderer, the xikit kit, speedfix), speedfix_hook, and dagnyfix, whose
  flarefix, aura fix and decalfix led to `flare_fix`, `aura_size` and `decal_fix`.
- **Geno:** co-credited by Dellingr's request.
- **voetre:** portfix (MIT licence). `disintegrate=1` is portfix's code. `sphere_shine` and `disintegrate=2` are built
  from portfix's measurements of the PS2.
- **Krauerlabs:** SpectralFix, the source of `aura_soft`'s two changes and of enlarging the aura capture.
- **Daleterrence:** the prototype that dagnyfix's decalfix grew from.

FINAL FANTASY, FINAL FANTASY XI, PlayOnline and SQUARE ENIX are trademarks or registered trademarks of SQUARE ENIX CO.,
LTD. This project isn't made, approved or endorsed by SQUARE ENIX. No game files or game code are included.
