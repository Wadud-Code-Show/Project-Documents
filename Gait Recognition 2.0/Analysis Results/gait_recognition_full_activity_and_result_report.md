# Gait Recognition Project — Scratch থেকে Current Achievement পর্যন্ত Full Activity, Results, Analysis & Model Details

## 0. Project Goal / Big Picture

এই project-এর মূল goal ছিল **CSI/vision-based gait recognition workflow** তৈরি করা, যেখানে মানুষের হাঁটার pattern থেকে identity recognize করা হবে। Project progression-এর সময় আমরা মূলত CASIA-B style gait recognition pipeline নিয়ে কাজ করেছি এবং শেষে skeleton expert + silhouette expert fusion করে strong result পেয়েছি।

Final major achievement:

```text
Skeleton-only mean Rank-1   ≈ 82.61%
Silhouette-only mean Rank-1 ≈ 82.97%
Fixed score fusion mean     ≈ 95.73%

Fusion condition-wise:
NM Rank-1 = 99.36%
BG Rank-1 = 95.91%
CL Rank-1 = 91.91%
```

এটার সবচেয়ে বড় value হলো: single-modality skeleton বা silhouette individually moderate performance দিলেও, দুই expert-এর score-level fusion huge improvement দিয়েছে, especially **BG** এবং **CL** condition-এ।

---

## 1. Dataset Understanding and Initial Assumption

### 1.1 Working dataset convention

User’s dataset convention অনুযায়ী:

```text
Odd-numbered samples  = left-to-right walking
Even-numbered samples = right-to-left walking
Samples 81–100        = with-bag walking variant
```

With-bag samples 81–100-এর ক্ষেত্রেও odd/even direction rule apply করে, unless otherwise stated.

Working subset plan:

```text
24 participants × 100 samples/person
YashFinal excluded for now
```

তবে পরে CASIA-B based skeleton/silhouette pipeline-এ কাজ proceed করেছে। CASIA-B style condition split:

```text
NM = normal walking
BG = walking with bag
CL = walking with clothing variation
```

CASIA-B view angles:

```text
000, 018, 036, 054, 072, 090, 108, 126, 144, 162, 180
```

---

## 2. Overall Workflow We Built

Full workflow step-by-step:

```text
01 — Extract CASIA-B skeletons using YOLO pose
02 — Validate skeleton outputs
03 — Prepare CASIA-B train/gallery/probe splits
04 — Test dataset and feature generation
05 — Train skeleton GaitTR expert
06 — Evaluate skeleton GaitTR expert
07 — Analyze skeleton results
08 — Extract/convert silhouettes
09 — Validate silhouette-skeleton alignment and create fusion splits
10 — Train silhouette GaitGL-style expert
11 — Evaluate silhouette expert
12 — Compare skeleton vs silhouette experts
13A — Export skeleton embeddings for fusion
13 — Train/evaluate fusion: score fusion + trainable adaptive fusion
```

The final strong result came from **Step 13 score-level fusion**.

---

# PART A — Skeleton Expert Pipeline

---

## 3. Script 01 — Skeleton Extraction

### 3.1 Purpose

Script 01-এর কাজ ছিল CASIA-B videos থেকে skeleton/keypoints extract করা।

Main output structure:

```text
data/poses/.../*.npz
```

Each `.npz` pose file stores per-frame skeleton/keypoint data.

### 3.2 Model used

Skeleton extraction-এর জন্য YOLO pose model use করা হয়েছিল:

```text
yolo26l-pose.pt / YOLO pose model
```

Later project files-এর মধ্যে দেখা যায়:

```text
yolo26l-pose.pt
yolo11m-seg.pt
```

For skeleton extraction, pose model was used.

### 3.3 Output count validation

Script 01 / extraction-related validation-এ output দেখা যায়:

```text
Total pose files: 13640
Expected:         13640
```

Analysis:

```text
✅ Skeleton extraction complete
✅ Expected sample count matched exactly
✅ 124 subjects × 10 sequences/conditions style sample coverage reached
```

---

## 4. Script 02 — Validate Skeleton Outputs

### 4.1 Purpose

Script 02-এর কাজ ছিল extracted skeleton `.npz` files validate করা।

It checked things like:

```text
- pose file readability
- missing/empty skeletons
- valid frame ratio
- keypoint shape
- whether expected file count exists
```

### 4.2 “Skipped 5” clarification

এক পর্যায়ে output-এ `skipped 5` দেখা যায়। এর meaning ছিল:

```text
Skipped 5 মানে ৫টা sample necessarily bad না।
Script কোনো condition অনুযায়ী ৫টা item process করেনি / already handled / filtered / not relevant ছিল।
```

অর্থাৎ এটা direct “৫টা sample missing” এমন না, unless validation summary specifically missing/error দেখায়।

### 4.3 Final skeleton file count

Final count:

```text
Total pose files: 13640
Expected:         13640
```

Analysis:

```text
✅ Skeleton extraction stage structurally successful
✅ Pose file count complete
```

---

## 5. Script 03 — Prepare CASIA-B Splits

### 5.1 Purpose

Script 03-এর কাজ ছিল CASIA-B split তৈরি করা।

It created files under:

```text
data/splits/
```

or later fusion stage:

```text
data/fusion_splits/
```

Important splits:

```text
train_LT.csv
test_LT.csv
gallery_LT.csv
probe_LT_nm.csv
probe_LT_bg.csv
probe_LT_cl.csv
```

Later multimodal fusion version:

```text
train_LT_fusion.csv
gallery_LT_fusion.csv
probe_LT_nm_fusion.csv
probe_LT_bg_fusion.csv
probe_LT_cl_fusion.csv
```

### 5.2 CASIA-B LT protocol idea

The LT split followed standard CASIA-B train/test style:

