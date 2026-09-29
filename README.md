# Bag Metashape Export (bag_metashape_export)

Bag Metashape Export, display name VECTOR, extracts the down-facing and forward-facing camera streams from one ROS1 `.bag`, joins each frame to an interpolated vehicle pose, and writes Agisoft Metashape import files for the operator who is starting a 3D reconstruction from an AUV dive.

- Tags: AUV, 3D | Version: 1.0.0 | Type: extract | Status: active | Owner: Lauren K Olinger
- Repo: `vicarius/modules/bag_metashape_export/github_repo` | Related studies: S1_historical, S2_3D_structure, S3_colony_tracking

## What it does and why it exists

Agisoft Metashape builds a georeferenced 3D reconstruction from two things: a folder of images and a reference table that gives a position and an attitude for each image. A rangerbot-class AUV does not record either of those directly. It records a ROS1 `.bag`, a single file that interleaves raw camera messages and navigation messages on their own clocks. Someone has to decode the camera messages into JPEGs and align every frame to the vehicle track before Metashape can use them. This module does that decoding and alignment.

The module reads one `.bag`, extracts the down-facing 4K science camera and the forward-facing ZED2 left camera to JPEG, linearly interpolates the `/pose` navigation track to each frame, and writes one Metashape-format georeference CSV per camera. It also renders a mission-map PNG so the operator can review the track and the sample counts before committing to a reconstruction.

The operator runs this module once per bag, right after a dive and before opening Metashape. It is the first step in the 3D reconstruction workflow. The module extracts frames and reference tables; it does not run photogrammetry and it does not build a model.

The extractor is the entry point from bag to frames. It exists so that the reconstruction workflow can start from a standard Metashape image folder plus reference CSV instead of a raw ROS bag.

## Where it sits in the pipeline

The upstream input is a ROS1 `.bag` produced by the AUV during a dive. The vehicle recorder writes the bag; no VICARIUS module produces it. The operator supplies the bag path at run time.

The immediate downstream consumer is Agisoft Metashape, run by hand outside VICARIUS. The operator adds the extracted images as photos, imports the matching reference CSV, and builds the reconstruction. This module produces the Metashape inputs; the Metashape reconstruction itself is a manual step and is not part of this module.

This module stands apart from the platform's 3D survey chain, `3D_phase_1 -> manual_edit -> 3D_phase2 -> tagfab` (VOYAGER 1, Start manual edit, VOYAGER 2, then VIGIL). VOYAGER 1 builds its models from TCRMP transect video listed in the TCRMP 3D registry, and no VICARIUS module reads this module's `down_images/`, `forward_images/` or reference CSVs; the Metashape project built from them by hand is the end of this module's line.

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
| `output_dir` | string | `.` | Base output directory. The script creates a subfolder named after the bag file inside this directory and writes every artifact under that subfolder. The module homepage does not create a run folder or fill this in on the operator's behalf: whatever the operator types into the form's Output directory field is passed through unchanged. Left empty, the script's own default (`.`) resolves to the runner's working directory, the module's `github_repo` clone, not a useful output location. |

The module declares no other parameters. There is no `cli.flag_map` in `module.yaml`, so from the CLI the only runtime settings are the two positional arguments (`bag_file` and `output_dir`). Camera topics, JPEG quality, and interpolation method are fixed in the script and are not exposed as parameters. The `output_dir` entry above is a duplicate of the module homepage's built-in Output directory field; see the launch-from-UI warning in Gotchas and troubleshooting before running this module from the UI.

## Outputs and data-dictionary entries

The script creates `<output_dir>/<bag_name>/` and writes everything under it, where `<bag_name>` is the bag filename without its `.bag` extension. `<output_dir>` is exactly what the operator supplies: the CLI's second positional argument, or the runner form's Output directory field passed through unchanged (see Parameters above). The `path_template` entries in `module.yaml` and the table below are written in the platform's general form, `inprocess/{run_name}/outputs/{bag_name}/...`, but this module never runs inside a numbered run folder: `{run_name}` never resolves to a real directory, because a run folder is minted only for a module whose `module.yaml` sets `ui.style: launcher`, and this one declares no `ui:` block. Read `{run_name}/outputs/` in the paths below as a naming convention, not a literal path: the artifact on disk sits directly under whatever `<output_dir>/<bag_name>/` the operator or the `vicarius run` command pointed at.

Every artifact binds to a data-dictionary descriptor, the YAML file in the data dictionary that documents that output. Each descriptor is `status: draft`. `module.yaml` marks `eml: true` on all five outputs, which is meant to make the platform write a metadata sidecar next to each artifact once a run launched through the runner (the UI or `vicarius run`, see How to run it) finishes. For this module that step never actually writes anything: it only fills in a `path_template` built from the placeholders `output_dir` and `input_dir`, and every `path_template` below instead carries `{run_name}` and `{bag_name}`, which that step does not resolve, so the entry is silently skipped on every launch route. This is a limitation in the platform's runner, not in this module; nothing in this module's own files can fix it.

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

