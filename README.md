# Bag Image Export (`bag_metashape_export`)

Extracts the down-facing and forward-facing camera streams from one ROS1 `.bag`, joins each frame to an interpolated vehicle pose, and writes Agisoft Metashape import files for the operator who is starting a 3D reconstruction from an AUV dive.

- Tags: AUV, 3D | Version: 1.0.0 | Type: extract | Status: dataset descriptors in draft | Owner: Lauren K Olinger
- Repo: `vicarius/modules/bag_metashape_export/github_repo` | Related studies: S1_historical, S2_3D_structure, S3_colony_tracking

## What it does and why it exists

Agisoft Metashape builds a georeferenced 3D reconstruction from two things: a folder of images and a reference table that gives a position and an attitude for each image. A rangerbot-class AUV does not record either of those directly. It records a ROS1 `.bag`, a single file that interleaves raw camera messages and navigation messages on their own clocks. Someone has to decode the camera messages into JPEGs and align every frame to the vehicle track before Metashape can use them. This module does that decoding and alignment.

The module reads one `.bag`, extracts the down-facing 4K science camera and the forward-facing ZED2 left camera to JPEG, linearly interpolates the `/pose` navigation track to each frame, and writes one Metashape-format georeference CSV per camera. It also renders a mission-map PNG so the operator can review the track and the sample counts before committing to a reconstruction.

The operator runs this module once per bag, right after a dive and before opening Metashape. It is the first step in the 3D reconstruction workflow. The module extracts frames and reference tables; it does not run photogrammetry and it does not build a model.

The extractor is the entry point from bag to frames. It exists so that the reconstruction workflow can start from a standard Metashape image folder plus reference CSV instead of a raw ROS bag.

## Where it sits in the pipeline

The upstream input is a ROS1 `.bag` produced by the AUV during a dive. The vehicle recorder writes the bag; no VICARIUS module produces it. The operator supplies the bag path at run time.

The immediate downstream consumer is Agisoft Metashape, run by hand outside VICARIUS. The operator adds the extracted images as photos, imports the matching reference CSV, and builds the reconstruction. This module produces the Metashape inputs; the Metashape reconstruction itself is a manual step and is not part of this module.

Within the platform survey chain, this module is the first link: `bag_metashape_export -> 3D_phase1 -> 3D_phase2 -> reef_point_seg`. The reconstruction that Metashape builds from these outputs feeds the later 3D phases and, downstream of those, `reef_point_seg`. The exact hand-off contract to `3D_phase1` is not defined in the files of this module and is not verified here.

## Inputs

| Input | Type | Formats | Required | What it is |
|-------|------|---------|----------|------------|
| `bag_file` | file | `.bag` | yes | One ROS1 bag file that holds the camera image topics and the `/pose` navigation track. The path is supplied at run time and normally points at a file on the NAS. |

The bag must contain the `/pose` topic and at least one of the two camera topics. The script registers the custom message types of the bag from the `msgdef` embedded in each connection, so no external ROS message package has to be installed to read a custom type such as `rangerbot_msgs/pose`.

The three topics the script reads:

| Topic | Message type | Role |
|-------|--------------|------|
| `/pose` | `rangerbot_msgs/pose` | Navigation track. Each sample carries `x` (longitude), `y` (latitude), `depth`, `altitudeUsed` (DVL altitude above the seafloor), `heading`, `pitch`, `roll`. This topic is required; if it is absent or empty the run aborts. |
| `/science/image_raw` | `sensor_msgs/Image` | Down-facing 4K science camera. Drives `down_images/` and `down_reference.csv`. |
| `/zed2/zed_node/left/image_rect_color` | `sensor_msgs/Image` | Forward-facing ZED2 left camera. Drives `forward_images/` and `forward_reference.csv`. |

If a camera topic is missing, the script skips that camera and continues with the other. Other topics that can appear in these bags (`/pressure`, `/dvl_dr`, `/dvl_fix`) are declared in `module.yaml` as optional and are not read by the extractor. The script reads the `depth` field of `/pose` into memory but never writes it to any output.

