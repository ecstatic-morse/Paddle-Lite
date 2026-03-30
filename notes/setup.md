## Model sources — what works and what doesn't

### ✅ Correct model: LAC v2.1 (Paddle 1.x, from baidu/lac releases)

Download URL:
```
https://github.com/baidu/lac/releases/download/v2.1.0/models_general.zip
```

The zip contains three models. Use `lac_model` for segmentation + POS tagging:
- `lac_model/model/` — Paddle 1.x format (`__model__` + individual param files)
- `lac_model/conf/` — config files: `word.dic` (58224 entries), `tag.dic` (49 B/I POS tags), `q2b.dic`

`seg_model` is segmentation-only (4 BIES tags). `rank_model` is unrelated.

### ❌ Incorrect models

| Source | Problem |
|---|---|
| `baidu/lac` Android testlac `model.nb` | `meta_version=0` — fatal crash on v2.12 (`LITE_ON_TINY_PUBLISH` build) |
| PaddleHub `lac==2.4.0` (`.pdmodel`/`.pdiparams`) | Optimized `.nb` produces wrong POS tags for sentences longer than ~8 chars (runtime inference bug) |
| PaddleHub `lac==2.4.0` assets `word.dic` | 20940-entry dict, incompatible with the 1.8MB segmentation-only model |

## Build the optimizer

```sh
bash lite/tools/build_macos.sh build_optimize_tool
# Output: build.opt/lite/api/opt
```

## Optimize the LAC v2.1 model

```sh
curl -L -o /tmp/models_general.zip https://github.com/baidu/lac/releases/download/v2.1.0/models_general.zip
unzip /tmp/models_general.zip -d /tmp/lac_models

build.opt/lite/api/opt \
    --model_dir=/tmp/lac_models/models_general/lac_model/model \
    --valid_targets=arm \
    --optimize_out=lac_v21_opt \
    --optimize_out_type=naive_buffer
# Output: lac_v21_opt.nb  (meta_version=2, ~30MB)
```

Copy the conf files alongside the binary:
```sh
cp -r /tmp/lac_models/models_general/lac_model/conf ./lac_v21_conf
```

## Build the Paddle-Lite library (macOS arm64)

```sh
bash lite/tools/build_macos.sh --with_extra=ON --with_log=ON --with_exception=ON arm64
# Output: build.macos.armmacos.armv8/inference_lite_lib.armmacos.armv8/
```

## Build the lac_demo

```sh
BUILD_DIR="build.macos.armmacos.armv8"
LITE_LIB="$BUILD_DIR/inference_lite_lib.armmacos.armv8"

clang++ -std=c++11 -O2 \
  -I"$LITE_LIB/cxx/include" \
  lite/demo/cxx/lac_demo/lac_demo.cc \
  lite/demo/cxx/lac_demo/lac.cc \
  lite/demo/cxx/lac_demo/lac_util.cc \
  "$LITE_LIB/cxx/lib/libpaddle_api_light_bundled.a" \
  -framework CoreFoundation \
  -o lac_demo
```

## Run with POS tagging

```sh
./lac_demo lac_v21_opt.nb ./lac_v21_conf input.txt label.txt <N>
```

Where `input.txt` has one sentence per line and `label.txt` is the reference output (use an empty file if not evaluating accuracy). `<N>` is the number of sentences to process.

Example output for `今天天气真好，我们去公园散步吧。`:
```
今天TIME 天气n 真好a ，w 我们r 去v 公园n 散步v 吧xc 。w
```

### Tag scheme (49 tags, B/I suffix)

The tag.dic uses `<pos>-B` / `<pos>-I` for word-begin/inside, plus a few named entity tags. POS categories include: `a` (adjective), `n` (noun), `nr` (person name), `ns` (place name), `nt` (org name), `v` (verb), `r` (pronoun), `m` (numeral), `q` (classifier), `d` (adverb), `p` (preposition), `c` (conjunction), `u` (auxiliary), `w` (punctuation), `xc` (other), `TIME`, `LOC`, `ORG`, `PER` (named entities).