The VICARIUS UI serves on port 5090. Start it with the `vicarius` command, which sweeps stale processes on port 5090, starts the Flask app detached, waits up to 15 seconds for the port to come up, and opens `http://127.0.0.1:5090/` (the root arms the loading splash and goes to `/desktop`):

```
vicarius
```

The `vicarius` launcher is a symlink at `~/.local/bin/vicarius`. When the command is not on the PATH, install it once with `bash /mnt/rip/vicarius_drive/vicarius_ui_os/install.sh`, then run `vicarius`. The first launch creates the bundled virtual environment of the UI and installs the UI requirements on its own; this is the interpreter the UI server runs under, which the "Environment for a UI run" paragraph below distinguishes from the interpreter that runs the extractor. To force a fresh server, run `/mnt/rip/vicarius_drive/vicarius_ui_os/restart_vicarius.sh`. To check the server is up, run `curl -sf http://127.0.0.1:5090/desktop && echo UP`. The command above is enough to bring the UI up for this module; `/mnt/rip/vicarius_drive/vicarius/_DOCS/START_HERE.md` covers the rest of the platform bootstrap.

Navigate to the module. The desktop loads at `http://127.0.0.1:5090/desktop`. Open the module browser at `http://127.0.0.1:5090/modules`, find the card labeled "VECTOR" (the module name is `bag_metashape_export`), and click it. The card opens the module homepage at `http://127.0.0.1:5090/modules/bag_metashape_export`. You can also open that homepage URL directly.

Fill the form and run. `module.yaml` declares no `ui:` block for this module, so the homepage shows the generic runner form: an Input directory field (paste the path to the `.bag` file there; the field label and placeholder are the platform defaults, not a `bag_file`-specific prompt) and an Output directory field. Enter a Purpose (the runner requires it), fill the Input directory field with the path to the `.bag`, and fill the Output directory field with an existing directory the operator controls, for example a folder set aside on the NAS for this bag's extraction. The homepage does not create a run folder or fill this field in: left empty, the script's own default (`.`) resolves to the module's `github_repo` clone on disk, not a useful output location. The module takes no interactive prompts, so the runner drives the script hands-free from the form. Before clicking Run (in page) or Run in terminal, clear the second `output_dir` field, the one inside the Module parameters block further down the form, to empty: with that field at its pre-filled default the launch fails before extraction starts, for the reason recorded in Gotchas and troubleshooting, and clearing it is what avoids that failure. The extractor itself runs straight through to completion once launched correctly, from the CLI or from the UI with the Output directory field filled in and that second field cleared; it has no pause or resume step. A run takes roughly 1 to 5 minutes, depending on how many frames the bag holds.

Environment for a UI run. The runner launches the extractor as `python3 extract_georeferenced_images.py <bag_file> <output_dir>` (see `_build_argv` in the UI runner). The runner does not activate a per-module virtual environment, does not install `requirements.txt` for you, and the module homepage has no field to select an interpreter. Run in page uses the `python3` inherited by the UI server process. That interpreter is the one on the PATH of the shell that started the VICARIUS UI. It differs from the private virtual environment the UI itself runs under. Run in terminal uses the `python3` a login shell resolves. The extractor imports `rosbags`, `cv2` (opencv-python), `scipy`, `pandas`, `numpy`, and `matplotlib`. When the interpreter that runs the job lacks any one of them, the script stops with a `ModuleNotFoundError` and writes no output. `requirements.txt` also lists `pyyaml`, but the extractor's own import path never reaches it: `pyyaml` backs only the unused `src/` run-folder scaffolding (`init_run.py`, `shelve_run.py`), and a missing copy does not stop extraction. Before the first UI run, confirm the interpreter can load the dependencies the extractor itself needs. In the same shell you use to launch the UI, run:

```
python3 -c "import rosbags, cv2, scipy, pandas, numpy, matplotlib; print('deps OK')"
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

Getting a run into the platform's audit trail. The command above calls the script directly, so it writes no row to `vicarius/_METADATA/logs/processing_log.csv` and produces no metadata sidecar (see the note under Outputs and data-dictionary entries). To have a real extraction recorded, launch it through the platform's generic run command instead, which shares its launch code with the module homepage rather than calling the script directly:

```
vicarius run bag_metashape_export --input <bag_file> --output <existing_directory> --purpose "<why>"
```

`--output` must point at a directory that already exists. Like a launch from the module homepage, this command does not create a numbered run folder under `inprocess/`: the extractor writes `<output>/<bag_name>/` directly under the directory supplied, exactly as it does from the raw script call. This route does not hit the module homepage's `output_dir` duplicate-flag bug (see Gotchas and troubleshooting): it only turns an explicit `--param key=value` into a launched flag, and called with no `--param` at all, which is the ordinary case here since the module declares nothing worth overriding from the CLI, it adds none. On completion, success or failure, it appends one row to `processing_log.csv`, the same audit trail a UI-launched run produces.

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
- The extractor needs `rosbags`, `cv2` (opencv-python), and `scipy`, plus `pandas`, `numpy`, and `matplotlib`. These are not present in every VICARIUS interpreter. The runner launches the module with a bare `python3` and does not activate a per-module virtual environment or install `requirements.txt`, so a UI run stops with a `ModuleNotFoundError` and writes no output when the launching interpreter lacks a dependency. The From the UI section gives the one-line import check and the fix. The CLI route avoids this failure because you build and activate a dedicated Python 3.10 or newer virtual environment yourself and install `requirements.txt` into it; the module ships no pre-built virtual environment, so no interpreter holds the dependencies until you create that environment. The Resume from cold section gives the exact venv commands.
- The generic run-folder scaffolding under `src/` (`init_run.py`, `shelve_run.py`) is not part of this extractor: the script never imports or calls either file, and this module never runs inside a run folder in the first place, because it declares no `ui:` block and is not `ui.style: launcher` (see Outputs and data-dictionary entries above).
- A launch from the module homepage fails with the form at its defaults. `module.yaml` declares one parameter, `output_dir`, which duplicates the name of the module homepage's built-in Output directory field. The page renders both: the built-in field and, in its own Module parameters block, a second field also named `output_dir`, pre-filled with the parameter's default (`.`). Left at that default, both Run (in page) and Run in terminal collect that second field's value and pass it through unchanged, so the launched command becomes `extract_georeferenced_images.py <bag_file> <output_dir> --output_dir .`. The script's argument parser accepts only the two positional arguments, so it rejects `--output_dir` as unrecognized and exits before extraction starts. Clearing that second `output_dir` field to empty before clicking Run avoids this: the platform drops an empty parameter field from the launched command instead of passing it as a flag, so the command reduces to the two positional arguments the script accepts. The cause is a redundant `parameters:` entry in `module.yaml`. The module runs correctly from the CLI, which never sees the extra flag, and from the UI once that field is cleared. See the CLI instructions above.
- The metadata sidecar VICARIUS is meant to write beside each output at run finalisation (`eml: true` in `module.yaml`) never gets written for this module, on the CLI route or the UI route: see the note under Outputs and data-dictionary entries above. Nothing in this module's own files can fix it; it is a limitation in the platform runner, not in this module.

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
python3 -c "import rosbags, cv2, scipy, pandas, numpy, matplotlib; print('deps OK')"
```

A `ModuleNotFoundError` here means that interpreter cannot run the extractor. The optional event logging looks for the VICARIUS logging module under `$VICARIUS_ROOT/_logging/src`. The script's own fallback, used only when `VICARIUS_ROOT` is unset, is `/mnt/vicarius_drive/vicarius`, which differs from the platform's usual root, `/mnt/rip/vicarius_drive/vicarius` (see `/mnt/rip/vicarius_drive/vicarius/_DOCS/START_HERE.md`). Set `VICARIUS_ROOT` explicitly to the real installation path before relying on this best-effort logging. Logging is best-effort and is not required for extraction; if the module is not found the script runs without it, and extraction proceeds either way.

Inputs. The input is one ROS1 `.bag`, supplied by path at run time, normally on the NAS. A smoke-test bag ships with the module at `tests/DocNsur_1_UTC_2026-01-07_16-03-26.bag`.

Outputs. The output is written to `<output_dir>/<bag_name>/`, where `<output_dir>` is the CLI's second positional argument or the runner form's Output directory field, and `<bag_name>` is the bag filename without `.bag`. This module never runs inside a numbered `inprocess/{run_name}/` run folder (see Where it sits in the pipeline and Outputs and data-dictionary entries above). That folder contains `down_images/`, `forward_images/`, `down_reference.csv`, `forward_reference.csv`, and `mission_map.png`.

Run it. From the CLI, with the `.venv` from the Environment step active, run from the module `github_repo` directory. The module homepage's Run buttons work too, once the second `output_dir` field in Module parameters is cleared to empty (see Gotchas and troubleshooting); the raw script call below writes no row to `processing_log.csv`, so use `vicarius run` (see From the CLI above) instead when the run needs to show up in the platform's audit trail:

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
- Survey chain: outside it; the 3D survey chain is `3D_phase_1 -> manual_edit -> 3D_phase2 -> tagfab`.
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