## Parameters

| Parameter | Type | Default | What it controls |
|-----------|------|---------|------------------|
| `output_dir` | string | `.` | Base output directory. The script creates a subfolder named after the bag file inside this directory and writes every artifact under that subfolder. When the VICARIUS runner launches the module, it sets this to the `outputs/` directory of the run folder, the per-run directory the VICARIUS launcher creates. |

The module declares no other parameters. There is no `cli.flag_map` in `module.yaml`, so the only runtime settings are the two positional arguments (`bag_file` and `output_dir`). Camera topics, JPEG quality, and interpolation method are fixed in the script and are not exposed as parameters.

## Outputs and data-dictionary entries

The script creates `<output_dir>/<bag_name>/` and writes everything under it. `<bag_name>` is the bag filename without its `.bag` extension. When the module runs under the VICARIUS runner, `<output_dir>` is the `outputs/` directory of the run folder, so the full path template is `inprocess/{run_name}/outputs/{bag_name}/`. Here `{run_name}` is the run folder name the launcher mints and `{bag_name}` is the bag stem.

Every artifact binds to a data-dictionary descriptor, the YAML file in the data dictionary that documents that output. Each descriptor is `status: draft`. With `eml: true` on all five outputs, the runner emits an EML 2.2.0 sidecar (an Ecological Metadata Language XML metadata file) alongside each artifact at run finalisation.

| Output dataset | Path template | Descriptor | What it is |
|----------------|---------------|------------|------------|
| `bag_metashape_export_down_images` | `inprocess/{run_name}/outputs/{bag_name}/down_images/` | [down_images](../../../_METADATA/dictionary/datasets/bag_metashape_export_down_images.yaml) | Directory of JPEG frames from the down-facing 4K science camera. Filenames are `down_{NNNN}.jpg`, where `NNNN` is the zero-padded four-digit message index in bag order (`down_0000.jpg`, `down_0001.jpg`, ...). The script decodes each frame to BGR per its ROS encoding and re-encodes it as JPEG at OpenCV quality 95. The resolution follows the source camera, and the script does not resize it. |
| `bag_metashape_export_forward_images` | `inprocess/{run_name}/outputs/{bag_name}/forward_images/` | [forward_images](../../../_METADATA/dictionary/datasets/bag_metashape_export_forward_images.yaml) | Directory of JPEG frames from the forward-facing ZED2 left-rectified camera. Filenames are `forward_{NNNN}.jpg` with the same four-digit index scheme and the same quality-95 JPEG encoding. |
| `bag_metashape_export_down_reference` | `inprocess/{run_name}/outputs/{bag_name}/down_reference.csv` | [down_reference](../../../_METADATA/dictionary/datasets/bag_metashape_export_down_reference.yaml) | Metashape camera-reference table for the down camera. One header row, then one row per down frame. Seven columns in Metashape import order: `label`, `longitude`, `latitude`, `altitude`, `yaw`, `pitch`, `roll` (WGS84 / EPSG:4326, attitude in degrees). |
| `bag_metashape_export_forward_reference` | `inprocess/{run_name}/outputs/{bag_name}/forward_reference.csv` | [forward_reference](../../../_METADATA/dictionary/datasets/bag_metashape_export_forward_reference.yaml) | Metashape camera-reference table for the forward camera. Same seven-column schema as `down_reference.csv`; the camera source and the `label` prefix differ. |
| `bag_metashape_export_mission_map` | `inprocess/{run_name}/outputs/{bag_name}/mission_map.png` | [mission_map](../../../_METADATA/dictionary/datasets/bag_metashape_export_mission_map.yaml) | One QA PNG. A two-panel matplotlib figure (14 by 7 inches, saved at 150 dpi): the left panel plots the `/pose` longitude against latitude, colored by time progression, with green Start and red End markers on equal-aspect Cartesian axes; the right panel is a monospace mission-statistics block. The map is for review only. It is not georeferenced and is not a Metashape input. |