```text
Training identities: first 74 subjects
Testing identities : remaining 50 subjects
```

Gallery/probe style:

```text
Gallery: normal walking sequences from test subjects
Probe NM: normal walking probe
Probe BG: bag condition probe
Probe CL: clothing condition probe
```

### 5.3 Analysis

```text
✅ Split files successfully created
✅ Later evaluation and fusion relied on these split files
```

---

## 6. Script 04 — Dataset and Feature Test

### 6.1 Purpose

Script 04 tested whether dataset loading and feature generation were correct before training.

The skeleton model did not use raw `(T, 17, 2)` only. It used engineered gait features.

### 6.2 Skeleton feature construction

Input skeleton:

```text
X shape = T × 17 × 2
```

Feature components:

```text
1. joint      = original normalized joint coordinates
2. joint_rel  = joints relative to nose/root
3. v1         = first temporal velocity
4. v2         = second temporal velocity / longer temporal difference
5. bone       = joint - parent_joint using COCO skeleton parents
```

Each component has 2 coordinate channels, so final channel count:

```text
5 components × 2 = 10 channels
```

Final GaitTR feature shape:

```text
C × T × V = 10 × 60 × 17
```

Where:

```text
C = feature channel
T = sequence length = 60
V = 17 joints
```

### 6.3 Analysis

```text
✅ Dataset loading worked
✅ Skeleton feature shape matched GaitTR input expectation
✅ Ready for training
```

---

## 7. Script 05 — Skeleton Expert Training: GaitTR

### 7.1 Initial tiny overfit test

Before full training, tiny overfit test was run.

Observed tiny overfit output:

```text
First loss: 1.1610...
Last loss : 0.2624...
Min loss  : 0.2624...
```

Graph showed loss decreasing sharply.

Analysis:

```text
✅ Model can learn from data
✅ Loss decreases properly
✅ Data-loader, feature generation, optimizer, and model forward/backward are working
✅ Safe to proceed to full training
```

---

## 8. Skeleton GaitTR Architecture Details

The exact architecture later confirmed from uploaded train/evaluation notebooks and used in `13A_export_skeleton_gaittr_embeddings_FULL_FIXED.ipynb`.

### 8.1 Input

```text
Input shape: B × C × T × V
C = 10
T = 60
V = 17
```

### 8.2 Feature channels

```text
joint      : 2 channels
joint_rel  : 2 channels
velocity 1 : 2 channels
velocity 2 : 2 channels
bone       : 2 channels
Total      : 10 channels
```

### 8.3 Core blocks

The GaitTR backbone uses:

```text
TCN + Spatial Transformer blocks
```

#### TCN block

Temporal convolution:

```text
Conv2d over temporal dimension
kernel_size = (temporal_kernel, 1)
temporal_kernel = 9
padding = temporal_kernel // 2
```

Followed by:

```text
Dropout
Mish activation
BatchNorm2d
```

#### Spatial Transformer block

For each time step, joint tokens are processed by multi-head self-attention over 17 joints.

Input transformation:

```text
B × C × T × V → (B*T) × V × C
```

Then:

```text
LayerNorm
MultiheadAttention
Linear projection
Mish
BatchNorm2d
```

#### TCNSTBlock

Each block:

```text
TCN → Spatial Transformer → residual connection
```

If channel changes, residual uses 1×1 Conv2d.

### 8.4 Channel configuration

From checkpoint/config:

```text
channels = (64, 64, 128, 256)
num_heads = 8
temporal_kernel = 9
dropout = 0.1
embedding_dim = 128
```

### 8.5 Global pooling and embedding

After TCN+ST blocks:

```text
x.mean(dim=(2, 3))
```

Then linear projection:

```text
fc: Linear(256 → 128), bias=False
```

Final embedding:

```text
L2-normalized 128-dimensional vector
```

### 8.6 Training loss

Later fixed skeleton training used:

```text
CrossEntropy loss + Triplet loss
```

The CE+Triplet version became the strong skeleton expert.

---

## 9. Skeleton Training — Fixed Recommended Version

### 9.1 Earlier fixed recommended training

A fixed training notebook was created after tiny overfit.

During run, progress bar showed:

```text
850/30000 steps ≈ 3%
loss ≈ 0.7138
```

GPU usage during training:

```text
GPU: RTX 3060
Memory used ≈ 1410 MiB / 6144 MiB
GPU utilization ≈ 97%
Temperature ≈ 87°C
Power ≈ 91W / 115W
```

Analysis:

```text
✅ GPU was highly utilized
✅ VRAM was not fully used, but compute utilization was high
✅ Training was GPU-bound enough
```

### 9.2 Full training output

One full fixed training run produced:

```text
Total steps: 30000
First loss: 1.2996...
Last loss : 0.3000...
Min loss  : 0.3000...
Batch size: 32
P = 8
K = 4
```

Loss curve:

```text
- Started around 1.3
- Dropped quickly under 0.4 around early steps
- Flattened near 0.30
```

Checkpoints produced:

```text
gaittr_LT_fixed_best_loss.pth
gaittr_LT_fixed_last.pth
gaittr_LT_fixed_step_1000.pth
...
gaittr_LT_fixed_step_29000.pth
```

Logs:

```text
gaittr_LT_fixed_train_log.csv
```

### 9.3 Analysis

```text
✅ Training converged
✅ Loss became stable
⚠️ But pure triplet/fixed version was not the final best skeleton strategy
```

---

## 10. Skeleton CE+Triplet Training

### 10.1 Motivation

The pure triplet/fixed run converged, but for identity recognition, CE helps class separation, while triplet improves metric embedding.

So we moved to:

```text
CrossEntropy + Triplet loss
```

### 10.2 CE+Triplet run progress

At 26% progress:

```text
7849/30000 steps
loss ≈ 1.0536
ce ≈ 1.020
triplet ≈ 0.168
acc ≈ 0.95
```

### 10.3 CE+Triplet final training output

Final training:

```text
30000/30000 steps completed
Final progress line:
loss ≈ 0.7896
ce ≈ 0.790
triplet ≈ 0.000
acc ≈ 1.00
```

Training log summary:

```text
First total loss: 4.3186...
Last total loss : 0.7896...
Min total loss  : 0.7759...
First CE loss   : 4.3186...
Last CE loss    : 0.7895...
First batch acc : 0.046875
Last batch acc  : 1.0
First pairwise  : 1.3765...
Last pairwise   : 1.4137...
```

Graph analysis:

```text
- Total loss and CE loss decreased steadily.
- Triplet loss decreased toward near-zero.
- Batch CE accuracy increased to near 1.0.
- Pairwise embedding distance stayed around 1.4, suggesting embeddings remained spread.
```

### 10.4 Checkpoints

Important CE+Triplet checkpoints:

```text
gaittr_LT_ce_triplet_best_loss.pth
gaittr_LT_ce_triplet_best_train_acc.pth
gaittr_LT_ce_triplet_full_last.pth
```

### 10.5 Analysis

```text
✅ CE+Triplet training successful
✅ Model learned identity classification on training set
✅ Embeddings became useful for retrieval
✅ This became our final skeleton expert family
```

---

## 11. Script 06 — Skeleton Evaluation

### 11.1 Purpose

Evaluate skeleton GaitTR checkpoints on gallery/probe retrieval.

Evaluation protocol:

```text
Gallery: gallery_LT
Probe: probe_LT_nm, probe_LT_bg, probe_LT_cl
Metric: Rank-1, Rank-5, Rank-10 / CMC
Exclude identical view: True
```

### 11.2 Checkpoints evaluated

We evaluated:

```text
gaittr_LT_ce_triplet_best_loss.pth
gaittr_LT_ce_triplet_best_train_acc.pth
gaittr_LT_ce_triplet_full_last.pth
```

### 11.3 Best skeleton result used later

From later skeleton embedding export sanity check, final skeleton expert achieved roughly:

```text
NM Rank-1 ≈ 95.00%
BG Rank-1 ≈ 78.64%
CL Rank-1 ≈ 74.18%
Mean Rank-1 ≈ 82.61%
```

Rank-5/Rank-10 from skeleton export sanity:

```text
NM: Rank-1=95.00%, Rank-5=97.18%, Rank-10=98.36%
BG: Rank-1=78.64%, Rank-5=88.82%, Rank-10=92.00%
CL: Rank-1=74.18%, Rank-5=86.27%, Rank-10=91.36%
```

### 11.4 Analysis

Skeleton expert behavior:

```text
NM: very strong
BG: moderate weakness
CL: better than silhouette-only later, but still challenging
```

Important insight:

```text
Skeleton is relatively more robust than silhouette under clothing variation, because body joints are less affected by clothing texture/outline changes.
```

---

# PART B — Silhouette Expert Pipeline

---

## 12. Silhouette Problem Discovery

### 12.1 Initial idea

We wanted silhouette expert training. Problem:

```text
If silhouette is extracted separately, frame-level mapping with already extracted skeletons may mismatch.
```

Initial concern:

```text
Skeleton frames and silhouette frames need to be aligned.
```

### 12.2 Two possible solutions considered

```text
1. Extract skeleton and silhouette together from same video frames
2. Use official CASIA-B silhouettes if already available
```

User found CASIA-B downloaded dataset already contained silhouette `.tar.gz` files.

Directory style after extraction:

```text
silhouettes/
├── 001.tar.gz
├── 002.tar.gz
...
├── 124.tar.gz
```

After extracting subject 001:

```text
001/
├── bg-01
├── bg-02
├── cl-01
├── cl-02
├── nm-01
├── nm-02
├── nm-03
├── nm-04
├── nm-05
├── nm-06
```

Inside a condition/view folder:

```text
000/
├── 001-bg-01-000-001.png
├── 001-bg-01-000-002.png
...
```

### 12.3 Decision

Initially official silhouette conversion was attempted, but missing output samples were found. Then user decided:

```text
Full clean YOLO silhouette extraction এ shift করা হবে
```

Later YOLO segmentation-based silhouette extraction was used.

---

## 13. Script 08 — Silhouette Extraction / Conversion

### 13.1 Official CASIA-B silhouette conversion attempt

A converter notebook converted official CASIA-B silhouettes to `.npz`.

Output metadata:

```text
source_official_sil_dir: /media/wadud/DriveUbuntu/CASIA/Main/DatasetB/silhouettes
output_root: /media/wadud/DriveUbuntu/GaitRecognition 2.0/data/silhouettes/casiab_official_64x44
output_tag: casiab_official_64x44
silhouette_height: 64
silhouette_width : 44
num_discovered_samples: 13593
num_converted_npz: 13593
expected_samples: 13640
```

Missing:

```text
Missing npz: 47
Missing/empty source dirs: 47
```

Example missing output samples included subject 005 under multiple conditions/views.

Analysis:

```text
⚠️ Official silhouette set was incomplete for our expected 13640 sample coverage
⚠️ Missing 47 samples could affect clean protocol
```

### 13.2 YOLO clean silhouette extraction

User decided to delete previous silhouettes and rerun clean YOLO extraction.

Model:

```text
yolo11m-seg.pt
```

Resolution choice:

```text
IMG_SIZE = 640
```

Discussion:

```text
IMG_SIZE=512 could be faster but may reduce mask quality.
IMG_SIZE=640 was chosen as a safer quality/speed balance.
```

GPU usage during silhouette extraction after performance mode:

```text
GPU utilization ≈ 86%
Memory ≈ 802 MiB / 6144 MiB
Temperature ≈ 85°C
Power ≈ 102W / 115W
```

Extraction speed improved somewhat after performance mode:

```text
Before: ~2.45 s/it
After : ~1.82 s/it
```

### 13.3 FP16 question

We discussed FP16 acceleration.

Conclusion:

```text
FP16 can reduce extraction time if YOLO inference supports half precision and GPU supports it.
```

However, final practical recommendation was:

```text
Use IMG_SIZE=640 for quality
Use performance mode
Use reasonable batch/worker settings
```

---

## 14. Script 09 — Validate Silhouette-Skeleton Alignment

### 14.1 Purpose

Validate whether silhouette `.npz` and skeleton `.npz` samples match by sample key and create fusion-ready split files.

Fusion rows require:

```text
subject
condition
seq
view
pose_path
silhouette_path
T_common
```

### 14.2 Initial validation successful output

A previous validation version ran successfully.

Fusion split summary examples:

```text
train_ST.csv     source_rows=2640  fusion_rows_clean=2640  missing_rows=0
test_ST.csv      source_rows=11000 fusion_rows_clean=10997 missing_rows=3
gallery_ST.csv   source_rows=4400  fusion_rows_clean=4397  missing_rows=3
probe_ST_nm.csv  source_rows=2200  fusion_rows_clean=2200  missing_rows=0
probe_ST_bg.csv  source_rows=2200  fusion_rows_clean=2200  missing_rows=0
probe_ST_cl.csv  source_rows=2200  fusion_rows_clean=2200  missing_rows=0

train_MT.csv     source_rows=6820 fusion_rows_clean=6820 missing_rows=0
test_MT.csv      source_rows=6820 fusion_rows_clean=6817 missing_rows=3
gallery_MT.csv   source_rows=2728 fusion_rows_clean=2725 missing_rows=3
probe_MT_nm.csv  source_rows=1364 fusion_rows_clean=1364 missing_rows=0
probe_MT_bg.csv  source_rows=1364 fusion_rows_clean=1364 missing_rows=0
probe_MT_cl.csv  source_rows=1364 fusion_rows_clean=1364 missing_rows=0

train_LT.csv     source_rows=8140 fusion_rows_clean=8140 missing_rows=0
test_LT.csv      source_rows=5500 fusion_rows_clean=5497 missing_rows=3
gallery_LT.csv   source_rows=2200 fusion_rows_clean=2197 missing_rows=3
probe_LT_nm.csv  source_rows=1100 fusion_rows_clean=1100 missing_rows=0
probe_LT_bg.csv  source_rows=1100 fusion_rows_clean=1100 missing_rows=0
probe_LT_cl.csv  source_rows=1100 fusion_rows_clean=1100 missing_rows=0
```

The 3 missing multimodal samples:

```text
subject: 109
condition: nm
seq: 01
views: 126, 144, 162
```

These affected gallery/test but not probe conditions.

### 14.3 Fixed validation error

A fixed version initially gave:

```text
NameError: name 'tqdm' is not defined
```

Cause:

```text
tqdm import missing before using tqdm(df_matched.iterrows())
```

Fix:

```python
from tqdm.notebook import tqdm
```

or fallback import.

### 14.4 Final alignment conclusion

```text
✅ Fusion splits created successfully
✅ Train LT mostly complete
✅ Probe NM/BG/CL complete
⚠️ Gallery LT has 3 missing samples from subject 109 nm-01 views 126/144/162
```

Question was asked whether keeping those 3 missing samples would cause big issue.

Conclusion:

```text
No major issue. Missing 3 gallery samples out of ~2200 is tiny.
It may slightly affect evaluation, but result remains meaningful.
```

---

# PART C — Silhouette Expert Training and Evaluation

---

## 15. Script 10 — Train Silhouette Expert: GaitGL-style Model

### 15.1 Model choice

Research discussion suggested silhouette-based models often perform very well for CASIA-B.

For implementation we used a practical **GaitGL-style silhouette expert**, lightweight but inspired by 3D convolution + horizontal pyramid pooling.

### 15.2 Input

Silhouette sequence input:

```text
B × T × H × W
T = 60
H = 64
W = 44
```

### 15.3 Architecture

Model name in scripts:

```text
GaitGLLiteBackbone / GaitGL-style expert
```

Core components:

```text
3D convolutional stem
Residual 3D blocks
Horizontal Pyramid Pooling 3D
Part-level projection
Embedding BN
L2 normalization
Classifier for training
```

#### 3D Conv Block

```text
Conv3D
BatchNorm3D
Mish activation
Dropout3D optional
```

#### Residual3DBlock

```text
ConvBNAct3D
Conv3D + BatchNorm3D
Residual shortcut
Mish activation
```

#### Channel configuration

Typical config used:

```text
channels = (32, 64, 128, 128)
embedding_dim = 256
hpp_bins = (1, 2, 4, 8)
dropout = 0.1
```

#### HPP bins

Horizontal Pyramid Pooling divides feature map height into bins:

```text
1 + 2 + 4 + 8 = 15 parts
```

Each part uses:

```text
mean pooling + max pooling
```

Then part features are projected and averaged to final embedding.

Final embedding:

```text
256-dimensional L2-normalized vector
```

### 15.4 Tiny sanity test

Tiny train graph showed:

```text
Loss decreased from around 1.4 to around 0.54
Batch CE accuracy increased to 1.0
```

Analysis:

```text
✅ Silhouette model can learn
✅ Data pipeline works
✅ Safe to run full training
```

### 15.5 Full training result

Final silhouette training output:

```text
Step: 29999 / 30000
First total loss: 4.2977...
Last total loss : 0.9831...
Min total loss  : 0.8395...
Last batch acc  : 1.0
Batch size      : 16
P = 8
K = 2
```

