# Neural Rendering before the upscaler

Status: BUILT (2026-09-03), not compiled on Windows yet, not in-game tested. Written for whoever
picks this up on a machine with the toolchain, the driver and an RTX 5080. D3D12 and the two
D3D12 bridges only; native Vulkan stays on the after-upscale path.

## The idea

Today the model runs **after** the upscaler, over the finished display-resolution frame:

    game colour (render size) --> upscaler --> output (display size) --> NR edits output in place

The model's cost and memory scale with the display resolution. At 4K Performance the model is
working on four times the pixels the game rendered.

Stage 1 moves it **before** the upscaler, over the game's render-resolution colour:

    game colour (render size) --> NR edits a private copy --> upscaler reads the copy --> output

The model works on the smaller picture, so its cost and memory fall with the upscaling ratio
squared. What the picture then looks like -- the model was trained on finished frames, this shows
it a jittered, aliased one, and the upscaler enlarges whatever it made of it -- is the open
question, and the reason this is a setting rather than a change.

## What was built

| Piece | Where | What it does |
|---|---|---|
| `[DlssNr] Stage` | `Config.h`, `Config.cpp`, `OptiScaler.ini` | 0 after (default), 1 before |
| Stage combo | `dlssnr/DlssNr_Menu.cpp` | the same control in the menu, greyed on native Vulkan |
| `DlssNr::ScopedPreUpscale` | `dlssnr/DlssNrFeature_Dx12.h`, `shaders/dlssnr/DlssNr_Dx12.cpp` | the before-upscale path, as a scope around the upscaler's evaluate |
| `DlssNr_Dx12::Dispatch` | `shaders/dlssnr/DlssNr_Dx12.{h,cpp}` | now reads `colour` and writes `output`, which may differ; returns whether it wrote |
| `DlssNrFrameInfo::OutputState` | `shaders/dlssnr/DlssNr_Common.h` | lets a caller that owns the output say what state it rests in |
| `GatherFrame` | `shaders/dlssnr/DlssNr_Dx12.cpp` | the parameter-block reading both paths share, carved out of `EvaluateAfterUpscale` |
| Four call sites | `inputs/NVNGX_DLSS_Dx12.cpp` (native DLSS and OptiScaler's upscalers), `upscalers/IFeature_Dx11wDx12.cpp`, `upscalers/IFeature_VkwDx12.cpp` | the scope around each evaluate |

### The scope, step by step

`ScopedPreUpscale` is constructed around the upscaler's evaluate and does nothing unless the pass
is enabled and `Stage == 1`.

1. Reads `Color`, `Depth`, `MotionVectors` from the parameter block, remembering whether `Color`
   was stored typed or untyped (NVIDIA's block keeps the two slots apart; OptiScaler's does not).
2. Sizes the working copy to the **render subrect** (`DLSS.Render.Subrect.Dimensions`), bounded by
   the colour texture. A dynamic-resolution game renders into the corner of a larger allocation.
3. Declines to the after-upscale path if: the evaluate is ray reconstruction (undenoised colour),
   the colour subrect base is not the origin, or the colour is an sRGB format (no UAV, and its
   typed twin would be read back as linear).
4. Allocates or reallocates the copy (`g_pre.scratch`, UAV + RT allowed) at the typed twin of the
   colour's format. The old one is parked, not released, because the upscaler may still be reading
   it on the GPU.
5. Moves the game's colour from `ColorResourceBarrier` (if set) to `NON_PIXEL_SHADER_RESOURCE`,
   calls `Dispatch(cmdList, colour, depth, motion, scratch, frame)`, moves it back.
6. If the resolve actually wrote the copy: moves the copy into the state the game's colour rests in
   and writes it into the block as `Color`, the same way (typed or untyped) the original was stored.
   If not -- the model is still being built, or the frame was skipped -- the block is left alone
   and the upscaler reads the game's own colour, exactly as without the pass.
7. The destructor puts the original `Color` back and rests the copy in `UNORDERED_ACCESS`.

`EvaluateAfterUpscale` stands down on stage 1 unless the scope set `g_preUpscaleDeclined` for that
evaluate, so the model never runs twice a frame.

### What `Dispatch` needed

Almost nothing. The pass already took `colour` and `output` separately and then ignored `colour`.
Now the encode and the meter read `colour` (`source`), the resolve writes `output` (`target`), and
the UAV <-> SRV barriers on the target around those reads are made only when the two are the same
resource. When they differ, the source is already in `NON_PIXEL_SHADER_RESOURCE` -- NGX's contract
for every input -- and is never transitioned. The encode reads the source's top-left
`width x height`, which is the subrect; the copy is exactly that size, so every size the pass
derives from `output->GetDesc()` is right without a new field.

Everything else -- the working scale, the exposure meter, the white point sources, the capture, the
compare views, the timing -- works unchanged on either side. They are all functions of `source`,
`target` and the parameter block.

## What is expected to go wrong first, in order

1. **Jitter.** The render-size colour is sub-pixel jittered every frame and the model's temporal
   reprojection has no idea. Motion vectors do not carry jitter. Expect shimmer or a soft crawl on
   fine detail, especially at rest. Things to try, in order:
   - Scan `nvngx_dlssnr.dll` for a `DLSSNR.Jitter*` parameter name (the model has 61 `DLSSNR.*`
     names; the current code writes about 20). If one exists, write `Jitter.Offset.X/Y` from the
     game's block into it before evaluate. That is a two-line change in the forwarder's evaluate.
   - Failing that, offset the motion vectors by `(jitterNow - jitterPrev)` in a small compute pass
     over the motion clone (`ReadableGuide` already clones typeless guides; a typed clone path
     plus one more `DlssNrMode` would do it). Sign convention is engine-dependent; try both.
   - Failing that, the model's working scale below 100% (`WorkingScale`) softens its input and may
     hide the jitter at a further cost in detail.
2. **Dynamic resolution.** The copy is reallocated and the model rebuilt every time the render
   subrect changes. `README.md` warns that rebuilding the feature too often "exhausts the driver's
   latches and the feature stops responding until the process restarts". In a DRS game expect
   exactly that. The fix is the one NGX itself uses: create the feature at the colour texture's
   full size and evaluate at the subrect. Both the forwarder's create and evaluate already take a
   width and height; whether the model honours a smaller evaluate size is untested. If it does,
   size `g_pre.scratch` and the feature to `colourDesc` and pass the subrect only to evaluate.
   Until then, lock the render resolution in DRS games when testing.
3. **The white point.** The exposure texture and pre-exposure the game hands the upscaler describe
   the same colour buffer the model now reads, so they should be *more* right here than after the
   upscaler, not less. But the paper-white slider defaults were tuned on the output. Re-check.
4. **Motion vector scale.** The pass passes the game's `MV.Scale` through unchanged and lets the
   model relate the guide subrect to its output. Before the upscaler the two are the same size, so
   nothing should change. If the model warps, this is the second thing to look at after jitter.
5. **Resource states on the native DLSS route.** The scope assumes the game's colour rests in
   `NON_PIXEL_SHADER_RESOURCE` (or `ColorResourceBarrier` when set). A game that hands DLSS its
   colour in `GENERIC_READ` or `PIXEL|NON_PIXEL` will draw a debug-layer complaint; on NVIDIA
   hardware the read still works. If it visibly does not, set `ColorResourceBarrier` in the ini.
6. **Colour formats.** `R11G11B10_FLOAT`, `R16G16B16A16_FLOAT`, `R8G8B8A8_UNORM`,
   `R10G10B10A2_UNORM` should all allocate with UAV + RT. If `CreatePreUpscaleScratch` falls back
   to the plain scratch and `ColorResourceBarrier` says `RENDER_TARGET`, the transition into that
   state is invalid; the log line "the model works on the ... render-size colour" says which format
   was chosen.

## Test plan

1. Build. `Stage=auto` (0): the pass must be byte-identical to before. `DlssNrEnabled=false`:
   the scope is a no-op on every route.
2. Cyberpunk or any D3D12 DLSS game with a fixed render resolution, `Stage=1`, DLSS Performance.
   Confirm in the log: "DLSS-NR before the upscaler: the model works on WxH", "DLSS-NR running at
   WxH ... before the upscaler", and the after-upscale skip line. Confirm the cost row in the
   timing table drops by roughly the upscaling ratio squared.
3. Toggle `Stage` in the menu while running. The target changes size, the pass rebuilds, no crash.
4. `Compare` mode 2 (wipe) on stage 1 shows the *render-size* frame against the model's edit,
   before the upscaler; that is expected, and it is the honest comparison of what the model did.
5. Take matched captures (`AutoCapture`) on both stages of the same scene and look at fine
   detail at rest and in motion. That is the answer to the question this was built for.
6. A DRS game (or the DRS override in OptiScaler) to reproduce item 2 above and decide whether
   create-at-max is worth doing.
7. A DLSS-RR game: the pass must log the ray-reconstruction decline and run after the upscaler.
8. A D3D11 game and a Vulkan game through the bridges: same log lines, same picture.

## What this deliberately does not do

- No jitter compensation yet. See above; it is the first follow-up.
- No native Vulkan path. `DlssNrFeature_Vk.cpp` would need the same source/target split and a
  scope around the Vulkan evaluate; the D3D12 version is the template.
- No change to the composition. The proxy, the ratio transfer, the hue correction and the guard
  are the same code on both sides. If the model's answer on a jittered frame needs a different
  composition, that is a second experiment, not this one.