### Reference CSV schema

`down_reference.csv` and `forward_reference.csv` share one layout. Row 1 is the header; data starts on row 2. The script builds the pose values by linear interpolation of the `/pose` track (`scipy.interpolate.interp1d`, `fill_value='extrapolate'`) to each frame, then converts the attitude channels from radians to degrees.

| Column | Name | Content |
|--------|------|---------|
| 1 | `label` | Frame filename, for example `down_0000.jpg`. The join key to the image in `down_images/` or `forward_images/`. |
| 2 | `longitude` | Decimal degrees, WGS84 (EPSG:4326), interpolated from the `/pose` `x` field. |
| 3 | `latitude` | Decimal degrees, WGS84 (EPSG:4326), interpolated from the `/pose` `y` field. |
| 4 | `altitude` | Meters, interpolated from the `/pose` `altitudeUsed` field (DVL altitude above the seafloor). This is the only vertical channel written. `depth` is read but never exported. |
| 5 | `yaw` | Degrees, from `/pose` `heading` (radians converted to degrees). |
| 6 | `pitch` | Degrees, from `/pose` `pitch` (radians converted to degrees). |
| 7 | `roll` | Degrees, from `/pose` `roll` (radians converted to degrees). |

## How to run it

### From the UI

The VICARIUS UI serves on port 5090. Start it with the `vicarius` command, which sweeps stale processes on port 5090, starts the Flask app detached, waits up to 15 seconds for the port to come up, and opens `http://127.0.0.1:5090/` (the root arms the loading splash and lands on `/desktop`):

```
vicarius
```

The `vicarius` launcher is a symlink at `~/.local/bin/vicarius`. When the command is not on the PATH, install it once with `bash /mnt/rip/vicarius_drive/vicarius_ui_os/install.sh`, then run `vicarius`. The first launch creates the bundled virtual environment of the UI and installs the UI requirements on its own; this is the interpreter the UI server runs under, which the "Environment for a UI run" paragraph below distinguishes from the interpreter that runs the extractor. To force a fresh server, run `/mnt/rip/vicarius_drive/vicarius_ui_os/restart_vicarius.sh`. To check the server is up, run `curl -sf http://127.0.0.1:5090/desktop && echo UP`. The command above is enough to bring the UI up for this module; `/mnt/rip/vicarius_drive/vicarius/_DOCS/START_HERE.md` covers the rest of the platform bootstrap.

Navigate to the module. The desktop loads at `http://127.0.0.1:5090/desktop`. Open the module browser at `http://127.0.0.1:5090/modules`, find the card labeled "Bag Image Export" (the module name is `bag_metashape_export`), and click it. The card opens the module homepage at `http://127.0.0.1:5090/modules/bag_metashape_export`. You can also open that homepage URL directly.

Fill the form and run. On the homepage, enter a Purpose (the runner requires it), fill the `bag_file` field with the path to the `.bag`, and set `output_dir` if you want a base directory other than the default. The module takes no interactive prompts, so the runner drives the script hands-free from the form. Click Run in page for output streamed into the browser, or Run in terminal for a detached gnome-terminal window that survives a browser reload. The extractor runs straight through to completion; it has no pause or resume step. A run takes roughly 1 to 5 minutes, depending on how many frames the bag holds.

Environment for a UI run. The runner launches the extractor as `python3 extract_georeferenced_images.py <bag_file> <output_dir>` (see `_build_argv` in the UI runner). The runner does not activate a per-module virtual environment, does not install `requirements.txt` for you, and the module homepage has no field to select an interpreter. Run in page uses the `python3` inherited by the UI server process. That interpreter is the one on the PATH of the shell that started the VICARIUS UI. It differs from the private virtual environment the UI itself runs under. Run in terminal uses the `python3` a login shell resolves. The extractor imports `rosbags`, `cv2` (opencv-python), `scipy`, `pandas`, `numpy`, `matplotlib`, and `yaml`. When the interpreter that runs the job lacks any one of them, the script stops with a `ModuleNotFoundError` and writes no output. Before the first UI run, confirm the interpreter can load the dependencies. In the same shell you use to launch the UI, run:

