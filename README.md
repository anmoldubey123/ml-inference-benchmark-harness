# ML Inference Benchmark Harness — NXP i.MX93

Reproducible latency benchmarking for TensorFlow Lite models on the NXP i.MX93, across both the Cortex-A55 CPU path and the Ethos-U65 NPU path.

The board runs a minimal Yocto image and is driven entirely over SSH from a Linux host. These scripts exist because the first round of benchmarks was run by hand, and when the numbers were questioned there was no way to tell whether a difference was real or an artifact of how the run was set up. Everything here is built around making a number defensible: fixed iteration counts, repeated runs, thermal state captured around every rep, and raw logs committed alongside the summaries.

---

## Environment

| | |
|---|---|
| Board | phyBOARD-Nash i.MX93 (2× Cortex-A55, Ethos-U65 NPU) |
| BSP | ampliPHY PD26.1.0, kernel 6.12.34 |
| Runtime | TensorFlow Lite 2.19.0 (`benchmark_model`) |
| CPU backend | XNNPACK, 2 threads |
| NPU backend | `libethosu_delegate.so`, Vela-compiled models |
| Model storage | NFS mount at `/mnt/models` |

Models live on an NFS share rather than the board's rootfs, which is only 1.5 GB and cannot hold the full model set. Inference performance is unaffected because TFLite mmaps the model file.

---

## Methodology

Every measurement follows the same procedure:

- **5 warmup iterations, then 50 timed iterations** per run. Warmup absorbs first-call allocation and cache effects.
- **3 repetitions per model**, with a 30-second cooldown between them.
- **Thermal state read before and after every rep** from `/sys/class/thermal/thermal_zone0/temp`, so thermal drift is visible rather than silently folded into the result.
- **Headline metric is the median of the three per-rep medians.** Medians rather than means because the tail is dominated by scheduler noise on a two-core board.
- **Standard deviation across reps is reported, and flagged above 1%.** In practice the published results hold under 1%; a run that does not is treated as suspect rather than published.
- The default 150-second wall clock limit in `benchmark_model` is disabled, since several models exceed it on CPU.

Raw logs for every rep are committed under `results/imx93/logs/` and `results/imx93/logs_npu/`. Summary tables cite the log filename for each row, so any published number can be traced back to the run that produced it.

---

## Layout

```
scripts/
  bench_model.sh        CPU benchmark over SSH, 3 reps with thermal capture
  bench_npu_model.sh    Same, offloading to the Ethos-U65 via the Ethos-U delegate
  inspect_model.py      Dump input/output shapes, dtypes and quantization params
models/aux/             Per-model metadata and label files
results/imx93/
  RESULTS.md            Per-rep tables and median-of-medians summaries
  logs/                 Raw CPU benchmark output
  logs_npu/             Raw NPU benchmark output
MODEL_SOURCING.md       Sourcing log: origin URL, SHA256, and validation status per model
```

---

## Usage

Set the board address, then run against a model basename:

```bash
export BOARD_HOST=root@<board-ip>

./scripts/bench_model.sh mobilenetv2_w8a8        # CPU
./scripts/bench_npu_model.sh mobilenetv2_w8a8    # NPU, expects <basename>_vela.tflite
```

Each script writes one log per rep and prints a summary block formatted for pasting directly into `RESULTS.md`.

To check a model's tensor specs before benchmarking it:

```bash
./scripts/inspect_model.py models/mobilenetv2_w8a8.tflite
```

This matters more than it sounds. A model that is nominally "int8" may still have float input or output tensors, which changes what the benchmark is actually measuring and whether the Ethos-U delegate will accept it at all.

---

## Running on the NPU

Models must be compiled with Vela before the Ethos-U delegate will run them. Two things determine whether a model works:

**Op support.** Vela reports how many operators it placed on the NPU. Anything it cannot place falls back to CPU.

**Island structure.** The delegate build used here accepts a model compiled into a single fused Ethos-U operator. A model that compiles into multiple islands — NPU segments interleaved with CPU fallback segments — is rejected at delegate initialization rather than running with partial offload. Non-maximum suppression is the usual cause, since it is not in the supported op set and splits the graph around itself.

Checking Vela's placement output before benchmarking is therefore worth the minute it takes; a model that reports CPU operators is going to fail at delegate init, not run slowly.

---

## Results

Full per-rep tables are in [`results/imx93/RESULTS.md`](results/imx93/RESULTS.md). CPU figures, 2 threads with XNNPACK:

| Model | Precision | Median | Throughput | Std across reps |
|---|---|---:|---:|---:|
| MobileNetV2 | w8a8 | 33.08 ms | 30.2 fps | 0.43% |
| ResNet50 | w8a8 | 165.18 ms | 6.1 fps | 0.15% |
| ResNet101 | w8a8 | 314.24 ms | 3.2 fps | 0.20% |
| DeepLabV3+ MobileNet | w8a8 | 1459.73 ms | 0.7 fps | 0.31% |

---

## Notes and limitations

- CPU frequency governor state is not exposed on this BSP; scaling is fixed via the kernel command line instead. Per-run clock is therefore assumed rather than measured.
- The Ethos-U compute clock is not exposed through the Linux clock framework. The 1 GHz figure used for cycle normalization is supported by the PLL hierarchy and by Vela's own reported accelerator clock, but it is inferred, not read from the hardware.
- Thermal readings come from a single zone sensor and are recorded for drift detection, not as a calibrated junction temperature.
- Results are specific to this board, BSP and TFLite version. Numbers from a different BSP are not directly comparable.
