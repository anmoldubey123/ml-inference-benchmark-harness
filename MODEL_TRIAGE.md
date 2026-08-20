# Model Triage — i.MX93 Competition Analysis

Tracking progress sourcing TFLite models for the i.MX93 CPU benchmarking: **i.MX93 CPU (2× Cortex-A55, 2 threads, XNNPACK)**.

Status legend:
- **Sourced** — TFLite file in hand, all parameters verified, ready to benchmark.
- **Needs conversion** — base model located but quantization/format conversion required.
- **Blocked** — no clear path to a TFLite file matching the target spec; documented why.
- **TBD** — not yet investigated.

## Triage table

| #    | Spreadsheet Name                          | Architecture     | Target quant         | Input shape       | Source URL | Format found | SHA256 | Status | Notes |
|------|-------------------------------------------|------------------|----------------------|-------------------|------------|--------------|--------|--------|-------|
| 11.1 | ResNet50 w8a8 224×224                     | ResNet50         | w8a8 (int8/int8)     | 1×224×224×3       |  https://huggingface.co/qualcomm/ResNet50          |TFLITE w8a8 (Qualcomm AI Hub, QAIRT 2.45)              |158ca1bd97b26e62c43ad0bd2f6a71e63d4342574c0e25ba7b996383d9cf1a76         |Sourced    |Smoke-tested on board, ~166ms with 2t+XNNPACK       |
| 11.2 | MobileNetV2 w8a8 224×224                  | MobileNetV2      | w8a8 (uint8/uint8)    | 1×224×224×3       |https://huggingface.co/qualcomm/MobileNet-v2            |TFLITE w8a8 (Qualcomm AI Hub, QAIRT 2.45)              | ce69c99c2b307d45b03c1bd5ccdd3ee8b66e9cf704c087c6db76d78340c90d71       |Sourced    |Smoke-tested on board, ~33ms with 2t+XNNPACK       |
| 11.3 | MobileNetV2 w8a16 224×224                 | MobileNetV2      | w8a16 (int8/int16)   | 1×224×224×3       |https://huggingface.co/qualcomm/MobileNet-v2            | ONNX/QNN_DLC w8a16 only; TFLite w8a16 not pre-exported              |  —      | Needs conversion    |TFLite w8a16 not in Qualcomm pre-exports. Options: (1) qai-hub-models Python export tool — requires Qualcomm AI Hub account (free, public), risk of non-universal ops; (2) self-convert from float MobileNetV2 via TFLite converter with int16 activation target_spec; (3) substitute w8a8 and note in spreadsheet. Decide after triage of remaining rows.       |
| 11.4 | MobileNetV2 w8a16_mixed_int16 224×224     | MobileNetV2      | w8a16 mixed int16    | 1×224×224×3       |https://huggingface.co/qualcomm/MobileNet-v2            |ONNX/QNN_DLC w8a16_mixed only; TFLite not pre-exported              |—        |Needs conversion    |Same situation as 11.3. Mixed-int16 is Qualcomm-specific quant recipe (selective int16 on sensitive layers). Available via qai-hub-models Python export with --quantize w8a16_mixed_int16. Defer decision until full triage complete.        |
| 11.5 | ViT-Base                                  | ViT-Base/16      | fp32 | 1×224×224×3   |https://huggingface.co/qualcomm/ViT            |TFLITE float (Qualcomm AI Hub, QAIRT 2.45)              |882af0616b1f6aa5a1f6b750300d75648644222fdafea0217fec1cef2a5702cc        | Sourced    | Smoke-tested on board, ~3.30s with 2t+XNNPACK |
| 11.6 | BEVDet MobileNetV2 w8a16_mixed_fp16       | BEVDet (MNv2 backbone) | fp32 (deviation — see notes) | 1×6×3×256×704 (5 inputs total, image is 1×18×256×704)    |https://huggingface.co/qualcomm/BEVDet            |TFLITE float (Qualcomm AI Hub, QAIRT 2.45)              | dd9c29d18c5f2436e7e6a89a089fbe44ad791edc0e2e0eaadeddd6522805807       | Sourced (deviation)   | TFLite w8a16_mixed_fp16 not available from Qualcomm; only ONNX has w8a8_mixed_fp16 (note: spreadsheet says w8a16, Qualcomm exports w8a8 — possible spreadsheet typo). Using TFLite float as fallback per ViT precedent. ~30.6s with 2t+XNNPACK; |
| 11.7 | DeepLabV3-Plus-MobileNet w8a8             | DeepLabV3+ (MNv2) | w8a8 (uint8/uint8)    | 1×520×520×3 (export pad from 513) |https://huggingface.co/qualcomm/DeepLabV3-Plus-MobileNet |TFLITE w8a8 (Qualcomm AI Hub, QAIRT 2.45)  |4296b534a2cf054cfae2bf79d1c5ad9e206b64ff3095f384841aee91a0e25eb8        | Sourced   | Smoke-tested via NFS, ~1.44s with 2t+XNNPACK; output is per-pixel class indices (argmax internal). Input shape 520 not 513 (Qualcomm pad). |
| 11.8 | DeepLabV3-Plus-MobileNet w8a16 | DeepLabV3+ (MNv2 backbone) | w8a16 (int8/int16) | 1×520×520×3 (assumed same as 11.7) | https://huggingface.co/qualcomm/DeepLabV3-Plus-MobileNet | ONNX/QNN_DLC w8a16 only; TFLite w8a16 not pre-exported | — | Needs conversion | Same situation as rows 11.3, 11.4. Options: (1) qai-hub-models Python export with --quantize w8a16, (2) self-convert from float, (3) substitute w8a8 (already have from 11.7). Decision deferred until full triage complete. |
| 11.9 | ResNet101 w8a8 224×224 | ResNet101 | w8a8 (uint8/uint8) | 1×224×224×3 | https://huggingface.co/qualcomm/ResNet101 | TFLITE w8a8 (Qualcomm AI Hub, QAIRT 2.45) | 5237460259347f850d5d89f2d4097b02900860c2fc74bc0b93a8b15a4de0661f | Sourced | Smoke-tested via NFS, ~313ms with 2t+XNNPACK |
| 11.10 | ViT-Tiny | ViT-Tiny/16 | fp32 (same as ViT-Base) | 1×224×224×3 (assumed standard) | — | Not found pre-exported in TFLite | — | Needs conversion | Qualcomm AI Hub does not host ViT-Tiny (only ViT-Base). No community TFLite found in HF search. Conversion options: (1) ai-edge-torch from WinKawaks/vit-tiny-patch16-224 PyTorch weights, (2) timm ViT-Tiny → ONNX → TFLite, (3) skip with documented "could not source." Decision deferred. |

