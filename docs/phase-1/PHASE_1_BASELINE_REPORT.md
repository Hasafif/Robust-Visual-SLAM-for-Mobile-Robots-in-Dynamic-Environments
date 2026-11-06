# Phase 1 Baseline Report

## Robust Visual SLAM for Mobile Robots in Dynamic Environments

**Phase:** 1 --- Baseline Setup, Execution, and Evaluation\
**Dataset:** Bonn RGB-D Dynamic Dataset --- Balloon sequence\
**Primary baselines:** ORB-SLAM3 Monocular and ORB-SLAM3 RGB-D\
**Status:** Completed, with documented configuration and dataset
limitations

------------------------------------------------------------------------

## 1. Purpose and scope

Phase 1 establishes reproducible reference implementations against which
later dynamic-scene and AI-enhanced methods can be evaluated.

Two ORB-SLAM3 configurations were executed on the Bonn Balloon sequence:

1.  **Monocular:** RGB images only.
2.  **RGB-D:** RGB images paired with registered depth images.

The Monocular baseline is evaluated primarily with Sim(3) alignment
because monocular reconstruction has an unknown global scale. The RGB-D
baseline is evaluated primarily with SE(3) alignment because its depth
input provides metric scale.

This phase establishes baseline operation and evaluation. It does not
claim that the baseline is robust to dynamic objects, nor that the two
sensor configurations form a perfectly controlled head-to-head
comparison.

## 2. Environment and software

  Component              Recorded value
  ---------------------- ---------------------------------------------
  Operating system       Ubuntu 24.04.4
  Architecture           x86_64
  Python                 3.12.3
  OpenCV                 4.6.0 (system Python / system installation)
  EVO                    1.38.1
  Compiler               g++ 13.3.0
  CMake                  3.28.3
  Eigen                  3.4.0
  ORB-SLAM3 Git commit   `4452a3c4ab75b1cde34e5505a36ec3f9edcdc4c4`

The EVO package is installed in the project virtual environment at:

`~/graduation_project/evo-env`

The system Python was used for the depth-distribution inspection because
it provides the OpenCV Python bindings. The EVO virtual environment was
not modified to add OpenCV.

### Hardware record

The machine used was an ASUS TUF Gaming A16 FA607NUG. The GPU was
identified as an NVIDIA RTX 4050 Laptop GPU / AD107M in the system
inspection. NVIDIA driver availability was not established as part of
this phase, so GPU acceleration is not claimed.

## 3. Dataset and preparation

Dataset directory:

`~/graduation_project/datasets/rgbd_bonn_balloon`

The extracted sequence contains:

  Item                                   Entries / files
  ------------------------------------ -----------------
  RGB image files                                    439
  Depth image files                                  438
  `rgb.txt` entries                                  439
  `depth.txt` entries                                440
  Valid RGB-D associations generated                 438
  Ground-truth entries                               441

### 3.1 Image properties and depth units

RGB images are 640 × 480, 3-channel, 8-bit images. Depth images are 640
× 480, single-channel unsigned 16-bit images.

The RGB-D configuration uses:

`RGBD.DepthMapFactor: 5000.0`

The conversion used for the depth-distribution inspection was:

`depth_in_metres = raw_depth / 5000.0`

Across the 438 depth images, the inspection reported 115,766,180 nonzero
depth pixels:

  Statistic             Depth
  ----------------- ---------
  Minimum             0.661 m
  5th percentile      1.536 m
  10th percentile     1.853 m
  25th percentile     2.286 m
  Median              2.723 m
  75th percentile     3.064 m
  90th percentile     3.300 m
  95th percentile     3.433 m
  99th percentile     3.732 m
  Maximum             4.405 m

This distribution is a dataset-level pixel statistic. It is not the
distribution of ORB keypoint depths or MapPoint depths.

### 3.2 Missing depth files

The `depth.txt` list contains two entries whose corresponding image
files are absent from the extracted directory:

-   `1548266472.91872 depth/1548266472.91872.png`
-   `1548266480.49338 depth/1548266480.49338.png`

The ZIP archive was tested with `unzip -t` and reported no
compressed-data errors. Inspection of the archive listing confirmed that
these two image files are also absent from this particular ZIP archive.
This establishes the discrepancy in the local archive used for the
experiment; it does not establish that every upstream distribution of
the Bonn dataset has the same discrepancy.

