# Multipose Virtual Try-On / Pose-Transfer Evaluation Dataset

A benchmark for evaluating **virtual try-on (VTO)** and **pose-transfer** models on
real people. Each sample pairs a reference image of a person with a target pose and
the garment(s) to be rendered; models are scored on how faithfully they reproduce
the garment on the person in the target pose.

This is an **evaluation** set (not a training set): it is scored item-by-item, and
the same garment may appear in many items.

> **Subjects consented to public release.** Every person depicted signed a contract
> that explicitly permits public release of this data and was specifically informed
> of it. See `DATASHEET.md` for details and limitations.

---

## Tiers

The dataset ships in two co-keyed tiers (same `photoshoot` and garment ids in both):

- **`core/` (this directory)** — the lightweight, portable tier. Garments are
  downscaled to **2048 px long edge**; body images are the **768×1024 processed
  crops** of the subjects. This is what most users need.
- **`raw/` (sibling, optional download)** — full-resolution originals: body images as
  uncropped full-res JPEG (e.g. **4000×6000**) plus their full-res **body masks**
  (`raw/photoshoots/<pid>/body_masks/`), and garments as full-res RGBA PNG **with their
  original soft segmentation alpha**. Use this only if you need full resolution; the
  benchmarks and loader target `core/`.

---

## Directory layout

```
<dataset root>/
├── README.md  DATASHEET.md  LICENSE  CHANGELOG.md  manifest.json  SHA256SUMS
├── index/
│   ├── photoshoots.jsonl        # one row per photoshoot (id, n_poses, has_top/has_bottom)
│   ├── poses.jsonl              # one row per (photoshoot, pose): orientation
│   └── garment_catalog.json     # sha -> {path, view, class(es), partners, instances}
├── photoshoots/
│   └── <pid>/                   # pid = <bucket>_<user>_<set>, e.g. c3_1001_02
│       ├── meta.json            # garments (by role) + per-pose orientation
│       └── images/
│           ├── pose_000.png     # canonical body images (the person, one per pose)
│           └── pose_001.png ...
├── garments/
│   └── <sha256>.png             # content-addressed garment catalog (RGBA), 769 unique
├── benchmarks/
│   ├── multipose_tryon/{f2b,f2f,f2l,f2r}.jsonl
│   └── pose_transfer/{f2b,f2f,f2l,f2r}.jsonl
└── tools/
    ├── load.py                  # benchmark loader (see “Loading”)
    └── resolve_paths.py
```

---

## Data model

**Photoshoot** (`photoshoots/<pid>/`) is the atomic unit: one **person** (`user`)
wearing one **outfit** (`garment_set`), photographed across multiple poses. The id is
`<bucket>_<user>_<set>` where `bucket` records the capture batch
(`c1`,`c2`,`c3`,`d1`,`sel`) and keeps ids globally unique.

- **Body images** — `photoshoots/<pid>/images/pose_NNN.png`, one per pose, re-indexed
  `pose_000…`. `meta.json` lists each pose's `orientation` (front/back/left/right/unknown).
- **Garments** — content-addressed in `garments/<sha>.png`. A photoshoot references its
  garments by **role** in `meta.json`:
  ```json
  "garments": {
    "top":    {"class": "top",    "front": "garments/<sha>.png", "back": "garments/<sha>.png"},
    "bottom": {"class": "bottom", "front": "garments/<sha>.png", "back": "garments/<sha>.png"}
  }
  ```
  `bottom` is `null` for one-piece / top-only outfits. Each garment has a `front`
  (`gar_0`) and `back` (`gar_-1`) view; the two are **front/back partners**
  (see `garment_catalog.json["<sha>"]["partners"]`).

**Garment filenames are SHA-256 content ids of the full-res original**, used as stable
ids **across both tiers** (`core/garments/<sha>.png` ↔ `raw/garments/<sha>.png`). Note:
because the core copy is downscaled, it no longer re-hashes to its own filename — the
name is an *identifier*, not a checksum of the core file. The full-res `raw/` copies
are self-verifying.

**Garment alpha (mask).** `core` garment alpha is the segmentation mask, produced by
**thresholding the full-res alpha at 127 then area-downscaling** to 2048 — giving a
clean mask with a ~1px anti-aliased coverage edge along the garment silhouette. The
`raw` garments keep the **original (soft) full-res alpha**. Downstream code that wants a
hard mask should threshold (`alpha ≥ 128`).

---

## Benchmarks

Two families, each with four pose-orientation splits. **`f2{b,f,l,r}` = front-to-
{back, front, left, right}** (the orientation of the *target* pose relative to a frontal
source). 600 items per split.

- **`multipose_tryon`** — source and target are different poses of the **same**
  photoshoot; the task is to render the outfit on the person in the target pose.
- **`pose_transfer`** — source and target are **different** photoshoots; the task is to
  transfer a person into a new pose.

### Item schema (`benchmarks/<family>/<split>.jsonl`, one JSON object per line)
```json
{
  "benchmark": "multipose_tryon",
  "split": "f2b",
  "source": {"photoshoot": "c3_1001_02", "pose": "pose_005"},
  "target": {"photoshoot": "c3_1001_02", "pose": "pose_012"},
  "garments": {
    "top":    {"class": "top",    "front": "garments/<sha>.png", "back": "garments/<sha>.png"},
    "bottom": {"class": "bottom", "front": "garments/<sha>.png", "back": "garments/<sha>.png"}
  }
}
```
- **`source`** = the reference person image (the "user"); **`target`** = the pose to
  render. Bodies are addressed by `(photoshoot, pose)`; resolve to
  `photoshoots/<photoshoot>/images/<pose>.png`.
- **`garments`** are the outfit worn in the *target* photoshoot, referenced by path.
  `bottom` (and any `front`/`back`) may be `null`.

---

## Loading

`tools/load.py` resolves a split into ready-to-use items with absolute paths:

```python
from tools.load import load_split, read_benchmark_items

items = load_split("multipose_tryon", "f2b")     # root defaults to the dataset root
# or: read_benchmark_items("/abs/path/to/f2b.jsonl", root="/abs/dataset/root")

for it in items:
    it["user_img_path"]          # ".../photoshoots/<src>/images/<pose>.png"
    it["target_pose_path"]       # ".../photoshoots/<tgt>/images/<pose>.png"
    it["top_garment_path"]       # {"front": ".../garments/<sha>.png", "back": ... }  (values may be None)
    it["bottom_garment_paths"]   # {"front": ... | None, "back": ... | None}
```

`load_split` returns a list; `read_benchmark_items(jsonl_path, root)` reads any single
file. Pass `root=` if the dataset isn't co-located with `tools/`.

---

## Index files

- **`index/photoshoots.jsonl`** — `{photoshoot, n_poses, has_top, has_bottom}` per shoot.
- **`index/poses.jsonl`** — `{photoshoot, pose, orientation}` per body image.
- **`index/garment_catalog.json`** — keyed by garment sha:
  `{path, view: front|back, classes: [...], partners: [sha...], instances: N}`.
  `instances` = how many photoshoot roles reference this garment (reuse; max 13).

---