## Per-model investigation log

### 11.1 ResNet50 w8a8
- Downloaded: <05/20/2026> from https://huggingface.co/qualcomm/ResNet50
- File: resnet50-tflite-w8a8.zip (21M zipped, 26.3M extracted)
- SHA256: 158ca1bd97b26e62c43ad0bd2f6a71e63d4342574c0e25ba7b996383d9cf1a76
- Auxiliary files preserved in models/aux/resnet50_w8a8/ (metadata.json, labels.txt — ImageNet 1000 classes)
- Inspect: input uint8 [1,224,224,3] scale=1/255 zp=0; output uint8 [1,1000] scale=0.164 zp=51
- No custom ops; XNNPACK delegate applied cleanly on Ubuntu and on board
- Smoke test (2 threads, XNNPACK, 3+7 iters): avg 165.7ms, std 0.5ms, init 205ms
- Model is fully sourced and validated.

### 11.2 MobileNetV2 w8a8
- Downloaded: <05/20/2026> from https://huggingface.co/qualcomm/MobileNet-v2
- File: mobilenet_v2-tflite-w8a8.zip (3.3M zipped, 4.0M extracted)
- SHA256: ce69c99c2b307d45b03c1bd5ccdd3ee8b66e9cf704c087c6db76d78340c90d71
- Aux files: models/aux/mobilenetv2_w8a8/ (metadata.json, labels.txt — same ImageNet 1000 labels)
- Inspect: input uint8 [1,224,224,3] scale=1/255 zp=0; output uint8 [1,1000] scale=0.171 zp=63
- No custom ops; XNNPACK applied with 36 delegate kernels (depthwise-separable architecture splits heavily)
- Smoke test (2t XNNPACK, 15+31 iters): avg 33.2ms, std 105μs, init 40ms
- Note: page listed model size as "w8a16: 4.39 MB" but file is uint8/uint8 = w8a8; treating as page typo
- Status: Sourced and validated.