The RGB timestamp list has 439 entries and the depth timestamp list has
440 entries. Their timestamp sequences are approximately 30 Hz and do
not show large timestamp gaps in the inspected interval.

### 3.3 RGB-D associations

A one-to-one RGB/depth association file was generated using global
minimum timestamp-difference matching, restricted to existing image
files and a 20 ms maximum difference.

File:

`datasets/rgbd_bonn_balloon/associations_rgbd.txt`

Recorded association summary:

  Measure                            Result
  ------------------------------ ----------
  Existing RGB images                   439
  Existing depth images                 438
  Valid associations                    438
  Unmatched RGB images                    1
  Unused depth images                     0
  Mean timestamp difference        8.604 ms
  Median timestamp difference       8.75 ms
  Maximum timestamp difference     16.67 ms

The association file contains data rows only. Comment/header lines were
removed because the ORB-SLAM3 `rgbd_tum` example parser used in this run
does not skip comment lines.

## 4. Camera and ORB-SLAM3 configuration

### 4.1 Monocular configuration

Configuration file:

`ORB_SLAM3/Examples/Monocular/BonnBalloon.yaml`

Recorded camera model and calibration:

  Parameter                     Value
  ---------------------- ------------
  Camera model                PinHole
  `Camera1.fx`             542.822841
  `Camera1.fy`             542.576870
  `Camera1.cx`             315.593520
  `Camera1.cy`             237.756098
  `Camera1.k1`               0.039903
  `Camera1.k2`              -0.099343
  `Camera1.p1`              -0.000730
  `Camera1.p2`              -0.000144
  `Camera1.k3`               0.000000
  `Camera.fps`                     30
  `Camera.RGB`                      1
  Image width × height      640 × 480

### 4.2 RGB-D configuration

Configuration file:

`ORB_SLAM3/Examples/RGB-D/BonnBalloon_RGBD.yaml`

The RGB-D configuration uses the same camera intrinsics listed above,
with:

  Parameter                  Value
  ----------------------- --------
  `Camera.fps`                  30
  `Camera.RGB`                   1
  `RGBD.DepthMapFactor`     5000.0
  `Stereo.ThDepth`            40.0
  `Stereo.b`                  0.08

**Important calibration qualification:** the Bonn-specific physical
value corresponding to `Stereo.b` was not found in the dataset
information inspected for this phase. The value `0.08` was used as an
ORB-SLAM3 RGB-D implementation parameter, close to values in ORB-SLAM3
example configurations. It must not be described as a measured or
officially calibrated physical baseline for the Bonn camera.

In ORB-SLAM3's RGB-D implementation, the depth image supplies the depth
measurements. The `b` parameter is used internally to form a synthetic
right-image coordinate from the RGB keypoint and measured depth, and it
also contributes to the close/far threshold. It is not evidence that the
dataset contains a second RGB camera.

With the chosen values, the effective close/far threshold is:

`Z_threshold = b × Stereo.ThDepth = 0.08 × 40 = 3.2 m`

The pixel-level depth distribution has approximately 83.04% of valid
pixels below 3.2 m and 16.96% at or above 3.2 m. This is contextual
information, not the actual close/far ratio of ORB keypoints used
internally by SLAM.

The selected `b` and threshold are therefore recorded as implementation
choices for this baseline, with a parameter-sensitivity study left as
future work. They are not claimed to be Bonn sensor calibration.

## 5. Execution and output trajectories

### 5.1 Monocular

Executable:

`ORB_SLAM3/Examples/Monocular/mono_tum`

Input: Bonn RGB sequence and `BonnBalloon.yaml`.

Output:

`results/bonn_balloon/KeyFrameTrajectory.txt`

Recorded output: 33 keyframe poses.

The ORB-SLAM3 TUM Monocular example saves the keyframe trajectory. In
the ORB-SLAM3 implementation used here, `SaveTrajectoryTUM()` is not
supported for the Monocular sensor mode, so a per-frame
`CameraTrajectory.txt` was not produced for this baseline.

### 5.2 RGB-D

Executable:

`ORB_SLAM3/Examples/RGB-D/rgbd_tum`

Inputs: Bonn RGB-D sequence, `BonnBalloon_RGBD.yaml`, and
`associations_rgbd.txt`.

Recorded output:

  Output                                                 Pose count
  ---------------------------------------------------- ------------
  `results/bonn_balloon/rgbd/CameraTrajectory.txt`              438
  `results/bonn_balloon/rgbd/KeyFrameTrajectory.txt`             49

