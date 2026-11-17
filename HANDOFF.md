# Handoff: 2DGS primitive and TSDF meshing

This file lives only on the fork's `handoff` branch; it must never reach the
PR branches. Last updated 2026-11-17.

## Where things stand

Two PRs are open on `harry7557558/spirula-studio`, **waiting for review**:

| PR | branch (fork `mlisi1`) | what |
|---|---|---|
| #360 | `pr/2dgs` | `--primitive 2dgs`, `spirula train 2dgs`, docs `docs/notes/2dgs.md` |
| #361 | `pr/tsdf-mesh` | `spirula mesh --method tsdf`, docs `docs/notes/tsdf.md`; stacked on #360 |

- Both were opened ready for review (not drafts) with GitHub's default titles
  "Pr/2dgs" and "Pr/tsdf mesh". Better titles: "2DGS surfel primitive
  (`--primitive 2dgs`)" and "TSDF meshing (`spirula mesh --method tsdf`)".
  The descriptions were written from drafts kept outside the repo.
- Both branches build and pass every test on CUDA and Vulkan (Linux); not
  built on Windows or macOS.
- `native-2dgs` is the old working branch: same code, older tests and docs
  (the deleted plan files). Superseded by `pr/*`; delete it once the user agrees.
- Upstream `master` at `b8e15cb8`; the fork's `master` is synced to it.

Results behind the PRs are in the two notes: Mip-NeRF 360 garden / bonsai
(the disc matches 3dgs indoors, -0.6 dB on garden; the preset's geometry
losses a further -0.6 to -0.8 dB; 1.5-1.6x train time), the preset sweep, and
the TSDF vs opacity-field mesh scores.

## Next session

1. **Check the PRs for review comments** (GitHub web; `gh` is not installed).
   Nothing else is pending until the maintainer answers.
2. **Answering a review on #360**: commit on `pr/2dgs`, push. Then restack:
   `git switch pr/tsdf-mesh && git rebase pr/2dgs`; the user force-pushes
   (`git push --force-with-lease=pr/tsdf-mesh:origin/pr/tsdf-mesh origin pr/tsdf-mesh`):
   the session's permission check blocks force-pushes, so hand the command over.
3. **When #360 merges**: `git fetch upstream`, then
   `git rebase --onto upstream/master pr/2dgs pr/tsdf-mesh` (works for a squash
   merge too), user force-pushes, and #361 shows only its own commits.
4. **Upstream moves before merge**: rebase `pr/2dgs` onto `upstream/master`,
   restack as in 2, rebuild both trees, run every test below, user force-pushes both.
5. **Open, user's decision**: commits carry the user's academic email as
   author; they were asked whether to switch to the GitHub no-reply address
   (a rewrite of both branches) and have not answered.

## Open technical items (none blocks the PRs)

- **DTU Chamfer** against the 2DGS paper's table: not run (no dataset on the
  machine). If TSDF on 2dgs falls short there, render depth with the model's
  own primitive through the engine (`forward_3dgs` with `output_median`)
  instead of the 3DGUT moments.
- **The sweep's 0.4 dB**: the preset sweep's absolute PSNR sits 0.4 dB above
  later runs of the same preset on the same capture. Not the seed cloud,
  presets, resolution or the dead-disc fix. The note presents the sweep for its
  differences only.
- **Low-pass**: whether the paper's screen-space low-pass is still needed next
  to the engine's antialiasing and `kernel_cut`.
- **Opacity-field sampler** wastes 2 of 7 samples per 2dgs splat (both on the
  centre, the third axis being `e^-20`).
- Preset weights and the `SettingCosts.h` row were chosen on one capture.

## Build and test

```bash
bash build_develop.bash -DSS_BACKEND=vulkan -DSS_BUILD_BACKEND_TESTS=ON
bash build_develop.bash -DSS_BACKEND=cuda -DSS_BUILD_BACKEND_TESTS=ON
for t in projection_parity render_parity raster_bwd_parity optimgeo_parity fpbo_parity; do
  ./build_cuda/$t dump /tmp/$t.bin && ./build_vulkan/$t compare /tmp/$t.bin; done
for b in vulkan cuda; do for t in train_config_test reg_loss_underflow tsdf_sphere \
  mesh_format_roundtrip preset_roundtrip_test render_project_test train_forecast_test; do
  ./build_$b/$t | tail -1; done; done
./build_vulkan/surfel_grad_test
python3 tools/check_ui_text.py && python3 tools/check_comment_length.py
```

The build runs the comment and UI-text checks; a piped `| tail` hides their
exit status, so check it.

## Things that bit

- Editing `src/shaders/` needs `src/shaders/build.bash` with slangc 2026.2.1,
  the version that made the committed headers.
- Scripted `spirula train` needs `--disable-viewer true` (or `1`), or it waits
  for Ctrl-C after training; `--num-iterations N` sets the length
  (`--max-steps` does not stop training).
- Timings need an idle GPU; other GPU jobs inflated step times 5-8x.
- `/tmp` (and the session scratchpad) is wiped on reboot: keep models and
  meshes the user may want to see elsewhere (`build_vulkan/` survives).
- Since upstream `69002756` a run writes `<run>/<run>.ply` and `<run>.tar`,
  no `step-*.ckpt/` folder; `spirula mesh <run>` reads both layouts.
- `spirula mesh` is not deterministic multi-threaded (parallel Delaunay); to
  compare two builds' meshes byte for byte use `--num-threads 1`.
