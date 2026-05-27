# buildpack-libtorch

Tiny Heroku-API buildpack that downloads the LibTorch CPU build (~200 MB) and configures bundler to compile `torch-rb` against it.

## What it does

1. **compile** — fetches `libtorch-cxx11-abi-shared-with-deps-${LIBTORCH_VERSION}+cpu.zip` from `download.pytorch.org`, caches it across deploys, unpacks under `/app/.libtorch/libtorch`, and writes:
   - `.bundle/config` with `BUNDLE_BUILD__TORCH__RB: "--with-torch-dir=/app/.libtorch/libtorch"`
   - `.profile.d/libtorch.sh` that exports `LD_LIBRARY_PATH=$HOME/.libtorch/libtorch/lib`
2. **release** — no-op.

`heroku/ruby` runs next, sees the bundle config, and successfully compiles `torch-rb` against the staged headers + libs. At runtime, every dyno sources `.profile.d/libtorch.sh` so `torch.so` finds `libtorch_cpu.so`.

## Deploying this buildpack to bld.io

This is an *inline* buildpack — it lives in the app repo, not on its own. There are two ways to wire it up:

### Option A: publish it as a public repo (recommended)

```bash
# in a fresh dir somewhere outside the app
git init
git add bin/ README.md
git commit -m "Initial libtorch buildpack"
gh repo create aluminumio/heroku-buildpack-libtorch --public --source=. --push

# then add it to the dicom-imager app, BEFORE heroku/ruby
bld buildpacks:add -a dicom-imager -i 2 https://github.com/aluminumio/heroku-buildpack-libtorch
# new order:
#   1. heroku-community/apt
#   2. https://github.com/aluminumio/heroku-buildpack-libtorch
#   3. heroku/ruby
#   4. heroku/procfile
```

### Option B: use heroku-community/inline

```bash
bld buildpacks:add -a dicom-imager -i 2 https://github.com/heroku/heroku-buildpack-inline
```

…but then `bin/detect`, `bin/compile`, and `bin/release` need to live at the **app repo root** (the inline buildpack expects exactly that layout). This conflicts with the existing `bin/` (rails, rake, jobs, etc.). Don't go this route unless you're prepared to namespace.

## Tuning

Override the LibTorch version per-app with:

```bash
bld config:set LIBTORCH_VERSION=2.9.0 -a dicom-imager
```

Pin to whatever version of LibTorch matches the `torch-rb` gem's codegen. As of `torch-rb` 0.24, that's PyTorch 2.9.x.
