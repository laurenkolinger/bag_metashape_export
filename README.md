# ROS Bag to Metashape Exporter

Extract georeferenced images from ROS bag files for Agisoft Metashape photogrammetry.

## Usage

```bash
python extract_georeferenced_images.py <bag_file> [output_dir]
```

**Example:**
```bash
python extract_georeferenced_images.py /data/mission.bag /output
```

Output creates a folder named after the bag file containing:
```
mission/
  ├── down_images/         # Down-facing camera images
  ├── down_reference.csv   # Georeferenced positions for down camera
  ├── forward_images/      # Forward-facing camera images
  ├── forward_reference.csv# Georeferenced positions for forward camera
  └── mission_map.png      # Path visualization with statistics
```

## Requirements

```bash
pip install rosbags pandas opencv-python numpy scipy matplotlib
```

## Metashape Import

1. Add photos from `down_images/` or `forward_images/`
2. Reference pane → Import → select corresponding `_reference.csv`
3. Settings:
   - Coordinate System: **WGS 84 (EPSG:4326)**
   - Columns: Label=1, Longitude=2, Latitude=3, Altitude=4, Yaw=5, Pitch=6, Roll=7
   - Start row: 2
   - Rotation: Yaw, Pitch, Roll (degrees)

## Expected Bag Topics

| Topic | Type | Description |
|-------|------|-------------|
| `/pose` | `rangerbot_msgs/pose` | Position (x=lon, y=lat), depth, heading, pitch, roll, altitudeUsed |
| `/science/image_raw` | `sensor_msgs/Image` | Down-facing science camera (4K) |
| `/zed2/zed_node/left/image_rect_color` | `sensor_msgs/Image` | Forward-facing ZED2 camera |

### Other Topics (may vary)

| Topic | Description |
|-------|-------------|
| `/pressure` | Pressure sensor depth (true depth below surface) |
| `/dvl_dr` | DVL dead reckoning position |
| `/dvl_fix` | DVL velocity and altitude |
| `/stereo_down/left/image_mono` | Stereo down camera (mono) |
| `/mavros/imu/data` | IMU data |
| `/tf` | Transform tree |

## Configuration

Edit `CAMERA_CONFIG` in the script to change camera topics:

```python
CAMERA_CONFIG = {
    "down": {
        "topic": "/science/image_raw",
        "compressed": False,
    },
    "forward": {
        "topic": "/zed2/zed_node/left/image_rect_color",
        "compressed": False,
    },
}
```

Set `compressed: True` for `sensor_msgs/CompressedImage` topics.