Both SLAM runs processed their input sequences and reached
trajectory-save/shutdown output. The process then emitted Qt timer
warnings and a segmentation fault during shutdown. The trajectories were
written before that shutdown crash and were used for evaluation. This
shutdown issue is recorded separately from tracking and trajectory
generation; it should be investigated if it affects later automated
batch runs.

## 6. Evaluation protocol

Evaluation was performed with EVO 1.38.1 using TUM trajectory format and
the Bonn ground-truth trajectory:

`datasets/rgbd_bonn_balloon/groundtruth.txt`

Common synchronization settings:

-   `--sync_method interpolation`
-   `--t_max_diff 0.05`

APE is the absolute pose error, reported here for the translation part
in metres.

RPE is the relative pose error for the translation part, with:

-   `--pose_relation trans_part`
-   `--delta 1`
-   `--delta_unit f`
-   consecutive pose pairs

The reported RPE runs also used `--align`. Therefore, these RPE values
must be described with that alignment protocol; they are not unaligned
drift measurements.

Alignment policy:

-   Monocular primary evaluation: `--align --correct_scale` (Sim(3)).
-   RGB-D primary evaluation: `--align` (SE(3)).
-   Additional RGB-D Sim(3) keyframe evaluation was retained as a
    diagnostic.
-   Monocular SE(3) evaluation was also performed during investigation,
    but Sim(3) is the primary reported monocular result because of
    monocular scale ambiguity.

## 7. Results

### 7.1 Primary baseline metrics

  ------------------------------------------------------------------------
  Baseline     Trajectory   Alignment            APE RMSE         RPE RMSE
  ------------ ------------ ------------ ---------------- ----------------
  Monocular    Keyframes    Sim(3)             0.054545 m       0.164095 m
               (33 poses)                                 

  RGB-D        Camera       SE(3)              0.045059 m       0.025198 m
               trajectory                                 
               (438 poses)                                

  RGB-D        Keyframes    SE(3)              0.060843 m       0.119238 m
               (49 poses)                                 
  ------------------------------------------------------------------------

The Monocular APE and RPE are calculated from its keyframe trajectory.
The RGB-D camera-trajectory metrics use all 438 saved frame poses. These
rows are not a directly controlled comparison because the trajectory
sampling differs.

### 7.2 Full recorded metrics

#### Monocular --- KeyFrameTrajectory, Sim(3)

  Metric                            APE              RPE
  -------------------- ---------------- ----------------
  Maximum                    0.176065 m       0.708005 m
  Mean                       0.042530 m       0.105764 m
  Median                     0.034136 m       0.061056 m
  Minimum                    0.004259 m       0.020226 m
  RMSE                   **0.054545 m**   **0.164095 m**
  SSE                          0.098180         0.861674
  Standard deviation         0.034152 m       0.125464 m

#### RGB-D --- CameraTrajectory, SE(3)

  Metric                            APE              RPE
  -------------------- ---------------- ----------------
  Maximum                    0.212318 m       0.233147 m
  Mean                       0.035176 m       0.019077 m
  Median                     0.028210 m       0.015953 m
  Minimum                    0.003544 m       0.002007 m
  RMSE                   **0.045059 m**   **0.025198 m**
  SSE                          0.889292         0.277474
  Standard deviation         0.028161 m       0.016462 m

#### RGB-D --- KeyFrameTrajectory, SE(3)

  Metric                            APE              RPE
  -------------------- ---------------- ----------------
  Maximum                    0.208523 m       0.322653 m
  Mean                       0.047797 m       0.099901 m
  Median                     0.038159 m       0.085698 m
  Minimum                    0.010060 m       0.014791 m
  RMSE                   **0.060843 m**   **0.119238 m**
  SSE                          0.181391         0.682455
  Standard deviation         0.037647 m       0.065097 m

#### RGB-D --- KeyFrameTrajectory, Sim(3) diagnostic

  Metric                            APE              RPE
  -------------------- ---------------- ----------------
  Maximum                    0.208337 m       0.316391 m
  Mean                       0.048737 m       0.097354 m
  Median                     0.039878 m       0.083204 m
  Minimum                    0.017198 m       0.014486 m
  RMSE                   **0.059503 m**   **0.116291 m**
  SSE                          0.173491         0.649136
  Standard deviation         0.034136 m       0.063608 m

### 7.3 Interpretation and comparison limitations