Training graph:

```text
- Loss starts around 4.3
- Decreases sharply until ~10000 steps
- Then slowly stabilizes around 0.9–1.0
- Batch CE accuracy reaches near 1.0 after early/mid training
```

Checkpoints:

```text
gaitgl_LT_silhouette_best_loss.pth
gaitgl_LT_silhouette_best_train_acc.pth
gaitgl_LT_silhouette_full_last.pth
gaitgl_LT_silhouette_last.pth
step checkpoints
```

### 15.6 Analysis

```text
✅ Silhouette expert trained successfully
✅ Training converged
✅ Final model memorized training identities reasonably
✅ Evaluation needed to judge generalization
```

---

## 16. Script 11 — Evaluate Silhouette Expert

### 16.1 Purpose

Evaluate silhouette expert checkpoints on CASIA-B LT split.

Evaluation:

```text
Gallery: gallery_LT_fusion.csv
Probe: probe_LT_nm_fusion.csv, probe_LT_bg_fusion.csv, probe_LT_cl_fusion.csv
Metric: Rank-1, Rank-5, Rank-10
Exclude identical view: True
```

### 16.2 Evaluated checkpoints

```text
gaitgl_LT_silhouette_last.pth
gaitgl_LT_silhouette_best_loss.pth
gaitgl_LT_silhouette_best_train_acc.pth
gaitgl_LT_silhouette_full_last.pth
```

### 16.3 Evaluation output examples

For `gaitgl_LT_silhouette_last.pth`:

```text
NM: Rank-1=97.55%, Rank-5=99.36%, Rank-10=99.82%
BG: Rank-1=85.64%, Rank-5=94.45%, Rank-10=96.36%
CL: Rank-1=65.55%, Rank-5=81.82%, Rank-10=88.91%
```

For `gaitgl_LT_silhouette_best_loss.pth`:

```text
NM Rank-1 ≈ 97.54%
BG Rank-1 ≈ 85.45%
CL Rank-1 ≈ 65.91%
Mean Rank-1 ≈ 82.97%
```

For `gaitgl_LT_silhouette_best_train_acc.pth`:

```text
NM Rank-1 ≈ 88.45%
BG Rank-1 ≈ 65.82%
CL Rank-1 ≈ 45.73%
Mean Rank-1 ≈ 66.67%
```

For `gaitgl_LT_silhouette_full_last.pth`:

```text
NM Rank-1 ≈ 97.54%
BG Rank-1 ≈ 85.63%
CL Rank-1 ≈ 65.54%
Mean Rank-1 ≈ 82.91%
```

### 16.4 Best silhouette checkpoint

Best by mean Rank-1:

```text
gaitgl_LT_silhouette_best_loss.pth
```

Best silhouette summary:

```text
NM ≈ 97.55%
BG ≈ 85.45%
CL ≈ 65.91%
Mean ≈ 82.97%
```

### 16.5 DataLoader warning

During evaluation, warnings appeared:

```text
Exception ignored in: _MultiProcessingDataLoaderIter.__del__
AssertionError: can only test a child process
```

Analysis:

```text
This is Jupyter multiprocessing cleanup warning.
Evaluation still completed.
Not a result-invalidating error.
```

Recommended mitigation:

```text
num_workers=0 for Jupyter safety
```

### 16.6 Analysis

Silhouette expert behavior:

```text
NM: excellent
BG: strong
CL: weak compared to skeleton
```

Interpretation:

```text
Silhouette captures body outline and appearance-motion, so normal and bag conditions work well.
But clothing variation changes silhouette shape heavily, so CL drops.
```

---

# PART D — Expert Comparison Before Fusion

---

## 17. Script 12 — Compare Skeleton vs Silhouette Experts

### 17.1 Purpose

Compare individual skeleton and silhouette experts before fusion.

### 17.2 Individual results

From final fusion table:

```text
Skeleton-only from embeddings:
NM = 95.00%
BG = 78.64%
CL = 74.18%
Mean = 82.61%

Silhouette-only from embeddings:
NM = 97.55%
BG = 85.45%
CL = 65.91%
Mean = 82.97%
```

### 17.3 Interpretation

Condition-wise best expert:

```text
NM: silhouette better
BG: silhouette better
CL: skeleton better
```

But both experts make different errors.

This suggested complementarity:

```text
Skeleton captures joint movement/posture.
Silhouette captures body shape/outline/motion appearance.
Errors are not identical.
Therefore fusion can improve.
```

### 17.4 Key research insight

```text
The individual expert means are similar (~82–83%), but their strengths differ by condition.
This is ideal for multimodal fusion.
```

---

# PART E — Skeleton Embedding Export for Fusion

---

## 18. Script 13A — Export Skeleton GaitTR Embeddings

### 18.1 Why this was needed

Initial fusion script had a missing functional part:

```text
Exact skeleton/GaitTR embedding exporter was missing.
```

Fusion needs paired embeddings:

```text
skeleton_train_LT_embeddings.npz
skeleton_gallery_LT_embeddings.npz
skeleton_probe_LT_nm_embeddings.npz
skeleton_probe_LT_bg_embeddings.npz
skeleton_probe_LT_cl_embeddings.npz

silhouette_train_LT_embeddings.npz
silhouette_gallery_LT_embeddings.npz
silhouette_probe_LT_nm_embeddings.npz
silhouette_probe_LT_bg_embeddings.npz
silhouette_probe_LT_cl_embeddings.npz
```

The fusion script initially had a fallback `FlexibleSkeletonTRBackbone`, but after uploaded train/eval scripts we saw actual GaitTR architecture was TCN+Spatial Transformer, not generic Transformer.

So a new exact exporter was created:

```text
13A_export_skeleton_gaittr_embeddings_FULL_FIXED.ipynb
```

### 18.2 Exporter architecture

It used exact GaitTR architecture:

```text
Input: 10 × 60 × 17
Backbone: TCN + Spatial Transformer blocks
Embedding: 128D normalized vector
```

### 18.3 Exported files

Output path:

```text
results/fusion_gate/expert_embedding_cache/
```

Created:

```text
skeleton_train_LT_embeddings.npz
skeleton_train_LT_meta.csv
skeleton_gallery_LT_embeddings.npz
skeleton_gallery_LT_meta.csv
skeleton_probe_LT_nm_embeddings.npz
skeleton_probe_LT_nm_meta.csv
skeleton_probe_LT_bg_embeddings.npz
skeleton_probe_LT_bg_meta.csv
skeleton_probe_LT_cl_embeddings.npz
skeleton_probe_LT_cl_meta.csv
```

### 18.4 Sanity evaluation after export

Output:

```text
NM: Rank-1=95.00%, Rank-5=97.18%, Rank-10=98.36%
BG: Rank-1=78.64%, Rank-5=88.82%, Rank-10=92.00%
CL: Rank-1=74.18%, Rank-5=86.27%, Rank-10=91.36%
```

Expected previous skeleton result was roughly:

```text
NM ≈ 94.91%
BG ≈ 79.27%
CL ≈ 75.18%
```

Difference:

```text
NM: +0.09
BG: -0.63
CL: -1.00
```

Analysis:

```text
✅ Skeleton exporter valid
✅ Checkpoint loaded correctly
✅ Feature builder correct
✅ Exported embeddings fusion-ready
```

---

# PART F — Fusion Pipeline

---

## 19. Script 13 — Fusion: Score Fusion + Adaptive Gated Fusion

### 19.1 Purpose

Combine skeleton expert and silhouette expert.

Two fusion stages:

```text
Stage A — Score-level fusion baseline
Stage B — Trainable adaptive fusion
```

### 19.2 Required embeddings

For each split:

```text
train
gallery
probe NM
probe BG
probe CL
```

Both modalities needed:

```text
Skeleton embedding
Silhouette embedding
```

### 19.3 Initial fusion error

Error:

```text
AssertionError: Need at least 8 identities, found 0
```

Cause:

```text
aligned['train'] was empty
```

Debug showed:

```text
Skeleton train rows   : 8139
Silhouette train rows : 2197
Overlap train keys    : 0
```

The silhouette train cache was wrong/old and had gallery-like size 2197 instead of train size.

### 19.4 Fix

A fix cell was used to:

```text
1. Detect zero overlap
2. Delete wrong silhouette_train cache
3. Re-extract silhouette train embeddings from df_train
4. Rebuild alignment
```

Fixed output:

```text
Re-extracted silhouette train: (8139, 256)
Aligned train: skeleton=(8139, 128), silhouette=(8139, 256), rows=8139
Fixed aligned train rows: 8139
Train subjects: 74
```

Then a force reload patch fixed stale in-memory state.

Final trainable fusion ready state:

```text
Skeleton train embeddings: 8139 × 128
Silhouette train embeddings: 8139 × 256
Train subjects: 74
```

### 19.5 Why 8139 instead of 8140?

Fusion split earlier had train LT 8140 rows, but final aligned train was 8139.

Interpretation:

```text
One sample likely missing/filtered in one modality/cache.
Not critical because 8139 rows and 74 identities are sufficient.
```

---

## 20. Fusion Methods Tested

### 20.1 Skeleton-only from embeddings

Uses only skeleton embedding similarity.

```text
score = skeleton_probe @ skeleton_gallery.T
```

Result:

```text
Mean = 82.61%
NM   = 95.00%
BG   = 78.64%
CL   = 74.18%
```

### 20.2 Silhouette-only from embeddings

Uses only silhouette embedding similarity.

```text
score = silhouette_probe @ silhouette_gallery.T
```

Result:

```text
Mean = 82.97%
NM   = 97.55%
BG   = 85.45%
CL   = 65.91%
```

### 20.3 Fixed score fusion alpha 0.5

Formula:

```text
final_score = 0.5 * skeleton_score + 0.5 * silhouette_score
```

Result:

```text
Mean = 95.73%
NM   = 99.36%
BG   = 95.91%
CL   = 91.91%
```

### 20.4 Best global alpha search

Alpha search over:

```text
alpha ∈ [0.0, 1.0]
```

where:

```text
final_score = (1 - alpha) * skeleton_score + alpha * silhouette_score
```

Best global alpha:

```text
alpha = 0.50
```

Same as fixed 0.5 fusion.

Result:

```text
Mean = 95.73%
NM   = 99.36%
BG   = 95.91%
CL   = 91.91%
```

### 20.5 Condition-guided score fusion

Hand-guided alpha based on expert behavior:

```text
NM/BG → silhouette weight higher
CL    → skeleton weight higher
```

Result:

```text
Mean = 94.88%
NM   = 99.64%
BG   = 95.45%
CL   = 89.55%
```

Analysis:

```text
Good, but worse than simple alpha=0.5.
```

### 20.6 Per-condition alpha diagnostic

This is oracle/diagnostic-like because it selects best alpha per condition using test condition outcome.

Result:

```text
Mean = 95.88%
NM   = 99.64%
BG   = 96.09%
CL   = 91.91%
```

Analysis:

```text
Best numerical result, but should not be claimed as practical main method because it uses condition-specific tuning based on evaluation.
```

### 20.7 Concatenation fusion baseline

Trainable method:

```text
concat = [skeleton_embedding, silhouette_embedding]
MLP projection → fused embedding
CE + triplet training
```

Result:

```text
Mean = 76.48%
NM   = 93.00%
BG   = 73.64%
CL   = 62.82%
```

