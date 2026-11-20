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

### How to reach me

If you work at SQUARE ENIX and would like to get in touch, whether about using any of this or to receive the full
version with every address and offset, please use this repository's private reporting. Open the **Security** tab and
choose **Report a vulnerability** ([direct link](https://github.com/WGINC/Longthinx-Gobbiefix-Source-Files/security/advisories/new)). Only the repository's maintainers can read what you send.
Despite the form's name, you don't need to be reporting a vulnerability: put "Contact" in the title and write freely.

For the full version, please include a SQUARE ENIX company email address in your message. I'll write to it to
confirm it's you, then give your GitHub account read-only access to the private repository.

## How to read this

- **The build this is based on.** Everything here is based on the FFXiMain.dll build with PE timestamp `6A995428`
  (3 September 2026), unpacked: sha256 `1b9b59aedc18a5b7dcbd531fff598c3b43a359eeb5dc2798763bc0ff8a21a3ad`, 6,208,512 bytes. Addresses assume its preferred base `0x10000000`.
  The hash is of the unpacked DLL, so it won't match the packed file as it's installed. Unless a section says
  otherwise, every address, offset and constant was read from that build's code.
- **The latest build** is November's, PE timestamp `6AE818B4` (2 November 2026). gobbiefix runs on it through a
  translation from the analysed build: 4,746 of the 4,826 functions the kit lists are proven the same there, and every
  address speedfix uses is settled and lies outside the functions that changed. The installed file of that build
  (packed, as it's on disk) has sha256 `d809ebe333019395ad43ab7ec1d94d9f58fe1f62c2ae730334f730fa8e7d48fb`.
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
| 4 | Lamp glows | Glows flicker behind their lamp | The glow's draw call sites | `flare_fix` (on) | v3.1 confirmed by our tester; the V0.7 change (v4.4) not yet run in the game |
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
| 16 | Occlusion readbacks | The CPU waits for the GPU | The occlusion readback | `occlusion_late`, `occlusion_hold`, `occlusion_stagger`, `occlusion_nowait`, `occlusion_first` (speedfix.ini, on) | In use; pop-in, its known side effect, seen; with the buffer arena, frame 16.9 ms → 14.5 ms |
| 17 | The renderer | — | d3d8on11 | `renderer` (d3d12) | Measured: texture tables 0.72–0.88 µs → 0.34–0.36 µs per draw |
| 18 | State caches | The same state set again and again | Device calls | `state_cache`, `extended_state_cache` (speedfix.ini, on) | Measured: 51–61% of calls skipped |
| 19 | Held state changes | Changes undone before any draw reads them | Device calls | `lazy_state` (speedfix.ini, on) | Measured with 22: frame 39.02 ms → 30.86 ms |
| 20 | State-block cache | — | Device calls | `block_cache` (speedfix.ini, on) | No measurable gain |
| 21 | One shader per declaration | About 1,450 creations for 3 shaders | Device calls | `vs_dedupe` (speedfix.ini, on) | Seen in the logs: 3 shaders made a session |
| 22 | Merged `DrawPrimitiveUP` | Text and interface drawn one quad at a time | Device calls | `batch_up_draws` (speedfix.ini, on) | Measured: 86–90% fewer calls |
| 23 | GPU skinning | Characters skinned on the CPU | `Model_SkinVerticesCpu` | `gpu_skin` (speedfix.ini and d3d8on11.ini, on) | On in our tester's sessions since the fix; no problems seen; not measured |
| 24 | Texture compression at load | DXT encoding on the CPU at load | Texture loading | `texture_compression` (speedfix.ini, on) | Not measured |
| 25 | Above 60 fps | Held at 60 fps | `Frame_Render`'s limiter | `uncap` (speedfix.ini, on) | Measured: game time follows real time from 31 to 184 fps |
| 26 | Adaptive pacing | — | At Present | `smooth_pacing` (on) | Measured: aim within 0.2–0.6 ms of each frame |
| 27 | Merged character draws | One draw per record | `CYyModelDt::SkinAndDrawV2` | `skin_merge` (speedfix.ini, off) | No measurable gain |
| 28 | One scene per frame | Several scenes a frame | Device calls | `merge_scenes` (speedfix.ini, off) | No measurable gain |
| 29 | Step from the counter | — | `Frame_Render` | `step_exact` (speedfix.ini, off) | No effect measured |
| 30 | Liminality | — | The task pass | `liminality` (speedfix.ini, off) | Latest version not run in the game |
| 31 | Sending only what changed | A 3,200-byte record for every draw | d3d8on11 | `draw_delta` (d3d8on11.ini, on) | Run; not measured |
| 32 | Merging consecutive draws | One draw call per draw | d3d8on11 | `draw_merge` (d3d8on11.ini, on) | Run: about 7.8 draws a frame merged; frame time not measured |
| 33 | Mip levels, anisotropic filtering | Speckle and shimmer in the distance | Texture creation and world draws | `mipmaps` (on), `anisotropy` (16) | Measured: build times; speckle even in replays |
| 34 | Swap-chain buffers | — | d3d8on11 | `swap_buffers` (2) | Measured: 2 best; 4 never measured |
| 35 | Loading and build check | — | d3d8on11, xikit | `load_kit`, `update_notice` (d3d8on11.ini, on) | In use in every start |

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

**Cause and fix.** This one comes from Dellingr's dagnyfix (flarefix v3.1, then flare fix v4.4), which found the glow draws by their
signature:
- `DrawPrimitive` from a 36-byte-stride vertex buffer, at most 8 vertices;
- depth test on, depth write off;
- blending `SRCALPHA / ONE`.

V0.7 moves the glow the way dagnyfix's flare fix v4.4 does (the earlier v3.1 move, `WORLD' = T(0, 0, −0.4) × WORLD`,
went 0.4 along the glow's own z: wherever WORLD turns or scales that axis the glow left its place on the screen, which
players saw as lights in the wrong place in zones after the base game's):
- the anchor: the centroid of the draw's vertices (read through a read-only lock, cached by buffer, first vertex and
  count, emptied on Reset), put through WORLD; `d` its distance from the origin;
- nothing done when `d < 0.8` or not finite; else WORLD's rows 0–3, columns 0–2 times `s = (d − 0.4) / d`, for that
  draw only.

That scales the glow about the origin. The glows' VIEW is the identity (their WORLD already holds the camera), so the
origin is the eye, and a scale about the eye keeps the glow's place and size on the screen while it comes 0.4 nearer.
gobbiefix moves a glow only when VIEW is the identity. In a frame captured in the game (62 glows), the v3.1 move
shifted every glow on the screen (median 21 px at 1512×949); this one shifts none.

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

```cpp
// WORLD for the glow's draw (row-major, D3D's row vectors); c: the vertices' centroid
static inline bool sf_flare_scale(const float w[16], const float c[3], float out[16]) {
  float a[3];
  for (int k = 0; k < 3; k++) a[k] = c[0] * w[k] + c[1] * w[4 + k] + c[2] * w[8 + k] + w[12 + k];
  for (int k = 0; k < 3; k++) if (!isfinite(c[k]) || !isfinite(a[k])) return false;
  float d = sqrtf(a[0] * a[0] + a[1] * a[1] + a[2] * a[2]);
  if (!isfinite(d) || d < 0.8f) return false;
  float s = (d - 0.4f) / d;
  for (int r = 0; r < 4; r++) for (int k = 0; k < 4; k++) out[4 * r + k] = k < 3 ? w[4 * r + k] * s : w[4 * r + k];
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

- `occlusion_nowait` (on): the late read never waits for d3d8on11's thread. It takes the newest finished picture, or
  the object's last answer again. With it and the renderer's buffer arena (section 17), the median frame in a busy
  town scene was 14.5 ms standing and 12.5 ms running, against 16.9 ms and 19.0 ms without them.
- `occlusion_first=2` (on): an object's first probe has no picture of its own yet, so the game's own readback would
  wait for the GPU. It's answered "visible" without waiting instead. In one A/B the game's readback ran once in the
  phases without it, taking 208.6 ms, and never in the phases with it: it removes that freeze, but it doesn't change
  the frame rate.

These rules were chosen by us, not taken from the game.

### 17. The renderer: Direct3D 8 on Direct3D 12 and 11 (d3d8on11)

gobbiefix ships its own Direct3D 8 implementation, `d3d8.dll`, which runs the game's Direct3D 8 calls on Direct3D 12
(the default) or Direct3D 11. It's entirely gobbiefix's own code: it doesn't patch or read the game, and the game
sees an ordinary Direct3D 8 device.

**Two halves joined by a command stream.**
- The **D3D8 side**, which the game calls, keeps the state the game sets and reads back, the state blocks, and a CPU
  copy of every texture and buffer that the game writes through `Lock`. `Unlock` sends only what changed.
- At each draw it decides everything Direct3D 8's fixed-function pipeline would do (the shader keys, the constants,
  the blend, depth and rasterizer states) and writes one draw record with the ids of the buffers and textures it reads.
- The records go through a 32 MB ring to the **executor**, the only code that uses Direct3D 12 or 11. By default it
  runs on its own thread. The game waits only for the four records that need an answer (start, resize, readback,
  finish), for room in the ring, or when it's two Presents ahead.
- A third mode runs the executor as a separate process, built natively for ARM64 on Windows on ARM machines. It
  presents into the game's window.

**The fixed-function pipeline as generated shaders.** Each combination of texture-stage operations, lighting, fog and
vertex format becomes HLSL once, is compiled and cached. The texture stages translate directly:

```cpp
switch (o) {
case D3DTOP_SELECTARG1: return A;
case D3DTOP_SELECTARG2: return B;
case D3DTOP_MODULATE: return A + " * " + B;
case D3DTOP_MODULATE2X: return "saturate(" + A + " * " + B + " * 2)";
case D3DTOP_MODULATE4X: return "saturate(" + A + " * " + B + " * 4)";
case D3DTOP_ADD: return "saturate(" + A + " + " + B + ")";
case D3DTOP_ADDSIGNED: return "saturate(" + A + " + " + B + " - 0.5)";
case D3DTOP_BLENDDIFFUSEALPHA: return "lerp(" + B + ", " + A + ", i.d0.a)";
...
```

**The Direct3D 12 executor.** The same command stream, run on Direct3D 12:
- one 64 MB upload ring, mapped once, each region reused only after the GPU has passed the submission that read it;
- one pipeline state object per combination of shaders and states, made once and cached;
- a flip-model swap chain;
- each draw's texture table built only from the views its generated pixel shader reads.

Why Direct3D 12: on our tester's Mac, 2,000 small draws took 6.1 ms through Direct3D 11's immediate context and
2.0 ms through Direct3D 12. If a Direct3D 12 device can't be made, the start runs on Direct3D 11.

**Readbacks without the wait.** A small lockable render target that's only written by `CopyRects` (the game's 16×16
occlusion target) gets a snapshot after each copy. `LockRect` reads the snapshot when it's of the surface as it is
now, instead of waiting for the GPU. This is exact because the only Direct3D 8 methods that write such a surface are
`CopyRects`, being the bound render target, and an unlock after a writing lock. The game's own readback locks right
after its copy, so it still waits. `occlusion_late` (section 16) locks a frame later and is answered from the snapshot.

**Fewer calls into Direct3D 11.** Two features exist for the Direct3D 11 executor only:
- the executor's filter skips Direct3D 11 calls that would set what it last set;
- its batches map each constant and vertex arena once per batch instead of once per draw.

The buffer arena puts the game's dynamic buffers in two large buffers. Measured in our tester's game: the upload time
fell from 2.4–2.7 ms a frame to 0.05–0.07 ms, and the draws cost about 1 ms more, for a net gain of about 1.4 ms a
frame. All three are on in the shipped settings, but only the Direct3D 11 executor uses them, so with the default
Direct3D 12 renderer they do nothing.

**Also always on.** Compiled shaders and the driver's pipeline library are cached on disk next to the executor, so a
later start doesn't compile them again. Fatal exceptions (access violations, illegal instructions, stack overflows
and the like) are written to the log with the module, offset, thread and registers.

**GPU skinning** is described in section 23.

**Measured.** On Direct3D 12, building each draw's texture table only from the views the pixel shader reads cut that
step from 0.72–0.88 µs to 0.34–0.36 µs per draw. In crowd scenes of 1,600–2,000 draws a frame, the executor took
11–13 ms a frame on Direct3D 12 against 17.5–19.7 ms on Direct3D 11. That run's picture had a texture-sampling fault,
which a check at start-up now guards against. In a busy scene the executor spends about 7.7 µs per draw at
1,500–1,700 draws a frame. The number of draws is what limits it, which is why sections 18–23 and 27 matter.

**Not established:** the executor as a native ARM64 process crashed inside a third-party overlay's DLL in one of our
tester's sessions, and the cause isn't known.

---

## More performance and timing work

### 18. Skipping device calls that change nothing (`state_cache`, `extended_state_cache`)

**What the game does.** It sets the same render state, texture-stage state, texture and vertex shader again and again,
often to the value already set.

**Fix.** Keep a shadow of what the device holds and drop a call that sets the same value. `extended_state_cache` does
the same for transforms, the viewport, lights, stream sources and indices. Calls from any other thread than the
render thread are never cached, and they make the whole shadow unknown. `vertex_shader_cache` does the same for
`SetVertexShader`, and `state_cache_reset_each_scene` forgets the shadow at each `BeginScene` and `EndScene`, in case
an overlay calls the device below gobbiefix.

```cpp
bool use = sf_cfg.state_cache && !g_recording && (unsigned)st < 256;
if (use && g_rsV[st] && g_rs[st] == v) { g_skip++; return D3D_OK; }   // the device already holds it
if (!(g_upPrims && g_upRhw && tnl_only(st))) flush();               // a real change: pending batched draws go out first
return fwd_rs(st, v);
```

**Measured:** the cached setters skipped 51–61% of calls. In one A/B, `extended_state_cache` was 2.53 ms a frame
faster, which is within the run's spread of 4–5 ms, so it isn't established as a gain.

### 19. Holding state changes until a draw reads them (`lazy_state`)

**What the game does.** It often changes a state and changes it back before drawing. For example, its quad drawing
(`Cell_DrawQuadBlended`) sets `ALPHABLENDENABLE` to 0 after each quad and to 1 before the next.

**Fix.** Direct3D 8 reads that state when it draws, not when it's set. So `SetRenderState`, `SetTextureStageState`
and `SetTexture` are held until something reads the state (a draw, `Clear`, a `Get`, a state block, the scene's end,
`Present`). A change undone before then never reaches the device.

```cpp
if ((unsigned)st < 256 && lazy_on()) {
  int k = (int)st;
  if (g_rsV[st] && g_rs[st] == v) { if (g_isP[k]) unpend(k); else g_skip++; return D3D_OK; }   // undone, or already set
  if (g_isP[k]) { g_pv[k] = v; g_dropped++; } else pend(k, v);                                 // held
  return D3D_OK;
}
```

**Measured:** about a third of held changes are never sent (for example 6,098,785 of 21,719,067 in a minute). Together
with section 22, it cut the frame from 39.02 ms to 30.86 ms in a busy town scene (−20.9%).

*In your source:* set the blend state once for a run of quads, not around each one.

### 20. State blocks that keep the cache (`block_cache`)

When the game applies a state block it recorded, the cache learns what the block sets instead of forgetting
everything. The effects apply two or three blocks each. **Measured:** in one A/B it was 1.43 ms slower, within the
run's spread, so it gives no measurable gain.

### 21. One vertex shader per declaration (`vs_dedupe`)

**What the game does.** It calls `CreateVertexShader` about 1,450 times a session for fixed-function declarations,
which come down to 3 distinct shaders.

**Fix.** A fixed-function vertex shader is completely defined by its declaration and usage (DirectX 8 SDK,
`CreateVertexShader`). So a second creation of the same declaration returns the first one's handle, and the shader is
deleted when the last of its creations is.

```cpp
for (int i = 0; i < g_nvsd; i++)
  if (g_vsd[i].usage == usage && g_vsd[i].n == n && !memcmp(g_vsd[i].decl, decl, n * sizeof(DWORD))) {
    g_vsd[i].refs++; *ph = g_vsd[i].h; LeaveCriticalSection(&g_vsdLock); g_vsdShared++; return D3D_OK;
  }
```

*In your source:* create the three shaders once at start-up.

### 22. Merging back-to-back `DrawPrimitiveUP` calls (`batch_up_draws`)

**What the game does.** Text and interface quads are drawn one small `DrawPrimitiveUP` at a time, with no state change
between them.

**Fix.** Copy each call's vertices into one buffer and send them as one draw when the state changes or another kind of
call comes. Strips and fans become lists, but only with solid fill and Gouraud shading, where the conversion is exact.

```cpp
if (g_upPrims && (stride != g_upStride || g_upBytes + outBytes > maxB || (DWORD)g_upPrims + prims > maxP)) flush();
expand(t, prims, (const BYTE *)data, stride, g_up + g_upBytes);  // the caller's data is only valid during the call
g_upBytes += outBytes; g_upPrims += (int)prims; g_upStride = stride;
return D3D_OK;
```

**Measured:** 86–90% fewer `DrawPrimitiveUP` calls reach the device. See section 19 for the frame time.

### 23. GPU skinning (`gpu_skin`)

**What the game does.** `Model_SkinVerticesCpu` (0x1002F1D0) skins every character on the CPU, then locks two vertex
buffers and writes the skinned positions and normals into them.

**Fix.** Hand the job to d3d8on11, which skins in the vertex shader. The game's thread neither computes nor uploads
those vertices. The layout is the game's own: one-bone vertices of 6 floats, two-bone vertices of 14, a bone word per
vertex, and a 64-byte matrix per bone. The shader makes the same sums as the CPU routine:

```hlsl
cbuffer B : register(b1) { row_major float4x4 bones[256]; };
...
float4 p = float4(mul(v.ska, bones[v.ski.x]).xyz + mul(v.skb, bones[v.ski.y]).xyz, 1);
```

Where the device declines a model, the game skins it as before.

**In the game:** it has run once. In that run it hid every character but the player's, because d3d8on11 dropped the
game's occlusion rectangles while a skinned buffer was bound. That was fixed and tested offline. It has been on by
default since, and our tester has seen no problems with it. How much it saves hasn't been measured.

### 24. Fewer textures compressed at load (`texture_compression`)

The game compresses a texture to DXT at load when its class is 0, or above the registry setting 0018
(`g_cfgTextureCompression`). The encoding is D3DX's, on the CPU. gobbiefix sets the threshold to 15, so
only classes 0 and 10 are compressed and the rest load uncompressed. That means less CPU work at load and no DXT
artefacts on those textures, at the cost of more video memory. **Not measured** in the game.

### 25. Above 60 fps (`uncap`)

**What the game does.** `Frame_Render`'s limiter waits until the frame's time reaches a floor of whole 1/60 s ticks.
The floor is 1 at 60 fps, and the game's own setter writes 2 or 4.

**Fix.** gobbiefix's copy of `Frame_Render` reads that floor as 0, in the wait and in the step. The game's field
itself isn't written.

```cpp
static inline int xi_floor(void *self_) { return xi_uncap != 0 ? 0 : LIMITER_FLOOR(self_); }
```

Above 60 fps the game would then run faster than real time, so this needs `frame_step` (section 13). **Measured:**
with `frame_step=3`, game time follows real time from 31 to 184 fps. Code that moves things a fixed amount each frame
without reading the step still runs faster above 60 fps.

### 26. Adaptive pacing (`smooth_pacing`)

Instead of a fixed rate, each frame is held until the 90th percentile of the last 120 frames' own time has passed
since the one before. The spacing stays even at a rate that follows the scene.

```cpp
double tgt = tmp[(g_paceN - 1) * sf_cfg.pace_pct / 100]; if (tgt > 50.0) tgt = 50.0;   // the percentile, at most 50 ms
LONGLONG due = g_paceRel + (LONGLONG)(tgt * (double)g_qpf.QuadPart / 1000.0);
if (due > now.QuadPart) { wait_until(due); QueryPerformanceCounter(&now); }
```

**Measured:** in one session, frames of 6.7–15.6 ms (148–64 fps), with the pacing's aim within 0.2–0.6 ms of each
frame. On by default.

### 27. Merging a character's draws (`skin_merge`, off)

`CYyModelDt::SkinAndDrawV2` walks a model's command list and draws each triangle-list record with its own
`DrawIndexedPrimitive`. Consecutive list records draw consecutive ranges of the same index buffer. gobbiefix's copy
merges consecutive records with no state call between them into one draw. In a traced town frame that's at most about
247 fewer draws out of 1,620 (15%). **Measured:** 1.36 ms slower in one A/B, within the spread. In one session
vertices "exploded"; with it off they didn't appear, but that session also changed the CPU's math tier, so the cause
isn't settled. Off by default.

### 28. One scene per frame (`merge_scenes`, off)

The game ends and begins scenes several times a frame. This holds an `EndScene` and drops it with the next
`BeginScene`, so there's one pair per frame. **Measured:** 1.17 ms slower, within the spread. Off by default.

### 29. The game's step from the performance counter (`step_exact`, off)

With `frame_step`, the game moves things by this frame's own time instead of its average of the last 4 frames.
**Measured:** no effect on movement or the camera. Off by default.

### 30. Liminality: frames between the game's ticks (`liminality`, off)

With `uncap`, the game's own logic still runs on whole 1/60 s ticks. On a frame with no tick, only the three drawing
tasks run (the actors, the zone, the name plates), so the world is drawn again unchanged. Stage 2 draws characters'
positions, facing and the camera, and with `lim_limbs` their animations, at the blend of their last two ticks. A move
of 50 units or more in a tick (a warp) isn't blended.

The facts it relies on come from the game's code: what OnMove clears and what the drawing path sets, so that a frame
without OnMove draws what the last tick drew. **In the game:** its latest version hasn't been run. It passes its
offline tests. Off by default.

---

## The renderer's other improvements, and how gobbiefix starts

### 31. Sending only what changed (`draw_delta`)

**What it costs.** A draw record is 3,200 bytes, built from the device's whole state at every draw: 2.2–2.4 µs of the
game's thread each, in our tester's game.

**Fix.** A counter goes up at every device call that can change the first 3,024 bytes (28 methods that write state,
the state blocks, the targets, the vertex shader table). The stream sources and the index buffer are in the last 176
bytes. A buffer draw made with the counter as it was at the last full record sends only those 176 bytes, and the
executor puts them after the last record's head.

**In the game:** run, not yet measured. The tests check that every scene is the same with and without it.

### 32. Merging consecutive draws (`draw_merge`)

**Fix.** Consecutive indexed triangle-list draws whose records are equal apart from their vertex and index buffers
(the same shaders, constants, states, viewport, textures and samplers) are drawn as one. On Direct3D 12, each draw's
vertex buffers are copied by the GPU, once per version, into one of four 16 MB slot buffers, and its indices are
rewritten as 32-bit numbers. The same triangles are drawn in the same order with the same state, so the picture is the
same.

**In the game:** in one run's log, about 7.8 draws a frame were merged, in crowd frames where about 1,040 of 1,650
draws a frame were sent as delta draws (section 31). Why most of those runs aren't merged isn't established, and the
effect on frame time hasn't been measured.

### 33. Mip levels and anisotropic filtering (`mipmaps`, `anisotropy`)

**What the game does.** Every texture it draws has one level: 148 of 148 in our tester's frame captures, and no draw
uses a mip filter. Distant ground and text therefore skip texels, which shows as speckle and shimmer.

**Fix.** A managed one-level 2D texture is created with a full chain of levels, made from level 0 each time the game
writes it.
- The levels are a 2×2 box average. Block formats are decoded, averaged and encoded again.
- For a cutout texture (every alpha 0 or full), each level's alpha is scaled so the share at or above 128 stays level
  0's. 128 is our choice, because the draws' alpha reference isn't known when the levels are made.
- The game still sees one level. A texture rebuilt more than twice its size within 60 frames gets no levels from then
  on.
- Only draws of 3D vertices use the levels, with linear minification and mip filtering. The interface keeps the
  game's own sampling.
- `anisotropy=16` makes those draws anisotropic where the game magnifies linearly.

The builds run off the game's thread, on the executor or a worker thread, so until a build is ready the texture is
drawn with level 0 only, for a frame or a few after loading.

**Measured.** On our build machine, a build takes about 1.3–1.7 ms for a 256×256 DXT texture and about 45 ms for the
2048×4096 font atlas. A later change that stopped copying each level brought a 2048×4096 32-bit texture from 51 ms
to 10 ms. In our tester's game, about 475 builds ran in the first 600 frames (the loading), then 0–300 per
600 frames. In replays of our tester's
captures, the far ground's speckle becomes even. The distant names change little, because their glyphs are drawn near
their own size.

*In your source:* create the game's textures with a mip chain and set a mip filter on world draws.

### 34. Swap-chain buffers (`swap_buffers`)

With Direct3D 12's flip-model swap chain and 2 buffers, one buffer is on the screen until the compositor's next
refresh and the other is being drawn, so `Present` waits for that refresh. In one measurement at 120 Hz, `Present` was
3.6–4.5 ms of the executor's 7.2–7.8 ms a frame. 3 or 4 buffers let a finished frame wait in the queue instead. In our
tester's measurements 2 did best (fewer frames thrown away, less judder) and 3 made more frames but threw more away.
4 is allowed but has never been measured. The shipped value is 2.

### 35. How gobbiefix starts, and checks the game's build (`load_kit`, `update_notice`)

1. At the game's first `Direct3DCreate8`, `d3d8.dll` reads FFXiMain.dll's PE timestamp in memory and looks it up in
   the kit's manifests.
2. If the build is known, it loads `xikit.dll`, the hooking kit. The kit waits for the device, then loads speedfix.
3. Each manifest lists every function the kit may hook, with a fingerprint of its first bytes. The kit checks the
   fingerprint before it hooks, and refuses a function that doesn't match.
4. For a build after the analysed one, the manifest carries the address ranges that moved. The kit translates every
   address at the boundary.
5. If the build isn't known, the kit isn't loaded and the renderer runs on its own. With `update_notice=1`, a window
   says gobbiefix needs an update and where to get it.

So a game update never runs gobbiefix's hooks against code they weren't checked against.

---

## Not covered in this document

Only settings that aren't in use by default are left out: the measuring tools (`motion_stats`, `task_draws`,
`null_draws`, `ab_levers`, `stats_every`, `log_stats`, `draw_sites`, and xikit's measuring modes) and options that are
off (`swap_flip`, `discard_hints`, `skip_actor_occlusion`, `skip_all_occlusion_readbacks`).

## Credits

- **Dellingr:** gobbiefix (the d3d8on11 renderer, the xikit kit, speedfix), speedfix_hook, and dagnyfix, whose
  flarefix, aura fix and decalfix led to `flare_fix`, `aura_size` and `decal_fix`.
- **Geno:** help with gobbiefix's Linux side, coming soon. Geno had no part in the reverse engineering or the fixes
  described here.
- **voetre:** portfix (MIT licence). `disintegrate=1` is portfix's code. `sphere_shine` and `disintegrate=2` are built
  from portfix's measurements of the PS2.
- **Krauerlabs:** SpectralFix, the source of `aura_soft`'s two changes and of enlarging the aura capture.
- **Daleterrence:** the prototype that dagnyfix's decalfix grew from.

FINAL FANTASY, FINAL FANTASY XI, PlayOnline and SQUARE ENIX are trademarks or registered trademarks of SQUARE ENIX CO.,
LTD. This project isn't made, approved or endorsed by SQUARE ENIX. No game files or game code are included.
