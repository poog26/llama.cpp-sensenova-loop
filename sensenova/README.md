# SenseNova-Loop-9B: build and run

This branch includes the compatible runtime and converter for the **November 17, 2026 final release** of [SenseNova-Loop-9B](https://huggingface.co/sensenova/SenseNova-Loop-9B), as well as the Preview. Clone this fork directly; no patches need to be applied.

GGUF downloads: [sld26t/SenseNova-Loop-9B-GGUF](https://huggingface.co/sld26t/SenseNova-Loop-9B-GGUF).

Stock llama.cpp, LM Studio, and Ollama do not currently implement this architecture.

## Build the tested CPU tools

Install CMake, Ninja, and a C++ compiler. On Windows, use a Visual Studio developer terminal so MSVC is available. The following headless build avoids an unrelated prebuilt web-UI asset dependency:

```sh
git clone --branch sensenova-loop --single-branch https://github.com/poog26/llama.cpp-sensenova-loop.git
cd llama.cpp-sensenova-loop
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=OFF -DGGML_NATIVE=OFF -DLLAMA_CURL=OFF -DLLAMA_BUILD_TESTS=OFF -DLLAMA_BUILD_SERVER=OFF
cmake --build build --target llama-completion llama-mtmd-cli llama-quantize -j 2
```

Executables are in `build/bin`; add `.exe` on Windows. These CPU tools were built and tested on Windows. CUDA and Vulkan support are inherited from the base fork but were not validated for this final-release export; see [backend build documentation](../docs/build.md) before enabling them.

## Text

Download a text GGUF, for example `SenseNova-Loop-9B-Q4_K_S.gguf`. From a terminal where that file is available, run the built `llama-completion` with:

```sh
./build/bin/llama-completion -m SenseNova-Loop-9B-Q4_K_S.gguf -c 1024 -b 32 -ub 32 -t 2 -ngl 0 --no-repack --no-warmup --single-turn -p "What is 13 times 7?" -n 128 --temp 0
```

Adjust the model path if the GGUF lives elsewhere. Text-only inference does not require a projector. The supplied chat template supports thinking, which can consume output tokens before the final answer.

## Images

Also download `mmproj-SenseNova-Loop-9B-F16.gguf`. The same projector is used with any of the text quants:

```sh
./build/bin/llama-mtmd-cli -m SenseNova-Loop-9B-Q4_K_S.gguf --mmproj mmproj-SenseNova-Loop-9B-F16.gguf --image image.png -p "Describe the image." -c 1024 -b 32 -ub 32 -t 2 -ngl 0 --no-mmproj-offload --no-repack --no-warmup --image-min-tokens 64 --image-max-tokens 64 -n 128 --temp 0
```

The small image-token budget is a smoke-test setting, not a quality recommendation for detailed images. Increase the image budget and context together for real use, within available memory.

## Conversion and precision fixes

The converter accepts the original Hugging Face checkpoint with `--outtype bf16`; add `--mmproj --outtype f16` for the vision projector. Run `convert_hf_to_gguf.py --help` for input/output options and install its dependencies from the repository requirements.

- The loop runs 56 logical blocks using 32 stored physical decoder layers; repeat visits alias the shared tensors rather than duplicating weights.
- All 120 mHC routing and recurrent convolution tensors are explicitly kept F32, including the virtual tail blocks after the loop.
- Final-release YaRN metadata is preserved: factor 2, original context 262144, configured maximum 524288.
- The final checkpoint has no MTP head.
- The published reference model does not register or use the checkpoint's extra `model.language_model.loop_gate.logits` parameter. A meta-device comparison found that this was the only unexpected key, with no missing keys. Its exact F32 values are preserved as provenance metadata under `sensenova_loop.source_unused.loop_gate.logits`.
- Vision export registers the model's existing Qwen3.5 vision tower and records its original image-resizing limits.

## Verified behavior

Windows, CPU, Q4_K_S with the F16 projector:

- Arithmetic: 13 times 7 produced 91.
- A solid red image produced Red; changing only the pixels to blue produced Blue.
- Text export contains 523 tensors; the projector contains 334.
- GGUF metadata, tensor shapes, and protected F32 tensor types were checked.

These are smoke checks, not full numerical parity or a quality benchmark. GPU performance, long-context behavior, video, and broad multimodal accuracy have not been validated here.

## Attribution

Original model: SenseNova / OpenSenseNova, Apache-2.0.

Runtime and shared-layer/mHC implementation: [GuideboardLabs/llama.cpp-sensenova-loop](https://github.com/GuideboardLabs/llama.cpp-sensenova-loop), based on llama.cpp, at `fad174ecd8bf22746a626c58e67e193f095aaa46`.

Vision registration adapted from [annelo's vision converter work](https://huggingface.co/annelo/SenseNova-Loop-9B-Preview-Q4_K_M-Vision-GGUF/blob/main/runtime/03-sensenova-loop-vision.patch).

The code retains llama.cpp's MIT license. This fork adds the final-release precision/provenance fixes and integrated vision export.
