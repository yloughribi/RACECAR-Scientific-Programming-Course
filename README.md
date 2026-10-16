# Visualising an overtake in the RACECAR dataset

From a whole-track LiDAR map to the view of the passing car's camera.

This project uses the [RACECAR dataset](https://github.com/linklab-uva/RACECAR_DATA) (nuScenes format, autonomous race cars at high speed) to combine data from two cars into one 28-second video of a real overtake. It also tests whether the passing car can be found in the LiDAR without hand-made labels.

## What is in this repository

| File | Content |
|---|---|
| `explore.ipynb` | The full analysis, with saved outputs (it renders on GitHub) |
| `requirements.txt` | Python packages |
| `figures/` | Figures produced by the notebook |

The extracted sensor data (`data/`) and the rendered videos (`videos/`) are not stored here because of their size. The notebook downloads and recreates them.

## The question

How can data from several sensors, recorded by two different cars, be brought into one coherent picture of a manoeuvre that lasts only a few seconds?

## Method in short

1. **Explore the metadata.** Only the small index tables are read first, to find which recordings contain two cars with overlapping logs. Pair found: MIT-PITT-RW (camera car) and KAIST, 2022-01-02, overlapping by about 699 s.
2. **Find the overtake.** The relative position of the two cars gives five overtakes. Pass 1 (66.4 mph / 106.9 km/h, about 7 m lateral gap) is used.
3. **Fix the data.** Timestamps were normalised per value, the stored orientation had a constant 90 degree offset, and the two logs had a clock offset.
4. **Project the LiDAR onto the camera.** Points are moved from the sensor frame to the vehicle frame, to the world with the ego pose, back to the vehicle at the photo's time, into the camera frame, and then to pixels with a pinhole model.
5. **Render the video.** Five pieces (map sweep, transition, cockpit view, virtual camera, real camera) are joined, and the four seams are checked.

## Extension: a detector without labels

- **Stage 1** builds a persistence map over 8 laps (background = places occupied on almost every lap), then clusters the remaining points and keeps car-sized clusters. No training.
- **Stage 2** is a small U-Net (about 144,000 parameters) trained on bird's-eye-view images of the scans, using only Stage 1's detections as labels.
- KAIST's own log is used for scoring only, never for finding or training.

Held-out results (151 scans from two encounters; KAIST within range in 72):

| | Recall | Precision |
|---|---|---|
| Stage 1 (map + clustering) | 78% (56/72) | 100% |
| Stage 2 (U-Net) | 60% (43/72) | 98% |

Both stages are near-perfect up to 40 m. Beyond 40 m Stage 1 finds 11 of 24 and Stage 2 finds none.

## Limitations

- One other car and two held-out encounters: a proof of concept, not a benchmark.
- The clock offset that fits the camera photo (-0.12 s) differs from the one that fits the LiDAR (+0.10 s). The cause was not tested.
- The camera delivers about 4.6 photos per second, so frames are interpolated.
- The camera projection is a pinhole model without lens distortion.

## How to run

1. Python 3.10 or newer is recommended.
2. Create an environment and install the packages:
   ```
   python -m venv .venv
   .venv\Scripts\activate        # Windows (macOS/Linux: source .venv/bin/activate)
   pip install -r requirements.txt
   pip install scikit-learn torch   # needed for the extension in Section 7
   ```
3. Open `explore.ipynb` from the project folder (so that `data/`, `figures/` and `videos/` are created next to it) and run all cells.

Notes:
- The data come from the public S3 bucket `racecar-dataset` (region us-west-2). No credentials are needed.
- A full run downloads about 1.5 GB of sensor files plus the LiDAR keyframes of the whole log, and it takes a while. Rendering the video is the slowest step.
- Results of the neural network may differ slightly between runs and machines (CPU, thread count).

## Data and references

- RACECAR dataset: Kulkarni et al., *RACECAR - The Dataset for High-Speed Autonomous Racing*.
- MODEST (arXiv 2203.15882) and OYSTER (arXiv 2311.02007): self-training object detection from repeated traversals, the idea behind Stage 1.

## Author

Yahya Loughribi, Maastricht University