```
python3 -c "import rosbags, cv2, scipy, pandas, numpy, matplotlib, yaml; print('deps OK')"
```

A printed `deps OK` means a UI run imports cleanly. A `ModuleNotFoundError` means that interpreter cannot run the extractor. To fix it, install the dependencies into that interpreter with `python3 -m pip install -r requirements.txt` from the `github_repo` directory and start the UI from that same shell, or use the CLI route below, where you create and activate a dedicated Python 3.10 or newer virtual environment and install `requirements.txt` into it. The module ships no pre-built virtual environment. The CLI route works because you build the environment yourself and install `requirements.txt` into it, so that interpreter holds every dependency the extractor imports. The Resume from cold section gives the exact commands.

### From the CLI

The script takes two positional arguments and no flags:

```
python3 extract_georeferenced_images.py <bag_file> [output_dir]
```

Run the script with a Python 3.10 or newer interpreter that has `requirements.txt` installed. The module ships no pre-built virtual environment; the Resume from cold section below builds and activates a dedicated `.venv` inside `github_repo` for this in four commands.

Example against the bundled test bag, run from the module `github_repo` directory:

```
python3 extract_georeferenced_images.py tests/DocNsur_1_UTC_2026-01-07_16-03-26.bag /path/to/output
# writes /path/to/output/DocNsur_1_UTC_2026-01-07_16-03-26/
#   down_images/, forward_images/, down_reference.csv, forward_reference.csv, mission_map.png
```

The entry script is `extract_georeferenced_images.py` at the `github_repo` root (this is the `cli.main_script` in `module.yaml`). It accepts no `--input`, `--output`, or `--purpose` flags, and there are no presets or `analysis_params.yaml`.

## How it works inside

One script does the whole job: `extract_georeferenced_images.py`. There are no step modules. The order of operations in `main()`:

1. `parse_args` reads the positional `bag_file` and the optional `output_dir` (default `.`).
2. `main` checks that the bag exists, sets `bag_name` to the bag stem, and creates `output_base = output_dir / bag_name`.
3. If the VICARIUS logging module imports, `main` calls `process_start` on the event log. Logging is best-effort: if the import or call fails, the script prints a warning and continues.
4. `main` opens the bag with `rosbags.rosbag1.Reader`. `setup_typestore` builds a `ROS1_NOETIC` typestore and registers the embedded `msgdef` of each connection, which is how custom types such as `rangerbot_msgs/pose` deserialize without an external package.
5. `extract_poses` reads every `/pose` message into a DataFrame (`timestamp_ns`, `x`, `y`, `depth`, `altitude_dvl`, `heading`, `pitch`, `roll`). If the frame is empty, `main` prints an error and exits with status 1.
6. `main` computes the mission statistics: duration, longitude and latitude range, a metric span (degrees times 111000 m per degree, longitude scaled by `cos(latitude)`), DVL altitude min, max, and mean, pose sample count, and pose rate.
7. For each camera in `CAMERA_CONFIG` (`down`, then `forward`): `extract_and_save_images` reads the camera topic message by message, decodes each `sensor_msgs/Image` to BGR by its `encoding` (`mono8`, `bgr8`, `rgb8`, `bgra8`, or a generic three-channel fallback), and writes `{prefix}_{idx:04d}.jpg` at JPEG quality 95. It records the image header timestamp, the bag receive timestamp, and the index for each frame. `interpolate_poses_to_images` then linearly interpolates the six pose channels to the bag receive timestamp of each frame and converts heading, pitch, and roll from radians to degrees. `export_metashape_csv` writes the seven-column CSV.
8. `create_mission_map` renders the two-panel PNG from the pose track and the statistics.
9. `main` prints an extraction summary and the Metashape import settings, then calls `process_end` on the event log (best-effort).