The RGB-D camera trajectory achieved an APE RMSE of approximately 4.51
cm and RPE RMSE of approximately 2.52 cm under the stated SE(3)-aligned
protocol. The Monocular keyframe trajectory achieved an APE RMSE of
approximately 5.45 cm under Sim(3) alignment, with RPE RMSE
approximately 16.41 cm under its stated aligned protocol.

These figures describe the observed runs on this sequence. They do not
establish a universal performance ranking. The Monocular and RGB-D
trajectories differ in sampling density, number of poses, and alignment
treatment. Even the keyframe trajectories have different keyframe counts
(33 versus 49) and timestamps. The results should therefore be treated
as separate reference baselines, not as a perfectly controlled
sensor-only comparison.

## 8. Result files and reproducibility record

The principal files are stored in the working experiment directory:

``` text
~/graduation_project/
├── ORB_SLAM3/
│   └── Examples/
│       ├── Monocular/BonnBalloon.yaml
│       └── RGB-D/BonnBalloon_RGBD.yaml
├── datasets/
│   └── rgbd_bonn_balloon/
│       ├── associations_rgbd.txt
│       └── groundtruth.txt
└── results/
    └── bonn_balloon/
        ├── KeyFrameTrajectory.txt
        ├── monocular/
        │   ├── ape_keyframes_sim3.zip
        │   └── rpe_keyframes_sim3.zip
        └── rgbd/
            ├── CameraTrajectory.txt
            ├── KeyFrameTrajectory.txt
            ├── ape_camera.zip
            ├── rpe_camera.zip
            ├── ape_keyframes_se3.zip
            ├── rpe_keyframes_se3.zip
            ├── ape_keyframes_sim3.zip
            └── rpe_keyframes_sim3.zip
```

All nine EVO ZIP archives in `results/bonn_balloon` passed `unzip -t`
integrity checks.

### 8.1 SHA-256 fingerprints

These hashes identify the exact configuration and association files used
in the recorded run:

  -------------------------------------------------------------------------------------------------------------------------
  File                                                 SHA-256
  ---------------------------------------------------- --------------------------------------------------------------------
  `ORB_SLAM3/Examples/Monocular/BonnBalloon.yaml`      `4e3226c8f111e0aeb417a7f24c15f2afd18eed2486edbacf5fdd64732f24e408`

  `ORB_SLAM3/Examples/RGB-D/BonnBalloon_RGBD.yaml`     `f82583ad51b276732700fbc9963e3288f6466062925413040d9a99c60827330f`

  `datasets/rgbd_bonn_balloon/associations_rgbd.txt`   `5c58de407f533428c9456ea0027023d8b0cf55366e8de0a7e9998fe63719080c`
  -------------------------------------------------------------------------------------------------------------------------

ORB-SLAM3 source commit:

`4452a3c4ab75b1cde34e5505a36ec3f9edcdc4c4`

EVO version:

`1.38.1`

### 8.2 Re-evaluation commands

Run these from `~/graduation_project` with the `evo-env` virtual
environment activated.

Monocular APE (Sim(3)):

``` bash
evo_ape tum \
  datasets/rgbd_bonn_balloon/groundtruth.txt \
  results/bonn_balloon/KeyFrameTrajectory.txt \
  --align --correct_scale \
  --sync_method interpolation \
  --t_max_diff 0.05 \
  --save_results results/bonn_balloon/monocular/ape_keyframes_sim3.zip
```

Monocular RPE (Sim(3)):

``` bash
evo_rpe tum \
  datasets/rgbd_bonn_balloon/groundtruth.txt \
  results/bonn_balloon/KeyFrameTrajectory.txt \
  --pose_relation trans_part \
  --delta 1 --delta_unit f \
  --align --correct_scale \
  --sync_method interpolation \
  --t_max_diff 0.05 \
  --save_results results/bonn_balloon/monocular/rpe_keyframes_sim3.zip
```

RGB-D CameraTrajectory APE (SE(3)):

``` bash
evo_ape tum \
  datasets/rgbd_bonn_balloon/groundtruth.txt \
  results/bonn_balloon/rgbd/CameraTrajectory.txt \
  --align \
  --sync_method interpolation \
  --t_max_diff 0.05 \
  --save_results results/bonn_balloon/rgbd/ape_camera.zip
```

RGB-D CameraTrajectory RPE (SE(3)):