### 11.3 MobileNetV2 w8a16
- Qualcomm HF page lists w8a16 only for ONNX and QNN_DLC, not TFLite.
- TFLite w8a16 obtainable via `qai-hub-models` Python export with --quantize w8a16
  (requires free Qualcomm AI Hub account; cloud-based compilation; risk of non-universal ops on i.MX93).
- Alternative: self-convert from float MobileNetV2 using TFLite converter
  (target_spec.supported_ops = TFLITE_BUILTINS_INT8 + activation int16, representative dataset).
- Decision deferred until full triage complete to allow batched conversion strategy.

### 11.4 MobileNetV2 w8a16_mixed_int16
- Qualcomm HF page lists w8a16_mixed for ONNX and QNN_DLC only, not TFLite.
- "Mixed int16" = selective int16 activations on sensitive layers (typically first/last, residual adds);
  rest stay int8. Qualcomm-specific recipe.
- Same sourcing options as row 11.3: qai-hub-models export tool, or self-convert.
- Self-conversion is harder than vanilla w8a16 — needs per-layer activation type assignment,
  which standard TFLite converter does not directly support. Likely requires qai-hub-models.
- Decision deferred until full triage complete.

### 11.5 ViT-Base
- Downloaded: <05/21/2026> from https://huggingface.co/qualcomm/ViT
- File: vit-tflite-float.zip (308M zipped, 331M extracted)
- SHA256: 882af0616b1f6aa5a1f6b750300d75648644222fdafea0217fec1cef2a5702cc
- Aux files: models/aux/vit_base_fp32/ (metadata.json, labels.txt — ImageNet 1000)
- Inspect: input fp32 [1,224,224,3]; output fp32 [1,1000]; no quant params (float)
- No custom ops; XNNPACK applied with 14 delegate kernels
- Smoke test (2t XNNPACK, 1+3 iters): avg 3,301ms (~0.30 fps), std 7.2ms (0.2%), init 806ms
- Memory: ~685MB peak working set (significant — ~33% of board RAM)
- confirmed quant target = fp32 (not w8a8); Qualcomm pre-exports preferred where available
- Status: Sourced and validated.

### 11.6 BEVDet
- Downloaded: <05/21/26> from https://huggingface.co/qualcomm/BEVDet
- File: bevdet-tflite-float.zip (158M zipped, 179M extracted)
- SHA256: 9dd9c29d18c5f2436e7e6a89a089fbe44ad791edc0e2e0eaadeddd6522805807
- Aux: models/aux/bevdet_fp32/metadata.json (no labels.txt — detection model, not classifier)
- Multi-input model: 5 inputs total
  - image [1, 18, 256, 704] fp32 (6 cameras × 3 RGB channels concatenated)
  - sensor2keyegos [1, 6, 4, 4], inv_intrins [1, 6, 3, 3], inv_post_rots [1, 6, 3, 3], post_trans [1, 6, 1, 3]
- 6 outputs: reg, height, dim, rot, vel, heatmap (all 1×*×128×128 fp32)
- No custom ops; XNNPACK applied with 10 delegate kernels
- Smoke test (2t XNNPACK, 1+3 iters): avg 30.57s (~0.033 scenes/s or ~0.20 cam-frames/s),
  std 20ms (0.07%), init 949ms
- Memory: 1420MB peak (~70% of 2GB) — fit but tight
- Quant deviation: target was w8a16_mixed_fp16 (TFLite); only float available from Qualcomm.