External software and versions come from `requirements.txt`: `rosbags>=0.9.0`, `numpy>=1.24.0`, `pandas>=2.0.0`, `scipy>=1.10.0`, `opencv-python>=4.8.0`, `matplotlib>=3.7.0`, `pyyaml>=6.0`. The runtime is Python 3.10 or newer, tested on Ubuntu 22.04 and 24.04. No GPU is required.

## Gotchas and troubleshooting

- The script interpolates pose at the bag receive time of each frame rather than at the image capture time. It stores the image header timestamp in the field `timestamp_ns` but interpolates against `bag_timestamp_ns`, the time the recorder received the message. When the receive time of the recorder lags the capture time of the sensor, every CSV row inherits that offset. Version 1.0.0 does not correct this and applies no clock-offset alignment between the pose and image streams. Verify a known position on the mission map before trusting the georeference.
- The extractor assumes `/pose` `x` is longitude and `y` is latitude, following the field comments in the script. If the upstream convention is swapped, the CSVs stay self-consistent but plot in the wrong place. Check against a known fix.
- ROS1 bags only. The script reads with `rosbags.rosbag1.Reader` and does not handle `.mcap` or ROS2 bags.
- `CAMERA_CONFIG` hard-codes exactly two cameras. Adding or renaming a stream requires a code edit at the top of `extract_georeferenced_images.py`; there is no runtime topic-remap flag.
- A missing or empty `/pose` topic aborts the run with exit status 1 and the message "No pose data found. Cannot continue." A missing camera topic is not fatal: the script prints a warning, skips that camera, and continues.
- A bag path that does not exist exits with status 1 and "Bag file not found" before any work starts.
- The script hard-codes JPEG quality (95), the per-encoding decode path, and image dimensions.
- The extractor needs `rosbags`, `cv2` (opencv-python), and `scipy`, plus `pandas`, `numpy`, `matplotlib`, and `yaml`. These are not present in every VICARIUS interpreter. The runner launches the module with a bare `python3` and does not activate a per-module virtual environment or install `requirements.txt`, so a UI run stops with a `ModuleNotFoundError` and writes no output when the launching interpreter lacks a dependency. The From the UI section gives the one-line import check and the fix. The CLI route avoids this failure because you build and activate a dedicated Python 3.10 or newer virtual environment yourself and install `requirements.txt` into it; the module ships no pre-built virtual environment, so no interpreter holds the dependencies until you create that environment. The Resume from cold section gives the exact venv commands.
- The generic run-folder scaffolding under `src/` (`init_run.py`, `shelve_run.py`) is not part of this extractor. The script never imports or calls either file. The VICARIUS runner creates and shelves run folders on its own.

## Resume from cold

This section assumes the VICARIUS platform is already running. To bring up the UI itself, follow `/mnt/rip/vicarius_drive/vicarius/_DOCS/START_HERE.md`.

Environment. The module runs on Python 3.10 or newer with no GPU, and it ships no pre-built virtual environment. Create and activate a dedicated virtual environment, then install the dependencies from the `requirements.txt` of the module. Run these from the `github_repo` directory:

```
cd /mnt/rip/vicarius_drive/vicarius/modules/bag_metashape_export/github_repo
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

That builds a dedicated interpreter at `.venv/bin/python` inside `github_repo` and brings in `rosbags`, `numpy`, `pandas`, `scipy`, `opencv-python`, `matplotlib`, and `pyyaml`. The `.venv` name is already listed in `.gitignore`, so the environment stays out of version control. Activate this environment with `source .venv/bin/activate` in any shell before you run the extractor from the CLI. The VICARIUS runner does not pin or activate this interpreter. It launches the module with a bare `python3` (see `_build_argv` in the UI runner), inherits that interpreter from the shell that started the UI, and does not install `requirements.txt` for you. `module.yaml` declares only `python_version: ">=3.10"` and `dependencies: requirements.txt`, and the module page offers no interpreter selector. Any Python 3.10 or newer environment with `requirements.txt` installed can run the extractor from the CLI. For a UI run, the dependencies must be importable by the interpreter on the PATH the UI server process inherited. Confirm any interpreter before you rely on it:

```
python3 -c "import rosbags, cv2, scipy, pandas, numpy, matplotlib, yaml; print('deps OK')"
```

A `ModuleNotFoundError` here means that interpreter cannot run the extractor. The optional event logging looks for the VICARIUS logging module under `$VICARIUS_ROOT/_logging/src` (`VICARIUS_ROOT` defaults to `/mnt/vicarius_drive/vicarius`). Logging is best-effort and is not required for extraction; if the module is not found the script runs without it.

Inputs. The input is one ROS1 `.bag`, supplied by path at run time, normally on the NAS. A smoke-test bag ships with the module at `tests/DocNsur_1_UTC_2026-01-07_16-03-26.bag`.

Outputs. Under the VICARIUS runner, the output lands at `inprocess/{run_name}/outputs/{bag_name}/`, where `{run_name}` is the run folder the launcher mints and `{bag_name}` is the bag filename without `.bag`. That folder contains `down_images/`, `forward_images/`, `down_reference.csv`, `forward_reference.csv`, and `mission_map.png`. Run from the CLI directly, the folder is `<output_dir>/<bag_name>/`.

Run it. From the UI, open `http://127.0.0.1:5090/modules/bag_metashape_export` (reach it from the module browser at `http://127.0.0.1:5090/modules`, card "Bag Image Export"), confirm the launching interpreter passes the import check above, enter a Purpose and the bag path, and start the run. From the CLI, with the `.venv` from the Environment step active, run from the module `github_repo` directory:

```
source .venv/bin/activate   # if the environment is not already active
python3 extract_georeferenced_images.py tests/DocNsur_1_UTC_2026-01-07_16-03-26.bag ./out
```

Verify success. The script exits with status 0 and prints "EXTRACTION COMPLETE". Confirm the output folder holds `down_images/` and `forward_images/` populated with `down_0000.jpg`, `forward_0000.jpg`, and so on, both `*_reference.csv` files, and `mission_map.png`. Each reference CSV has one header row plus one data row per frame across seven columns, so the number of data rows in `down_reference.csv` equals the number of files in `down_images/`. Open `mission_map.png` and confirm the plotted track matches the expected dive area.

Metashape import. After extraction, in Metashape: add photos from `down_images/` or `forward_images/`; in the Reference pane, import the matching `down_reference.csv` or `forward_reference.csv`; in the import dialog set Coordinate System to `WGS 84 (EPSG:4326)`, Columns to Label=1, Longitude=2, Latitude=3, Altitude=4, Yaw=5, Pitch=6, Roll=7, Start row to 2, and Rotation format to `Yaw, Pitch, Roll (degrees)`. These values are also recorded in `module.yaml` under `metashape_import`.

## Provenance and links

- Repo: `vicarius/modules/bag_metashape_export/github_repo`
- Entry script: `extract_georeferenced_images.py`
- Related studies: S1_historical, S2_3D_structure, S3_colony_tracking
- Survey chain: `bag_metashape_export -> 3D_phase1 -> 3D_phase2 -> reef_point_seg`
- Data-dictionary descriptors:
  - [bag_metashape_export_down_images](../../../_METADATA/dictionary/datasets/bag_metashape_export_down_images.yaml)
  - [bag_metashape_export_forward_images](../../../_METADATA/dictionary/datasets/bag_metashape_export_forward_images.yaml)
  - [bag_metashape_export_down_reference](../../../_METADATA/dictionary/datasets/bag_metashape_export_down_reference.yaml)
  - [bag_metashape_export_forward_reference](../../../_METADATA/dictionary/datasets/bag_metashape_export_forward_reference.yaml)
  - [bag_metashape_export_mission_map](../../../_METADATA/dictionary/datasets/bag_metashape_export_mission_map.yaml)
- Platform bootstrap: `/mnt/rip/vicarius_drive/vicarius/_DOCS/START_HERE.md`

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-02-04 | Initial VICARIUS integration. |
