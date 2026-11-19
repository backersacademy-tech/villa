# Real-CT evidence for native crop translation coverage

This is a supplementary evidence record for the proposed fix in [ScrollPrize/villa](https://github.com/ScrollPrize/villa) from `backersacademy-tech`, not a new model or an improvement in recovered text. The **code-review PR intentionally changes only the downloader and its regression test**; this separate branch keeps the supporting logs public without adding non-code files to the minimal PR.

## Problem and result — real Herculaneum CT, PHerc0139-w035

The official sparse-download planner selects CT chunks around the **untranslated** native crop. During ink-model training, `maybe_translate_crop_bbox` can translate the crop by up to **40 native voxels on Y and/or X**. Missing Zarr chunks then silently read as fill values. The proposed fix plans a ±40 Y/X safety margin, without widening Z.

Two authentic CT samples were read with the official native `vc_reader` and `InkDataset._native_sample`, comparing the sparse copy to the original source for *the same* crop and random translation. The 2026-11-14 Google Colab run was executed by the human owner; the **exact final patch** was rebuilt, installed and independently retested in an assistant-managed environment on **2026-11-18**. Both runs' raw voxel/tensor hashes matched.

| Sample | Crop shape (ZYX) | Native shift (Y,X) | Planner chunks without → with patch | Missing CT voxels before → after | `robust_mad` differing supervised tensor positions before → after |
| --- | --- | --- | --- | --- | --- |
| A | (80, 64, 64) | (-37, -14) | 272 → 448 | **1,890 → 0** | **23 → 0** |
| B | (80, 128, 128) | (+32, 0) | 410 → 577 | **5,602 → 0** | **715 → 0** (679 before at ink-labeled positions) |

After the patch, the sampled tensors are **bitwise identical** to the tensors sampled from the original CT in those two tests, with unchanged ink annotations. Without translation, the sparse copy also equals the original source. Source chunk copying was **bounded to these witnesses**; full official CLI invocation was `--dry-run`, not a full scroll download. The planner counts measure **planned chunks**, not actual compressed network-transfer overhead.

**Dataset:** Vesuvius Challenge, *CT Scans of Herculaneum Papyri*, PHerc0139, segment w035, scan `20250728140407-9.362um-1.2m-113keV-masked.zarr/0`, hosted in the public Vesuvius Challenge data bucket. Public CT data are under **CC BY-NC 4.0** subject to any asset-specific exceptions. [Data and citation](https://scrollprize.org/data) • [Open-data repository](https://github.com/ScrollPrize/open-data).

## Exact integrated test evidence

Before applying the patch, the original curation tests reported `35 passed`. With the **exact final patch**, the official curation test module ran inside an installed, non-editable Python 3.14 package with compiled C++ `vc_reader._reader` and `vc_reader._transforms` extensions:

```text
python -m pytest -q vesuvius/tests/ink_detection/test_curation_preprocessing.py
.......................................                                  [100%]
39 passed in 4.91s
```

A focused **negative control** replaces the final downloader with the old official downloader while retaining the new regression test. The expected failure confirms the regression test detects this specific bug rather than merely running:

```text
FAILED test_downloader_plan_wires_native_translation_halo
E       assert 4 == 9
1 failed, 38 deselected in 2.24s
```

`vc_reader` was compiled from source, not stubbed. The installed `vesuvius` wheel used the exact candidate download module SHA-256 `668b37c024efbd33733084969b911706a42106529fee21a5d75e47dd4949f5da` and test SHA-256 `d711f73e3b9f4a1f4442971c5222de3534659af3f62b6b05444757e7d41a83fb`. The unified patch SHA-256 is `62c3ea6732e5389b5dac803514bc9775239c488604a79cb73fa71314cb98ae44`. Tested against official commit `1386f072e80b094f1699c6f8d4454095fad66d8c` (download module unchanged in newer main when published). The **published public code** is the exact same diff.

## Scope and candid limitations

- Two level-0 CT witnesses from one segment, not an exhaustive evaluation of the public dataset, and no `validation_mask`.
- No ink-model training results, benchmark improvement, or increased decipherment measured.
- The conservative planner can select more chunks even when later translations do not occur. The measured increases (272→448; 410→577) are counts in two samples, **not downloaded bytes**.
- This record comes from real source CT with source-vs-sparse comparisons. It does **not** claim an independently repeated full model-training run or all-scales tests.
- The risk of native ±40 translations was already pointed out by the author of [PR #2379](https://github.com/ScrollPrize/villa/pull/2379); the new contribution is the functional downloader patch and integrated real-CT verification, not discovery of the possibility of translation.
- Bicubic interpolation was fixed upstream by [#2506](https://github.com/ScrollPrize/villa/pull/2506) and is not part of this change.

**Authorship / review:** The code, technical reporting and PR draft were developed with AI assistance. The human owner actually ran the Colab validation on 2026-11-14 and authorized publication; personally reviewing the source code is a distinct step and should not be asserted without it.
