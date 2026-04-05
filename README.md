<div align="center">

# Wildlife Detection using YOLOv8 with AWS Cloud Pipeline

### Real-Time Camera Trap Image Classification & Geospatial Analytics

**Vivek Vasisht Ediga**

</div>

---

## Overview

An end-to-end cloud-based wildlife detection pipeline that simulates camera trap image ingestion, classifies wildlife species using a **YOLOv8** object detection model deployed on **AWS SageMaker**, and visualizes detection results on geospatial maps.

The system processes images of African elephants (*Loxodonta* species) from the [Spatiotemporal Wildlife Dataset](https://www.kaggle.com/datasets/travisdaws/spatiotemporal-wildlife-dataset) on Kaggle, enriching each detection with geographic coordinates, temperature, and elevation metadata for downstream analytics.

## Architecture

```
Camera Trap Images (Kaggle Dataset)
        |
        v
  [ main.py - Simulated Streaming ]
        |  (exponential inter-arrival, retry logic)
        v
  [ Amazon S3 ] ──> [ Lambda: IngestionLogger ] ──> [ DynamoDB ]
                                                          |
                                                          v
                                              [ Lambda: ImageEventClassifier ]
                                                          |
                                                          v
                                              [ SageMaker Endpoint (YOLOv8s) ]
                                                          |
                                                          v
                                              [ DynamoDB (predictions stored) ]
                                                          |
                              ┌────────────────────────────┼─────────────────────┐
                              v                            v                     v
                   [ Lambda: BatchNotifier ]    [ Lambda: CreateGeoJSON ]   [ Power BI ]
                        (SNS Email)             (GeoJSON + Flattened JSON)   (Dashboard)
```

## Key Features

- **YOLOv8s Object Detection** — Trained model (`best_yolo.pt`) detects wildlife species with confidence thresholding (>0.25)
- **Simulated Camera Trap Streaming** — Realistic upload simulation with exponential inter-arrival times (~3s avg) and 30% network failure rate with retries
- **Serverless Processing Pipeline** — 4 Lambda functions handle ingestion logging, classification, batch notifications, and GeoJSON generation
- **Real-Time Inference** — SageMaker endpoint serves YOLOv8 predictions on 640x640 images via base64-encoded payloads
- **Geospatial Analytics** — GeoJSON output with lat/long, temperature, and elevation for web map visualization
- **Email Notifications** — SNS-based alerts for image uploads and classification summaries every 5 minutes
- **Power BI Dashboard** — Interactive map visualization of detection results (`Camera_Trap_Map.pbix`)

## Tech Stack

| Layer | Technologies |
|---|---|
| **ML / CV** | YOLOv8 (Ultralytics), PyTorch, OpenCV, Pillow |
| **Cloud** | AWS S3, DynamoDB, Lambda, SageMaker, EventBridge, SNS, IAM |
| **Language** | Python 3.12 |
| **Visualization** | Power BI, GeoJSON |
| **Utilities** | boto3, PyYAML, pandas, tqdm |

## Project Structure

```
WildlifeDetection_YOLOv8/
├── main.py                             # Entry point - runs the simulation pipeline
├── config.yaml                         # AWS and camera trap configuration
├── requirements.txt                    # Python dependencies
├── Camera_Trap_Map.pbix                # Power BI dashboard
│
├── src/
│   ├── simulate_image_streaming.py     # Camera trap upload simulation with retry logic
│   ├── s3_loader.py                    # S3 image loading utilities
│   └── s3_streamer.py                  # S3 image streaming and decoding
│
├── Model/
│   ├── best_yolo.pt                    # Trained YOLOv8s model weights
│   ├── model.tar.gz                    # Packaged model for SageMaker deployment
│   └── inference.py                    # SageMaker inference handler
│
├── stage2_yolov8/
│   ├── deploy_endpoint.py              # SageMaker endpoint deployment script
│   ├── run_realtime_inference.py       # Batch inference on S3 images
│   ├── create_images_csv.py            # Generate CSV of S3 image URIs
│   └── local_inference_test.py         # Local model testing
│
├── lambdas/scripts/
│   ├── ingestion_logger.py             # Stage 1: Log image metadata to DynamoDB
│   ├── image_event_classifier.py       # Stage 2: Invoke SageMaker for predictions
│   ├── batch_notifier.py              # Stage 3: Send SNS email summaries
│   └── create_geojson.py              # Stage 4: Generate GeoJSON from results
│
├── utils/
│   ├── download_dataset.py             # Download Kaggle wildlife dataset
│   ├── provision_resources.py          # Provision all AWS resources
│   ├── clean_up.py                     # Tear down AWS resources
│   └── read_yaml.py                    # YAML config utilities
│
└── files/
    └── WildlifeDetection_YOLOv8.pdf    # Project report
```

## Setup & Usage

### Prerequisites

- Python 3.12+
- AWS account with appropriate permissions
- [Kaggle API token](https://www.kaggle.com/docs/api) (`kaggle.json`)
- Power BI Desktop (optional, for dashboard viewing)

### 1. Install Dependencies

```bash
pip install -r requirements.txt
pip install kaggle
```

### 2. Download the Dataset

Place your Kaggle API token in `~/.kaggle/kaggle.json`, then run:

```bash
python utils/download_dataset.py
```

This downloads the [Spatiotemporal Wildlife Dataset](https://www.kaggle.com/datasets/travisdaws/spatiotemporal-wildlife-dataset) into a `data/` directory. By default, the pipeline uses the African Forest Elephant (*loxodonta cyclotis*) subset for testing. To use the larger African Bush Elephant (*loxodonta africana*) set, update the `root_dir` path in `config.yaml`.

### 3. Configure AWS Credentials

Create an `aws_auth.yaml` file in the project root:

```yaml
aws:
  access_key_id: YOUR_ACCESS_KEY_ID
  secret_access_key: "YOUR_SECRET_ACCESS_KEY"
  region: "us-east-1"
```

### 4. Update config.yaml

Set your AWS username and notification email:

```yaml
USER_INFO:
  user_name: 'your_aws_username'
  region: 'us-east-1'
  email: 'your_email@example.com'
```

### 5. Provision AWS Resources

```bash
python utils/provision_resources.py
```

This creates the S3 bucket, DynamoDB table, Lambda functions, IAM roles, and SNS topic.

### 6. Prepare the AWS Console

- Clear any leftover images from the `from-camera-trap-1` S3 bucket
- Delete all **items** (not the table) from the `image_event` DynamoDB table
- In **Amazon EventBridge > Rules**, enable: `BatchNotifierRule`, `IngestionLoggerRule`, `CreateGeoJSON`

### 7. Deploy the SageMaker Endpoint

In the AWS Console, navigate to **SageMaker AI > Deployments & Inference > Endpoints**:
1. Click **Create Endpoint**
2. Name: `yolov8s`
3. Endpoint Configuration: `yolov8-prod-config`
4. Wait for the endpoint status to show *InService*

### 8. Run the Pipeline

```bash
python main.py
```

Accept the SNS subscription confirmation email to receive notifications.

### 9. View Results

| Output | Location |
|---|---|
| Uploaded images + metadata | S3: `from-camera-trap-1` |
| Detection results & metadata | DynamoDB: `image_event` table |
| Email notifications | Your configured email via SNS |
| Power BI dashboard data | S3: `wildlife_predictions_FLATTENED.json` |
| Web map data | S3: `wildlife_predictions.geojson` |

Open `Camera_Trap_Map.pbix` in Power BI Desktop to view the interactive detection map.

### 10. Clean Up

> **Important:** The SageMaker endpoint is a provisioned resource that incurs costs while running.

1. Go to **SageMaker AI > Endpoints** and **delete** the `yolov8s` endpoint
2. Leave endpoint configurations and deployable models untouched

To tear down all other AWS resources:

```bash
python utils/clean_up.py
```

## Dataset

[Spatiotemporal Wildlife Dataset](https://www.kaggle.com/datasets/travisdaws/spatiotemporal-wildlife-dataset) from Kaggle.

Each image includes metadata: latitude, longitude, positional accuracy, temperature, elevation, and timestamp.

## License

This project was developed as a personal work.
