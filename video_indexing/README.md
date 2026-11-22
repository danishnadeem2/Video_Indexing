This project indexes videos stored in Parquet files using the Ultralytics
YOLO11l object-detection model. It samples approximately one frame per second,
writes per-video JSONL records, and can back up completed outputs to Google
Drive using rclone.

1. Validated Kaggle configuration
Component	Value
Dataset	HuggingFaceFV/finevideo
Parquet files	50
Dataset size	Approximately 23.12 GB
Model	Official pretrained YOLO11l (yolo11l.pt)
Image size	640
Confidence	0.40
IoU threshold	0.70
Sampling	1 frame per second
Inference batch size	16
Target classes	0, 2, 16, 63, 67
Tested GPU	NVIDIA Tesla T4
Class IDs: 0 person, 2 car, 16 dog, 63 laptop, 67 cell phone.

The validated Kaggle environment reported PyAV 19.0.2, OpenCV 4.14.0,
PyArrow 23.0.1, Ultralytics 8.4.216, and CUDA available. The OpenCV runtime
version is not guaranteed to equal its PyPI distribution version.

2. Create a Python environment in VS Code
Open the project folder in VS Code and use its integrated terminal.

Windows PowerShell
py -3.11 -m venv .venv
.venv\\Scripts\\Activate.ps1
python -m pip install --upgrade pip
If activation is blocked in PowerShell:

Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.venv\\Scripts\\Activate.ps1
Linux
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
3. Install PyTorch for your computer
Install PyTorch separately because the correct package depends on whether
your computer has an NVIDIA GPU and which CUDA version it supports. Use the
official selector and run the command it generates:

https://pytorch.org/get-started/locally/

For CPU-only execution, choose the CPU build. In the pipeline configuration,
set DEVICE = "cpu" and HALF_PRECISION = False.

Then install the remaining project dependencies:

python -m pip install -r requirements.txt
4. System packages (optional)
PyAV normally installs from a prebuilt Python wheel. If your Linux system
needs FFmpeg tools or libraries, install:

sudo apt-get update
sudo apt-get install -y ffmpeg
For Google Drive backup on Ubuntu/Debian, install rclone:

sudo apt-get update
sudo apt-get install -y rclone
On Windows, install rclone from https://rclone.org/downloads/ and verify:

rclone version
5. Download the FineVideo dataset
Install or update the Hugging Face CLI:

python -m pip install --upgrade huggingface_hub
From the project root, download the same five shard groups used in the Kaggle
run:

hf download HuggingFaceFV/finevideo --repo-type dataset --include "data/train-000[0-4]*" --local-dir .
The expected Parquet directory is data/. Verify the download:

from pathlib import Path

DATA_DIR = Path("data")
ORDERED_PARQUET_FILES = sorted(DATA_DIR.rglob("*.parquet"))

print("Parquet files:", len(ORDERED_PARQUET_FILES))
print("Size (GB):", round(
    sum(p.stat().st_size for p in ORDERED_PARQUET_FILES) / (1024 ** 3), 2
))

if len(ORDERED_PARQUET_FILES) != 50:
    raise RuntimeError(f"Expected 50 Parquet files, found {len(ORDERED_PARQUET_FILES)}")
If the files are downloaded into a different directory, update DATA_DIR
in the processing script. The Kaggle-specific /tmp/finevideo_workspace
path will not automatically exist on your PC.

6. Prepare YOLO11l weights
The Kaggle run used the official pretrained YOLO11l weights, not a custom
fine-tuned checkpoint. Ultralytics can download the official weights:

from ultralytics import YOLO
model = YOLO("yolo11l.pt")
For a stable local path, place the weights in a models/ directory and
update MODEL_PATH in your script.

7. Prepare output directories
Example local configuration:

from pathlib import Path

WORK_DIR = Path("workspace")
OUTPUT_DIR = WORK_DIR / "finevideo_results"
PER_VIDEO_DIR = OUTPUT_DIR / "per_video"
CHECKPOINT_DIR = OUTPUT_DIR / "checkpoints"
LOG_DIR = OUTPUT_DIR / "logs"

for folder in (OUTPUT_DIR, PER_VIDEO_DIR, CHECKPOINT_DIR, LOG_DIR):
    folder.mkdir(parents=True, exist_ok=True)
Update all Kaggle-specific paths in the processing script to local paths.

8. Configure Google Drive backup (optional)
Install rclone and create a personal Google Drive remote named gdrive:

rclone config
Follow the interactive prompts to authorize your own Google account. Do not
put OAuth client secrets, access tokens, or refresh tokens in source code or
commit them to Git.

Test the remote:

rclone lsd gdrive:
Create the backup folders:

rclone mkdir gdrive:FineVideo_Backup/per_video
rclone mkdir gdrive:FineVideo_Backup/checkpoints
rclone mkdir gdrive:FineVideo_Backup/logs
The processing script expects gdrive:FineVideo_Backup. If you do not want
Drive backup, disable or remove the script's rclone upload/download calls.

9. Processing configuration
The Kaggle-tested values were:

IMG_SIZE = 640
CONFIDENCE = 0.40
IOU_THRESHOLD = 0.70
TARGET_CLASSES = [0, 2, 16, 63, 67]
DEVICE = 0
HALF_PRECISION = True
INFERENCE_BATCH_SIZE = 16

TARGET_FPS = 1
MAX_NEW_VIDEOS = 1       # first validation run
UPLOAD_RETRIES = 3
UPLOAD_TIMEOUT = 900
DRIVE_BACKUP = "gdrive:FineVideo_Backup"
For CPU-only execution:

DEVICE = "cpu"
HALF_PRECISION = False
Start with MAX_NEW_VIDEOS = 1. Confirm the JSONL output is created and,
if Drive backup is enabled, confirm both the JSONL file and .done marker
are present remotely. Then set MAX_NEW_VIDEOS = None for an unlimited run.

The script's resume logic should skip a video only when its JSONL file and
completion marker both exist in the configured backup.

10. Run order
Activate .venv.
Install the correct PyTorch build for your hardware.
Install requirements.txt.
Download and verify the 50 Parquet files.
Load YOLO11l.
Update dataset, model, and output paths for your computer.
Configure and test rclone if using Google Drive backup.
Run one video and verify its output and backup.
Set MAX_NEW_VIDEOS = None and run the remaining dataset.
Run the project's notebook in VS Code, or run its Python entry-point if
the processing code has been saved as a .py file.

11. Expected outputs
The processing code creates per-video JSONL records and completion markers.
The current implementation stores them under:

finevideo_results/
  per_video/
    <parquet-filename>/
      <row-index>.jsonl
      <row-index>.done
  checkpoints/
  logs/
The final merge step may create finevideo_detections.json or
finevideo_detections.jsonl, depending on the version of the processing
script you use.

12. Important notes
Kaggle /tmp is temporary. Local PC storage persists independently.
Google Drive protects only files that were successfully uploaded.
A completion marker should be uploaded only after its matching JSONL file.
An interrupted video may need to be processed again.
Sampling one frame per second reduces YOLO inference calls, but the decoder
may still decode intervening frames.
Processing time depends on video duration, disk speed, CPU/GPU, and network.
Ensure adequate disk space for the dataset, model, temporary files, and
generated results.
Never commit OAuth credentials, refresh tokens, Kaggle secrets, or other
private credentials to Git.




