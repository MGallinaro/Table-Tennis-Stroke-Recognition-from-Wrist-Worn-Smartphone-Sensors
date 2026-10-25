# Table-Tennis Stroke Recognition from Wrist-Worn Smartphone Sensors

Real-time recognition of three table-tennis strokes (**serve**, **forehand topspin**, **backhand topspin**) from the accelerometer and gyroscope of a smartphone worn on the wrist. The project uses MATLAB and a 1D convolutional neural network, and runs in Simulink on an Android phone.

This repository is a **rework** of a university group project (Automation and Safety course, UCBM, 2024). The original system reached 98% accuracy offline but only 40% in the real-time test on the phone. I rebuilt the whole pipeline with one rule: **every sample that reaches the network goes through exactly the same processing in training, in testing and in real time.**

📄 **Full report:** [`docs/report.pdf`](docs/report.pdf) covers the challenges, the decisions behind them and the results.

---

## Results

| | Original project | This rework |
| --- | --- | --- |
| Offline accuracy | 98.2% (random split of windows, inflated by data leakage) | **95.5%** on subjects never seen in training (969 / 1015 windows) |
| Real-time accuracy on the phone | 40% (6 / 15 trials) | **91.7%** (55 / 60 strokes: serve 90%, forehand 90%, backhand 95%) |

The real-time result comes from **one session by one person** (the author, a test subject never used for training), with 20 strokes per class. It shows that the system works in real time; it is not yet a general accuracy figure.

## What was fixed

- **Corrupted labels.** The "rest" class in the dataset is an exact copy of the backhand class, and one recording is duplicated. Both were excluded.
- **Wrong sampling rate.** The files store 400 rows per second, but the sensor delivered about **50 values per second**, each value repeated until the next one. All processing now runs at 50 Hz, the rate the phone actually provides.
- **Filters that could not run in real time.** A causal high-pass filter is used instead of `filtfilt`, so it gives the same result offline and sample by sample.
- **Windows cut differently offline and online.** Each window is 1 s centred on the stroke peak, both in training and on the phone, where the sliding buffer itself becomes the window.
- **Data leakage.** The data are split **by subject** (train / validation / test with different people), and normalization statistics come from the training set only.
- **Position-dependent model.** A 1D CNN with global average pooling (11,683 parameters) replaces a feed-forward network on the flattened window.

## Pipeline

```
raw recording ─► decimation to 50 Hz ─► causal high-pass filter (0.5 Hz)
             ─► stroke detection on the acceleration magnitude (adaptive threshold)
             ─► 1 s window centred on the peak (50 samples × 6 channels)
             ─► normalization (training statistics) ─► 1D CNN ─► S / TPD / TPR
```

Online, the same steps run **one sample at a time** inside `passoOnline.m`, which keeps its memory (filter state, buffer, recent history) between calls. In Simulink this function sits in a MATLAB Function block and the network in a Predict block.

## Repository structure

```
├── data/
│   └── dataset_anon.mat          anonymized recordings (see "Data")
├── models/
│   └── cnn_model.mat             trained network + normalization statistics + class names
├── src/
│   ├── preprocessing_pipeline.m  raw data → windows.mat (filtering, stroke detection, windows)
│   ├── train_cnn.m               windows.mat → cnn_model.mat (split, normalization, training, test)
│   ├── online_simulation.m       replays a recording one sample at a time and verifies the online function
│   ├── passoOnline.m             real-time stroke detector (one sample per call), used in Simulink
│   ├── passoOnline_debug.m       same algorithm, with internal signals as extra outputs (for verification)
│   ├── exploration.m             archive of the analyses behind the design choices
│   └── anonymize_dataset.m       how the original names were replaced by codes (needs the original file)
└── docs/
    └── report.pdf                full project report
```

Variable names in the code are in Italian (for example `soglia` = threshold, `finestra` = window, `colpo` = stroke); all comments are in English.

## How to run

**Requirements:** MATLAB R2026a (earlier recent releases should work) with the Signal Processing, Statistics and Machine Learning, and Deep Learning Toolboxes. Training is set to run on a GPU (`ExecutionEnvironment="gpu"` in `train_cnn.m`); change it to `"auto"` or `"cpu"` if you do not have one.

1. Open MATLAB and set `src/` as the current folder.
2. **Full pipeline:**
   1. `preprocessing_pipeline.m`: when asked, select `data/dataset_anon.mat`. It creates `windows.mat` and `preprocessing.mat`.
   2. `train_cnn.m`: trains the network and creates `cnn_model.mat`. Expected test accuracy is about 95%; small differences between runs are normal on a GPU.
   3. `online_simulation.m`: select `data/dataset_anon.mat` again. It replays a test recording sample by sample and prints the checks (all differences 0, all comparisons 1).
3. **Quick start without training:** copy `models/cnn_model.mat` into `src/` and run only `online_simulation.m`.

## Data

`data/dataset_anon.mat` contains the struct `dataset`:

```
dataset.<activity>.<recording>.Acc    24001 × 3   accelerometer x, y, z (m/s²)
dataset.<activity>.<recording>.Gyro   24001 × 3   gyroscope x, y, z (rad/s)
```

- **Activities:** `S` (serve), `TPD` (forehand topspin), `TPR` (backhand topspin), `F` (intended as "rest").
- **Recordings:** `P01_File1` … `P11_File5`, i.e. 11 subjects × 5 recordings of 60 s per activity.
- **Sampling:** 400 rows per second in the file, but the sensor delivered about 50 values per second (each value held until the next one). The scripts keep one row in 8.
- **Known issues** (handled in the code, left untouched in the file):
  - `F` is an exact copy of `TPR`, so it cannot be used as a rest class;
  - `P01_File5` duplicates `P01_File4` in `TPD` and `TPR`;
  - in `TPR`, `P01_File1` and `P01_File2` contain continuous movement and are excluded by the pipeline.

Subjects are identified only by anonymous codes; the key linking codes to names is not published. The data were collected by the project group during the course. If you want to reuse them, please contact the author.

## Real-time version (Simulink, Android)

The online detector was moved into a Simulink model, verified on the PC against MATLAB (same strokes, same classes), and then compiled into an Android app with Simulink Coder, Embedded Coder and the Simulink Support Package for Android Devices. The app reads the phone sensors every 20 ms and shows the class of the last stroke and a stroke counter on screen. Details and the problems solved along the way are in Section 4.10 of the [report](docs/report.pdf).

## Limitations

- Only two test subjects offline; a leave-one-subject-out evaluation is planned.
- Real-time accuracy measured in one session by one person.
- No real "rest" class: "no stroke" is decided by the detector.
- Continuous movements (some backhands) can defeat the median-based threshold.

## License

The code in `src/` is released under the MIT License (see [`LICENSE`](LICENSE)). The dataset is **not** covered by this license (see "Data").

## Author

**Marco Gallinaro**