### 11.7 DeepLabV3-Plus-MobileNet w8a8
- Downloaded: <05/21/26> from https://huggingface.co/qualcomm/DeepLabV3-Plus-MobileNet
- File: deeplabv3_plus_mobilenet-tflite-w8a8.zip (5.1M zipped, 6.3M extracted)
- SHA256: 4296b534a2cf054cfae2bf79d1c5ad9e206b64ff3095f384841aee91a0e25eb8
- Aux: models/aux/deeplabv3plus_mobilenet_w8a8/ (metadata.json, labels.txt — 21 VOC classes)
- Inspect: input uint8 [1,520,520,3] scale=1/255 zp=0; output uint8 [1,520,520]
  (argmax internal — class indices, no class channel)
- IMPORTANT: input is 520×520, not the 513×513 the HF page advertises (likely 8-aligned pad)
- No custom ops; XNNPACK with 35 delegate kernels
- Smoke test (2t XNNPACK, NFS, 1+3 iters): avg 1,440ms (~0.69 fps), std 235μs (0.016%), init 135ms
- Memory: 65MB peak — comfortable
- Page typo: listed model size as "w8a16: 6.67 MB" but it's w8a8 (consistent with prior pages)
- Note: Cortex-A55 latency dominated by spatial activations (520² vs 224²) more than param count.
- Status: Sourced and validated.

### 11.8 DeepLabV3-Plus-MobileNet w8a16
- Qualcomm HF page lists w8a16 only for ONNX and QNN_DLC, not TFLite.
- Same situation as rows 11.3 and 11.4. Conversion options:
  (1) qai-hub-models export with --quantize w8a16 (free Qualcomm AI Hub account)
  (2) self-convert from float DeepLabV3+ MobileNet
  (3) substitute w8a8 (already have from 11.7) and note in spreadsheet
- Status: Needs conversion. Deferred until after Phase 1 formal benchmarks complete.

### 11.9 ResNet101 w8a8
- Downloaded: <05/21/26> from https://huggingface.co/qualcomm/ResNet101
- File: resnet101-tflite-w8a8.zip (37M zipped, 44M extracted)
- SHA256: 5237460259347f850d5d89f2d4097b02900860c2fc74bc0b93a8b15a4de0661f
- Aux: models/aux/resnet101_w8a8/ (metadata.json, labels.txt — ImageNet 1000)
- Inspect: input uint8 [1,224,224,3] scale=1/255 zp=0; output uint8 [1,1000] scale=0.147 zp=51
- No custom ops; XNNPACK applied with 1 delegate kernel (whole graph delegated)
- Smoke test (2t XNNPACK, NFS, 2+4 iters): avg 313ms (~3.2 fps), std 710μs (0.23%), init 803ms
- Memory: 116MB peak
- Init time high (803ms) — likely first-pass NFS fault-in; expected to drop on re-run
- Param count vs ResNet50: 1.74× more params, but 1.89× slower wall-clock (depth penalty)
- Status: Sourced and validated.

### 11.10 ViT-Tiny
- Qualcomm AI Hub: ViT-Base only, no ViT-Tiny variant.
- HF model search for "vit-tiny tflite" and tflite library tag: no community TFLite found.
- Reference PyTorch weights: WinKawaks/vit-tiny-patch16-224 (most-used HF mirror)
  or timm's vit_tiny_patch16_224.
- Conversion candidates:
  - ai-edge-torch (Google) — PyTorch → TFLite, current-recommended path for ViT-class models.
  - timm + onnx + onnx2tflite — older path, more steps, may have op-coverage issues.
- Status: Needs conversion. No pre-exported source available.


## Verification protocol (applied per sourced model)

Before promoting a row to **Sourced**, confirm via Python script (`scripts/verify_model.py`, TBD):

1. Input shape matches target.
2. Input/output dtype matches quantization claim.
3. Quantization scheme via `interpreter.get_input_details()` / `get_output_details()` `quantization_parameters`.
4. Single inference runs without operator errors.
5. Output shape and value range pass sanity check (not all zeros, no NaN).

## Transfer protocol

1. Download to `~/ml-inference-benchmark/models/` on Ubuntu.
2. `sha256sum` and record in table.
3. Run verification script on Ubuntu.
4. `scp` to i.MX93 (target dir TBD — likely `/home/root/models/` or similar).
5. `sha256sum` on board, confirm match.
6. Smoke-test with `benchmark_model` (1 thread, 1 iter) before full benchmarking run.