Analysis:

```text
Not good. Projection disturbed expert embedding spaces and overfit training identities.
```

### 20.8 Adaptive scalar gate

Formula:

```text
fused = alpha * silhouette_projected + (1 - alpha) * skeleton_projected
```

Alpha is scalar per sample.

Architecture:

```text
Skeleton projection: Linear → LayerNorm → GELU → Dropout
Silhouette projection: Linear → LayerNorm → GELU → Dropout
Gate input: [fs, fi, |fs-fi|, fs*fi]
Gate MLP: Linear → LayerNorm → GELU → Dropout → Linear → GELU → Linear → Sigmoid
Final BN
Classifier
```

Result:

```text
Mean = 87.52%
NM   = 96.82%
BG   = 86.82%
CL   = 78.91%
```

Analysis:

```text
Better than concat/vector, but far below score fusion.
```

### 20.9 Adaptive vector gate

Same as scalar gate, but alpha is vector dimension-wise.

Result:

```text
Mean = 77.73%
NM   = 92.73%
BG   = 74.09%
CL   = 66.36%
```

Analysis:

```text
Poor generalization. Likely overfit and distorted embedding space.
```

---

## 21. Final Fusion Result Table

Final table from notebook 13:

```text
1. score_fusion_per_condition_alpha_diagnostic
   Mean = 95.8788
   BG   = 96.0909
   CL   = 91.9091
   NM   = 99.6364

2. score_fusion_best_global_alpha_0.50
   Mean = 95.7273
   BG   = 95.9091
   CL   = 91.9091
   NM   = 99.3636

3. score_fusion_fixed_alpha_0p5
   Mean = 95.7273
   BG   = 95.9091
   CL   = 91.9091
   NM   = 99.3636

4. score_fusion_condition_guided
   Mean = 94.8788
   BG   = 95.4545
   CL   = 89.5455
   NM   = 99.6364

5. adaptive_scalar_gate
   Mean = 87.5152
   BG   = 86.8182
   CL   = 78.9091
   NM   = 96.8182

6. silhouette_only_from_embeddings
   Mean = 82.9697
   BG   = 85.4545
   CL   = 65.9091
   NM   = 97.5455

7. skeleton_only_from_embeddings
   Mean = 82.6061
   BG   = 78.6364
   CL   = 74.1818
   NM   = 95.0000

8. adaptive_vector_gate
   Mean = 77.7273
   BG   = 74.0909
   CL   = 66.3636
   NM   = 92.7273

9. concat_fusion_baseline
   Mean = 76.4848
   BG   = 73.6364
   CL   = 62.8182
   NM   = 93.0000
```

---

## 22. Fusion Gain Analysis

### 22.1 Best practical result

Main practical method:

```text
score_fusion_fixed_alpha_0p5
```

or equivalent:

```text
score_fusion_best_global_alpha_0.50
```

Result:

```text
Mean Rank-1 = 95.73%
NM Rank-1   = 99.36%
BG Rank-1   = 95.91%
CL Rank-1   = 91.91%
```

### 22.2 Improvement over individual experts

Against skeleton-only:

```text
Mean: 95.73 - 82.61 = +13.12 percentage points
NM:   99.36 - 95.00 = +4.36
BG:   95.91 - 78.64 = +17.27
CL:   91.91 - 74.18 = +17.73
```

Against silhouette-only:

```text
Mean: 95.73 - 82.97 = +12.76 percentage points
NM:   99.36 - 97.55 = +1.81
BG:   95.91 - 85.45 = +10.46
CL:   91.91 - 65.91 = +26.00
```

### 22.3 Biggest improvement

Largest improvement:

```text
CL over silhouette-only: +26.00 percentage points
```

This is very important because CL is the hardest clothing variation condition.

### 22.4 Interpretation

```text
Skeleton and silhouette experts are complementary.
The fusion does not only average results; it combines different matching evidence.
```

---

## 23. Gate Alpha Analysis

### 23.1 Alpha meaning

Fusion formula:

```text
fused = alpha * silhouette + (1 - alpha) * skeleton
```

Alpha interpretation:

```text
alpha close to 1 → silhouette dominant
alpha close to 0 → skeleton dominant
```

### 23.2 Observed alpha values

Concat baseline:

```text
alpha = 0.5 fixed
```

Adaptive scalar gate:

```text
probe_NM alpha_mean ≈ 0.5478
probe_BG alpha_mean ≈ 0.5237
probe_CL alpha_mean ≈ 0.4716
gallery  alpha_mean ≈ 0.5478
```

Adaptive vector gate:

```text
probe_NM alpha_mean ≈ 0.5856
probe_BG alpha_mean ≈ 0.5846
probe_CL alpha_mean ≈ 0.5827
gallery  alpha_mean ≈ 0.5864
```

### 23.3 Analysis

Scalar gate learned slight condition behavior:

```text
NM/BG: alpha higher → silhouette slightly more weight
CL: alpha lower → skeleton slightly more weight
```

But difference was small.

Vector gate stayed almost constant:

```text
~0.58 across all conditions
```

Conclusion:

```text
Trainable gate did not learn strong useful condition-specific fusion.
Fixed score fusion remains best practical method.
```

---

# PART G — Literature and Publication-Level Interpretation

---

## 24. Is CL 91.91% new?

Conclusion:

```text
No, CL 91.91% is not absolute first / not absolute SOTA.
```

Known previous methods include:

```text
BiFusion: CL ≈ 92.1%
TriGait: CL ≈ 94.3%
Gait-TR: CL ≈ 90.0%
```

Our result:

```text
CL = 91.91%
```

So:

```text
✅ Better than many single-modality/skeleton baselines
✅ Better than Gait-TR reported CL-level result
⚠️ Slightly below BiFusion
⚠️ Below TriGait
```