``` bash
evo_rpe tum \
  datasets/rgbd_bonn_balloon/groundtruth.txt \
  results/bonn_balloon/rgbd/CameraTrajectory.txt \
  --pose_relation trans_part \
  --delta 1 --delta_unit f \
  --align \
  --sync_method interpolation \
  --t_max_diff 0.05 \
  --save_results results/bonn_balloon/rgbd/rpe_camera.zip
```

RGB-D KeyFrameTrajectory APE (SE(3)):

``` bash
evo_ape tum \
  datasets/rgbd_bonn_balloon/groundtruth.txt \
  results/bonn_balloon/rgbd/KeyFrameTrajectory.txt \
  --align \
  --sync_method interpolation \
  --t_max_diff 0.05 \
  --save_results results/bonn_balloon/rgbd/ape_keyframes_se3.zip
```

RGB-D KeyFrameTrajectory APE (Sim(3), diagnostic):

``` bash
evo_ape tum \
  datasets/rgbd_bonn_balloon/groundtruth.txt \
  results/bonn_balloon/rgbd/KeyFrameTrajectory.txt \
  --align --correct_scale \
  --sync_method interpolation \
  --t_max_diff 0.05 \
  --save_results results/bonn_balloon/rgbd/ape_keyframes_sim3.zip
```

RGB-D KeyFrameTrajectory RPE (SE(3)):

``` bash
evo_rpe tum \
  datasets/rgbd_bonn_balloon/groundtruth.txt \
  results/bonn_balloon/rgbd/KeyFrameTrajectory.txt \
  --pose_relation trans_part \
  --delta 1 --delta_unit f \
  --align \
  --sync_method interpolation \
  --t_max_diff 0.05 \
  --save_results results/bonn_balloon/rgbd/rpe_keyframes_se3.zip
```

RGB-D KeyFrameTrajectory RPE (Sim(3), diagnostic):

``` bash
evo_rpe tum \
  datasets/rgbd_bonn_balloon/groundtruth.txt \
  results/bonn_balloon/rgbd/KeyFrameTrajectory.txt \
  --pose_relation trans_part \
  --delta 1 --delta_unit f \
  --align --correct_scale \
  --sync_method interpolation \
  --t_max_diff 0.05 \
  --save_results results/bonn_balloon/rgbd/rpe_keyframes_sim3.zip
```

## 9. Known limitations and follow-up work

1.  **RGB-D virtual baseline parameter:** `Stereo.b: 0.08` is not
    confirmed as a Bonn-specific physical calibration. Keep it
    explicitly documented as an implementation parameter. A sensitivity
    experiment can be performed later, with each setting recorded as a
    separate run.
2.  **Missing depth images:** two depth-list entries have no
    corresponding files in the local ZIP. The current RGB-D experiment
    uses the 438 valid associations and documents the one unmatched RGB
    frame.
3.  **Different trajectory sampling:** Monocular produces keyframe poses
    in this example, while RGB-D also produces a per-frame camera
    trajectory. Do not compare metrics across these different sampling
    densities as if they were identical protocols.
4.  **Different keyframe samples:** Monocular has 33 keyframes and RGB-D
    has 49. Keyframe-to-keyframe metrics are informative but are not a
    perfectly matched comparison.
5.  **Aligned RPE:** all reported RPE values used alignment. Preserve
    this fact in tables and discussion.
6.  **Shutdown segmentation fault:** both example runs emitted Qt timer
    warnings and a segmentation fault after trajectory output. Preserve
    the generated trajectories, but investigate the shutdown issue if it
    disrupts future automated experiments.
7.  **Dynamic-scene robustness:** this phase establishes standard
    ORB-SLAM3 baselines only. It does not yet implement semantic
    masking, dynamic feature rejection, optical flow, learned features,
    or other proposed improvements.
8.  **Monocular trajectory export:** the Monocular example output used
    here is a keyframe trajectory; no per-frame Monocular camera
    trajectory was produced by the standard save method in this
    configuration.

## 10. Phase 1 completion statement

Phase 1 is considered complete for the scoped objective of establishing
and evaluating reproducible Monocular and RGB-D ORB-SLAM3 reference runs
on the Bonn Balloon sequence.

The baseline trajectories, EVO evaluation archives, configuration files,
association file, source commit, software versions, and configuration
hashes have been recorded. The limitations above remain explicit and
must be carried forward into later experimental design.

**Baseline freeze:** Do not overwrite these recorded outputs when
testing new methods or parameter settings. Store future experiments in
separate, clearly named result directories so that the Phase 1 reference
remains unchanged.