### 24.1 Correct claim

Do not claim:

```text
We are the first to achieve 91%+ CL.
```

Safe claim:

```text
Our simple equal-weight score-level fusion achieves a competitive 91.91% Rank-1 accuracy under the challenging CL condition on CASIA-B, substantially outperforming both individual skeleton and silhouette experts.
```

---

## 25. Paper Potential Assessment

### 25.1 Is it paper-worthy?

Yes.

Current result is strong enough for a paper-style study, especially because:

```text
- Individual experts are around 82–83% mean Rank-1
- Simple score fusion reaches 95.73% mean Rank-1
- CL improves dramatically
- BG improves dramatically
```

### 25.2 Publication level assessment

Honest assessment:

```text
Q1 journal: Low chance currently
Q2 journal: Possible with stronger experiments and novelty
Q3 journal: Reasonably possible
Conference/workshop: Strong chance
None: No, it is definitely not “none”
```

### 25.3 Why Q1 is hard now

Reason:

```text
Main best method is simple late score fusion.
Late fusion is effective but not highly novel.
Absolute SOTA is not beaten.
```

Q1 would likely require:

```text
- More novel fusion method
- More datasets or protocols
- Strong ablations
- Statistical validation
- SOTA or near-SOTA with stronger novelty
```

### 25.4 Strong paper angle

Possible title:

```text
A Simple and Effective Score-Level Fusion of Skeleton and Silhouette Experts for Robust Gait Recognition
```

Main contribution:

```text
We show that independently trained skeleton and silhouette gait experts have complementary strengths, and a simple equal-weight score-level fusion significantly improves robust cross-view gait recognition, especially under bag and clothing variations.
```

---

# PART H — Final Technical Conclusion

---

## 26. What We Achieved Technically

### 26.1 Complete multimodal pipeline

We built:

```text
1. Skeleton extraction pipeline
2. Skeleton validation
3. Skeleton GaitTR training
4. Skeleton GaitTR evaluation
5. Silhouette extraction/conversion pipeline
6. Silhouette-skeleton alignment validation
7. Silhouette GaitGL-style training
8. Silhouette evaluation
9. Expert comparison
10. Skeleton embedding exporter
11. Multimodal fusion system
12. Score fusion + trainable fusion comparison
13. Final fusion report and plots
```

### 26.2 Best result

```text
Best practical method:
score_fusion_fixed_alpha_0p5

Mean Rank-1 = 95.73%
NM Rank-1   = 99.36%
BG Rank-1   = 95.91%
CL Rank-1   = 91.91%
```

### 26.3 Best numerical/diagnostic result

```text
score_fusion_per_condition_alpha_diagnostic

Mean Rank-1 = 95.88%
NM Rank-1   = 99.64%
BG Rank-1   = 96.09%
CL Rank-1   = 91.91%
```

But this is diagnostic/oracle-like, not main practical method.

### 26.4 Best research claim

```text
A simple equal-weight score-level fusion of skeleton-based GaitTR and silhouette-based GaitGL-style experts improves mean Rank-1 accuracy from approximately 83% for individual experts to 95.73%, with CL Rank-1 improving to 91.91%.
```

---

## 27. What Not to Claim

Do not claim:

```text
- This is absolute SOTA on CASIA-B.
- This is the first 91%+ CL result.
- Adaptive gate is the best fusion method.
- Trainable fusion outperforms score fusion.
```

Correct claims:

```text
- The result is highly competitive.
- Fusion strongly improves over both individual experts.
- Simple score-level fusion is the best practical method in our experiments.
- CL and BG improvements show strong modality complementarity.
```

---

## 28. Recommended Next Steps

### 28.1 Final analysis notebook

Create:

```text
14_final_fusion_analysis_and_report.ipynb
```

It should produce:

```text
- Skeleton vs silhouette vs fusion table
- Rank-1/Rank-5/Rank-10 table
- CMC curves
- Per-condition improvement plots
- Error/failure analysis
- Paper-ready report
```

### 28.2 Improve publication strength

For Q2-level target, add:

```text
1. More rigorous ablation
2. Alpha sweep plot
3. Fusion method comparison
4. Failure-case visualization
5. Confusion/error analysis by view angle
6. Statistical confidence or repeated run
7. Comparison with published methods
8. Clear protocol verification
```

### 28.3 Improve novelty

Potential new method direction:

```text
Reliability-aware score fusion
```

Instead of simple alpha, estimate reliability based on:

```text
- silhouette mask quality
- skeleton confidence
- sequence length
- view angle
- condition proxy
- embedding margin between top-1 and top-2
```

Then:

```text
final_score = reliability_skeleton * skeleton_score + reliability_silhouette * silhouette_score
```

This may give a more novel contribution than fixed 0.5 fusion.

---

# 29. Final One-Paragraph Summary

From scratch, we built a complete CASIA-B gait recognition workflow: YOLO-based skeleton extraction, skeleton validation, CASIA-B LT split preparation, GaitTR skeleton expert training/evaluation, YOLO/official silhouette processing, silhouette-skeleton alignment, GaitGL-style silhouette expert training/evaluation, expert comparison, exact GaitTR embedding export, and final multimodal fusion. The skeleton expert achieved around 82.61% mean Rank-1, while the silhouette expert achieved around 82.97% mean Rank-1. Their condition-wise strengths were complementary: silhouette was stronger on NM/BG, while skeleton was stronger on CL. Using simple equal-weight score-level fusion, the system achieved 95.73% mean Rank-1, with NM=99.36%, BG=95.91%, and CL=91.91%. This is not absolute SOTA, but it is a strong, competitive, publication-worthy result and clearly validates the central idea that skeleton and silhouette experts provide complementary gait information.

