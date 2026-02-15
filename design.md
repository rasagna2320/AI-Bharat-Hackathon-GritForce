# CropPulse: System Design Document
## AI-Powered Crop Stress Detection System

---

## 1. Document Overview

### 1.1 Purpose
This document provides the detailed system design for CropPulse, an AI-powered early crop stress and disease alert system for smallholder farmers in India's hilly and remote regions.

### 1.2 Scope
This design covers the MVP implementation for hackathon demonstration, with considerations for future scalability and production deployment.

### 1.3 Audience
- Software Engineers and Developers
- DevOps and Cloud Engineers
- Data Scientists and ML Engineers
- System Architects
- Technical Reviewers

---

## 2. High-Level Architecture

### 2.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Data Sources Layer                        │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Satellite APIs │  Weather APIs   │  Farmer Mobile App/SMS      │
│  (Sentinel-2)   │  (IMD/OpenWX)   │  (Image Upload - Future)    │
└────────┬────────┴────────┬────────┴──────────┬──────────────────┘
         │                 │                    │
         ▼                 ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Ingestion Layer (AWS)                        │
├─────────────────┬─────────────────┬───────────────────────────┬─┤
│  Lambda:        │  Lambda:        │  API Gateway:             │ │
│  Satellite      │  Weather        │  Farmer Input             │ │
│  Data Fetcher   │  Data Fetcher   │  Handler                  │ │
└────────┬────────┴────────┬────────┴──────────┬────────────────┴─┘
         │                 │                    │
         ▼                 ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Storage Layer (AWS S3)                        │
├─────────────────┬─────────────────┬───────────────────────────┬─┤
│  Raw Satellite  │  Weather Data   │  Farmer Images            │ │
│  Imagery        │  (JSON)         │  (Future)                 │ │
└────────┬────────┴────────┬────────┴──────────┬────────────────┴─┘
         │                 │                    │
         └─────────────────┴────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Processing Layer (AWS Lambda)                   │
├─────────────────────────────────────────────────────────────────┤
│  • Data Preprocessing & NDVI Calculation                         │
│  • Feature Engineering (Weather + Vegetation Indices)            │
│  • Data Validation & Quality Checks                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                AI/ML Layer (Amazon SageMaker)                    │
├─────────────────────────────────────────────────────────────────┤
│  • Crop Stress Classification Model (Inference Endpoint)         │
│  • Model Versioning & A/B Testing                                │
│  • Confidence Score Generation                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Decision & Alert Layer (AWS Lambda)                 │
├─────────────────────────────────────────────────────────────────┤
│  • Alert Rule Engine                                             │
│  • Advisory Content Generation                                   │
│  • Priority & Routing Logic                                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│            Delivery Layer (AWS Services)                         │
├─────────────────┬─────────────────┬───────────────────────────┬─┤
│  Amazon Polly   │  SNS/SMS        │  API Gateway              │ │
│  (Voice TTS)    │  Gateway        │  (Mobile App)             │ │
└────────┬────────┴────────┬────────┴──────────┬────────────────┴─┘
         │                 │                    │
         └─────────────────┴────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              Data Persistence (DynamoDB)                         │
├─────────────────────────────────────────────────────────────────┤
│  • User Profiles & Farm Plots                                    │
│  • Alert History & Advisory Records                              │
│  • Feedback & Acknowledgments                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│         Monitoring & Logging (CloudWatch)                        │
├─────────────────────────────────────────────────────────────────┤
│  • System Metrics & Performance Monitoring                       │
│  • Error Tracking & Alerting                                     │
│  • Audit Logs & Compliance                                       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Architecture Principles

- **Serverless-First**: Minimize operational overhead using AWS Lambda and managed services
- **Event-Driven**: Asynchronous processing triggered by data availability and schedules
- **Scalable**: Auto-scaling components to handle variable load
- **Resilient**: Graceful degradation and retry mechanisms for failures
- **Cost-Effective**: Pay-per-use model suitable for pilot and scale
- **Privacy-Preserving**: Data encryption and minimal data collection

### 2.3 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Serverless Architecture | Reduces operational complexity, auto-scales, cost-effective for variable workloads |
| DynamoDB over RDS | Better fit for key-value access patterns, serverless, lower latency |
| SageMaker Endpoints | Managed ML inference with versioning and monitoring |
| S3 for Data Lake | Cost-effective storage for satellite imagery and raw data |
| API Gateway | Managed API layer with authentication, throttling, and monitoring |
| Amazon Polly | Native AWS TTS service supporting multiple Indian languages |

---

## 3. Component-Level Design

### 3.1 Data Ingestion Components

#### 3.1.1 Satellite Data Fetcher (Lambda)

**Purpose**: Fetch satellite imagery from Sentinel-2 or Landsat APIs

**Trigger**: CloudWatch Events (scheduled daily or every 3 days)

**Inputs**:
- Target district coordinates (bounding box)
- Date range for imagery
- Cloud cover threshold (<30%)

**Processing**:
1. Query Sentinel Hub or Google Earth Engine API
2. Filter by cloud cover and date
3. Download relevant bands (Red, NIR for NDVI)
4. Store raw imagery in S3 with metadata

**Outputs**:
- Raw satellite imagery → S3 bucket: `croppulse-satellite-raw/`
- Metadata JSON → S3 bucket: `croppulse-satellite-metadata/`
- Event trigger → Data Processing Pipeline

**Error Handling**:
- Retry with exponential backoff (3 attempts)
- Log failures to CloudWatch
- Send alert to admin if persistent failure

**Configuration**:
```python
{
  "schedule": "cron(0 6 * * ? *)",  # Daily at 6 AM IST
  "timeout": 300,  # 5 minutes
  "memory": 512,  # MB
  "retry_attempts": 3
}
```

#### 3.1.2 Weather Data Fetcher (Lambda)

**Purpose**: Collect localized weather data for target regions

**Trigger**: CloudWatch Events (scheduled every 6 hours)

**Inputs**:
- List of farm plot coordinates
- Weather parameters (temp, rainfall, humidity, wind)

**Processing**:
1. Query weather API (IMD, OpenWeatherMap)
2. Aggregate data by district/block
3. Calculate derived metrics (heat stress index, moisture deficit)
4. Store in S3 and DynamoDB

**Outputs**:
- Weather data → S3: `croppulse-weather-data/`
- Latest weather → DynamoDB table: `WeatherData`

#### 3.1.3 Farmer Input Handler (API Gateway + Lambda)

**Purpose**: Accept farmer-uploaded images and manual observations

**Trigger**: API Gateway POST request

**Endpoints**:
```
POST /api/v1/upload-image
POST /api/v1/report-observation
GET /api/v1/plot-status/{plot_id}
```

**Processing**:
1. Authenticate farmer using JWT token
2. Validate image format and size (<5MB)
3. Generate pre-signed S3 URL for upload
4. Store metadata in DynamoDB
5. Trigger image analysis pipeline (future)

**Security**:
- JWT authentication
- Rate limiting (10 requests/minute per user)
- Image malware scanning
- Input validation and sanitization

### 3.2 Data Processing Components

#### 3.2.1 NDVI Calculator (Lambda)

**Purpose**: Calculate Normalized Difference Vegetation Index from satellite imagery

**Trigger**: S3 event (new satellite image uploaded)

**Processing**:
```python
NDVI = (NIR - Red) / (NIR + Red)
```

**Steps**:
1. Load Red and NIR bands from S3
2. Calculate NDVI pixel-wise
3. Aggregate NDVI by farm plot boundaries
4. Calculate statistics (mean, std, min, max)
5. Detect anomalies (sudden drops)
6. Store processed data

**Outputs**:
- NDVI raster → S3: `croppulse-processed/ndvi/`
- Plot-level NDVI stats → DynamoDB: `PlotNDVI`

#### 3.2.2 Feature Engineering Pipeline (Lambda)

**Purpose**: Prepare ML-ready features combining multiple data sources

**Inputs**:
- NDVI time series (last 30 days)
- Weather data (last 14 days)
- Crop type and growth stage
- Historical stress patterns

**Feature Set**:
1. **Vegetation Features**:
   - Current NDVI value
   - NDVI trend (7-day, 14-day slope)
   - NDVI volatility (standard deviation)
   - NDVI anomaly score (deviation from historical mean)

2. **Weather Features**:
   - Cumulative rainfall (7-day, 14-day)
   - Temperature extremes (max, min)
   - Heat stress days (temp >35°C)
   - Moisture deficit index

3. **Temporal Features**:
   - Days since planting
   - Crop growth stage (encoded)
   - Season indicator

4. **Spatial Features**:
   - Elevation
   - Slope
   - Neighboring plot health (average NDVI)

**Output**:
- Feature vector → S3: `croppulse-features/`
- Ready for ML inference

### 3.3 AI/ML Components

#### 3.3.1 Crop Stress Classification Model

**Model Architecture**:
```
Input Layer (Feature Vector: 20 dimensions)
    ↓
Dense Layer (128 units, ReLU, Dropout 0.3)
    ↓
Dense Layer (64 units, ReLU, Dropout 0.2)
    ↓
Dense Layer (32 units, ReLU)
    ↓
Output Layer (5 classes, Softmax)
    ↓
[Healthy, Early Stress, Moderate Stress, Severe Stress, Disease Risk]
```

**Training Approach**:
- Transfer learning from pre-trained models (optional for image inputs)
- Supervised learning with labeled historical data
- Class imbalance handling (SMOTE, class weights)
- Cross-validation across seasons and regions

**Model Outputs**:
```json
{
  "plot_id": "MZ-AZ-001-P123",
  "prediction": "Early Stress",
  "confidence": 0.82,
  "class_probabilities": {
    "Healthy": 0.05,
    "Early Stress": 0.82,
    "Moderate Stress": 0.10,
    "Severe Stress": 0.02,
    "Disease Risk": 0.01
  },
  "contributing_factors": [
    "NDVI decline: -0.15 over 7 days",
    "Low rainfall: 5mm in last 14 days",
    "High temperature: 3 days >35°C"
  ],
  "timestamp": "2026-02-15T10:30:00Z"
}
```

**Deployment**:
- SageMaker Real-Time Endpoint for on-demand inference
- Model versioning with aliases (prod, staging)
- Auto-scaling based on request volume
- Endpoint monitoring for latency and errors

#### 3.3.2 Model Training Pipeline (SageMaker)

**Training Data Sources**:
- Historical satellite imagery with ground truth labels
- Expert-labeled crop stress examples
- Farmer feedback on alert accuracy
- Field validation data from extension workers

**Training Process**:
1. Data collection and labeling
2. Feature engineering and preprocessing
3. Train/validation/test split (70/15/15)
4. Hyperparameter tuning (SageMaker Automatic Model Tuning)
5. Model evaluation and validation
6. Model registration and versioning
7. Deployment to staging endpoint
8. A/B testing before production promotion

**Retraining Strategy**:
- Scheduled retraining: Quarterly with new seasonal data
- Triggered retraining: When model performance degrades
- Continuous learning: Incorporate farmer feedback

### 3.4 Alert and Advisory Components

#### 3.4.1 Alert Rule Engine (Lambda)

**Purpose**: Determine when and what alerts to send based on ML predictions

**Alert Triggering Rules**:

| Condition | Alert Level | Action |
|-----------|-------------|--------|
| Confidence >0.7 AND Prediction = "Severe Stress" | Critical | Immediate voice + SMS |
| Confidence >0.7 AND Prediction = "Disease Risk" | Critical | Immediate voice + SMS |
| Confidence >0.6 AND Prediction = "Moderate Stress" | High | SMS + App notification |
| Confidence >0.5 AND Prediction = "Early Stress" | Medium | App notification |
| NDVI drop >20% in 7 days | Critical | Immediate alert |
| Confidence <0.5 | Low | No alert, log for review |

**Alert Suppression**:
- No duplicate alerts within 48 hours for same plot
- Batch multiple low-priority alerts into daily digest
- Respect farmer notification preferences (time windows)

**Advisory Content Generation**:
```python
def generate_advisory(prediction, factors, crop_type, language):
    """
    Generate contextual advisory based on prediction
    """
    template = get_template(prediction, crop_type, language)
    advisory = template.format(
        factors=factors,
        recommended_actions=get_actions(prediction, crop_type),
        timing=get_timing_guidance(prediction)
    )
    return advisory
```

**Example Advisory Templates**:

*Early Stress - Rice - English*:
```
"Your rice crop in Plot P123 shows early signs of water stress. 
NDVI has decreased by 0.15 in the last week. 
Recommended action: Check soil moisture and consider irrigation within 2-3 days. 
Monitor for leaf curling. Contact extension worker if stress continues."
```

*Early Stress - Rice - Mizo*:
```
"I plot P123-ah i rice crop-ah tui awm lo vang hian harsatna a nei mek. 
NDVI chu kar khat chhung 0.15 a tlahniam ta. 
Tih tur: Lei tui awm dan enfiah la, ni 2-3 chhungin tui pe rawh. 
Hnah kual dan enfiah. Harsatna a chhunzawm chuan extension worker hnenah hriattir rawh."
```

#### 3.4.2 Voice Advisory Generator (Lambda + Polly)

**Purpose**: Convert text advisories to voice messages in local languages

**Process**:
1. Receive advisory text and language code
2. Apply SSML formatting for natural speech
3. Call Amazon Polly synthesize_speech API
4. Store audio file in S3
5. Generate time-limited download URL
6. Send URL via SMS or app notification

**Polly Configuration**:
```python
{
  "Engine": "neural",
  "LanguageCode": "en-IN",  # or "hi-IN" for Hindi
  "VoiceId": "Aditi",  # Female Indian English voice
  "OutputFormat": "mp3",
  "SampleRate": "16000",  # Optimized for mobile
  "TextType": "ssml"
}
```

**SSML Enhancement**:
```xml
<speak>
    <prosody rate="slow" volume="loud">
        Your rice crop in Plot P123 shows early signs of water stress.
    </prosody>
    <break time="500ms"/>
    <emphasis level="strong">Recommended action:</emphasis>
    Check soil moisture and consider irrigation within 2-3 days.
</speak>
```

### 3.5 Delivery Components

#### 3.5.1 Multi-Channel Notification Service (Lambda)

**Supported Channels**:
1. **SMS**: Text alerts with voice advisory link
2. **Voice Call**: Automated voice advisory (future)
3. **Mobile App**: Push notifications with rich content
4. **USSD**: Feature phone support (future)

**SMS Format**:
```
CropPulse Alert: Your rice crop (Plot P123) shows early water stress. 
Listen to advisory: https://croppulse.in/v/abc123 
Reply HELP for support.
```

**Delivery Logic**:
```python
def deliver_alert(farmer_id, alert, advisory):
    preferences = get_farmer_preferences(farmer_id)
    
    if alert.level == "Critical":
        send_sms(farmer_id, alert.summary, advisory.voice_url)
        if preferences.voice_enabled:
            initiate_voice_call(farmer_id, advisory.audio_file)
    
    if preferences.app_enabled:
        send_push_notification(farmer_id, alert, advisory)
    
    log_delivery(farmer_id, alert.id, channels_used)
```

**Retry Mechanism**:
- SMS: 3 attempts with 5-minute intervals
- Push notification: 2 attempts
- Fallback to alternative channel if primary fails

### 3.6 Data Persistence Components

#### 3.6.1 DynamoDB Tables

**Table: Users**
```
Partition Key: user_id (String)
Attributes:
  - phone_number (String)
  - name (String)
  - language_preference (String)
  - notification_preferences (Map)
  - created_at (Number - timestamp)
  - last_active (Number - timestamp)
```

**Table: FarmPlots**
```
Partition Key: plot_id (String)
Sort Key: user_id (String)
Attributes:
  - coordinates (Map: lat, lon, boundary)
  - area_hectares (Number)
  - crop_type (String)
  - planting_date (String)
  - growth_stage (String)
  - elevation (Number)
  - slope (Number)
GSI: user_id-index (for querying all plots by user)
```

**Table: Alerts**
```
Partition Key: plot_id (String)
Sort Key: timestamp (Number)
Attributes:
  - alert_id (String)
  - prediction (String)
  - confidence (Number)
  - severity (String)
  - advisory_text (String)
  - advisory_voice_url (String)
  - contributing_factors (List)
  - status (String: sent, acknowledged, resolved)
  - acknowledged_at (Number)
TTL: 90 days (auto-delete old alerts)
GSI: user_id-timestamp-index (for user alert history)
```

**Table: WeatherData**
```
Partition Key: location_id (String)
Sort Key: timestamp (Number)
Attributes:
  - temperature_max (Number)
  - temperature_min (Number)
  - rainfall (Number)
  - humidity (Number)
  - wind_speed (Number)
TTL: 365 days
```

**Table: PlotNDVI**
```
Partition Key: plot_id (String)
Sort Key: date (String: YYYY-MM-DD)
Attributes:
  - ndvi_mean (Number)
  - ndvi_std (Number)
  - ndvi_min (Number)
  - ndvi_max (Number)
  - cloud_cover (Number)
  - data_quality (String)
TTL: 730 days (2 years)
```

#### 3.6.2 S3 Bucket Structure

```
croppulse-data-lake/
├── satellite-raw/
│   ├── sentinel2/
│   │   └── {date}/
│   │       └── {tile_id}/
│   │           ├── B04_red.tif
│   │           ├── B08_nir.tif
│   │           └── metadata.json
│   └── landsat/
│       └── {date}/...
├── satellite-processed/
│   └── ndvi/
│       └── {date}/
│           └── {plot_id}_ndvi.tif
├── weather-data/
│   └── {date}/
│       └── {location_id}_weather.json
├── farmer-images/
│   └── {user_id}/
│       └── {plot_id}/
│           └── {timestamp}_image.jpg
├── ml-features/
│   └── {date}/
│       └── {plot_id}_features.json
├── ml-models/
│   └── {model_version}/
│       ├── model.tar.gz
│       ├── training_metrics.json
│       └── validation_results.json
└── voice-advisories/
    └── {date}/
        └── {alert_id}_advisory.mp3
```

**S3 Lifecycle Policies**:
- Raw satellite data: Move to Glacier after 90 days
- Processed NDVI: Move to Glacier after 180 days
- Voice advisories: Delete after 30 days
- ML models: Retain latest 5 versions, archive older

---

## 4. Data Flow Diagrams

### 4.1 End-to-End Data Flow

```
1. DATA COLLECTION (Daily)
   Satellite API → Lambda Fetcher → S3 Raw Storage
   Weather API → Lambda Fetcher → S3 + DynamoDB

2. DATA PROCESSING (Triggered by S3 events)
   S3 Raw → Lambda NDVI Calculator → S3 Processed + DynamoDB
   
3. FEATURE ENGINEERING (Scheduled every 6 hours)
   DynamoDB (NDVI + Weather) → Lambda Feature Pipeline → S3 Features

4. ML INFERENCE (Triggered by new features)
   S3 Features → Lambda Orchestrator → SageMaker Endpoint → Predictions

5. ALERT GENERATION (Immediate)
   Predictions → Lambda Alert Engine → DynamoDB Alerts
   
6. ADVISORY CREATION (Immediate)
   Alert → Lambda Advisory Generator → Polly TTS → S3 Voice Files

7. DELIVERY (Immediate)
   Alert + Advisory → Lambda Notification Service → SNS/SMS → Farmer

8. FEEDBACK LOOP (Async)
   Farmer Acknowledgment → API Gateway → Lambda → DynamoDB
   Field Validation → Extension Worker App → DynamoDB → Model Retraining
```

### 4.2 Real-Time Alert Flow

```
┌─────────────────┐
│ New NDVI Data   │
│ Available       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Feature         │
│ Engineering     │
│ (Lambda)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ ML Inference    │
│ (SageMaker)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐      No Alert
│ Alert Rules     ├──────────────→ Log & Store
│ Engine          │
└────────┬────────┘
         │ Alert Triggered
         ▼
┌─────────────────┐
│ Advisory        │
│ Generation      │
└────────┬────────┘
         │
         ├──────────────┬──────────────┐
         ▼              ▼              ▼
┌──────────────┐ ┌──────────┐ ┌──────────────┐
│ Voice (Polly)│ │ SMS (SNS)│ │ App (Push)   │
└──────────────┘ └──────────┘ └──────────────┘
         │              │              │
         └──────────────┴──────────────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Farmer       │
                 │ Receives     │
                 │ Alert        │
                 └──────────────┘
```

### 4.3 Farmer Interaction Flow

```
Farmer Mobile App / SMS Interface
         │
         ├─→ Register/Login (API Gateway + Lambda)
         │   └─→ Store user data (DynamoDB)
         │
         ├─→ Add Farm Plot (API Gateway + Lambda)
         │   └─→ Store plot data (DynamoDB)
         │
         ├─→ Receive Alert (SNS/Push)
         │   ├─→ View alert details
         │   ├─→ Listen to voice advisory
         │   └─→ Acknowledge alert (API Gateway + Lambda)
         │
         ├─→ Upload Crop Image (Future)
         │   └─→ Trigger image analysis
         │
         └─→ View Alert History (API Gateway + Lambda)
             └─→ Query DynamoDB
```

---

## 5. AI Model Design

### 5.1 Model Training Architecture

**Training Infrastructure**:
- SageMaker Training Jobs with ml.p3.2xlarge instances
- Distributed training for large datasets (future)
- Spot instances for cost optimization

**Model Framework**: TensorFlow/Keras or PyTorch

**Training Script Structure**:
```python
# train.py
import tensorflow as tf
from tensorflow import keras

def create_model(input_dim=20, num_classes=5):
    model = keras.Sequential([
        keras.layers.Dense(128, activation='relu', input_shape=(input_dim,)),
        keras.layers.Dropout(0.3),
        keras.layers.Dense(64, activation='relu'),
        keras.layers.Dropout(0.2),
        keras.layers.Dense(32, activation='relu'),
        keras.layers.Dense(num_classes, activation='softmax')
    ])
    
    model.compile(
        optimizer='adam',
        loss='categorical_crossentropy',
        metrics=['accuracy', 'precision', 'recall']
    )
    
    return model

def train(train_data, val_data, epochs=50):
    model = create_model()
    
    callbacks = [
        keras.callbacks.EarlyStopping(patience=10, restore_best_weights=True),
        keras.callbacks.ReduceLROnPlateau(factor=0.5, patience=5),
        keras.callbacks.ModelCheckpoint('best_model.h5', save_best_only=True)
    ]
    
    history = model.fit(
        train_data,
        validation_data=val_data,
        epochs=epochs,
        callbacks=callbacks,
        class_weight=calculate_class_weights(train_data)
    )
    
    return model, history
```

### 5.2 Model Evaluation Metrics

**Primary Metrics**:
- Accuracy: Overall classification accuracy
- Precision: Minimize false positives (avoid alert fatigue)
- Recall: Maximize true positives (catch all stress events)
- F1-Score: Balance between precision and recall

**Per-Class Metrics**:
```
Class: Early Stress
  - Target Recall: >80% (critical to catch early)
  - Target Precision: >70%

Class: Severe Stress
  - Target Recall: >90% (must not miss)
  - Target Precision: >75%

Class: Healthy
  - Target Precision: >85% (avoid false alarms)
```

**Confusion Matrix Analysis**:
- Monitor misclassifications between adjacent classes
- Acceptable: Early Stress ↔ Moderate Stress
- Unacceptable: Healthy ↔ Severe Stress

### 5.3 Model Inference Pipeline

**Inference Request Format**:
```json
{
  "instances": [
    {
      "plot_id": "MZ-AZ-001-P123",
      "features": {
        "ndvi_current": 0.45,
        "ndvi_7day_slope": -0.02,
        "ndvi_14day_slope": -0.03,
        "ndvi_std": 0.08,
        "rainfall_7day": 5.2,
        "rainfall_14day": 12.5,
        "temp_max_7day": 36.5,
        "temp_min_7day": 22.0,
        "heat_stress_days": 3,
        "moisture_deficit": 0.65,
        "days_since_planting": 45,
        "growth_stage": 2,
        "elevation": 850,
        "slope": 15,
        "neighbor_ndvi_avg": 0.52
      }
    }
  ]
}
```

**Inference Response Format**:
```json
{
  "predictions": [
    {
      "plot_id": "MZ-AZ-001-P123",
      "predicted_class": "Early Stress",
      "confidence": 0.82,
      "probabilities": {
        "Healthy": 0.05,
        "Early Stress": 0.82,
        "Moderate Stress": 0.10,
        "Severe Stress": 0.02,
        "Disease Risk": 0.01
      },
      "feature_importance": {
        "ndvi_7day_slope": 0.35,
        "rainfall_7day": 0.28,
        "heat_stress_days": 0.18,
        "ndvi_current": 0.12,
        "moisture_deficit": 0.07
      },
      "timestamp": "2026-02-15T10:30:00Z"
    }
  ]
}
```

### 5.4 Model Monitoring and Retraining

**Monitoring Metrics**:
- Prediction distribution (detect data drift)
- Average confidence scores (detect model uncertainty)
- Inference latency (performance monitoring)
- Error rates and exceptions

**Retraining Triggers**:
1. Scheduled: Every 3 months (seasonal updates)
2. Performance degradation: Accuracy drops below 70%
3. Data drift detected: Input distribution changes significantly
4. New labeled data: Accumulation of 500+ new labeled examples

**Retraining Process**:
```
1. Collect new labeled data (farmer feedback + field validation)
2. Merge with existing training dataset
3. Retrain model with updated data
4. Evaluate on held-out test set
5. Compare with current production model
6. Deploy to staging endpoint
7. A/B test (10% traffic to new model)
8. Monitor for 7 days
9. Promote to production if performance improves
10. Archive old model version
```

---

## 6. AWS Service Integration

### 6.1 Service Configuration

#### 6.1.1 Lambda Functions

**Function: satellite-data-fetcher**
```yaml
Runtime: python3.11
Memory: 512 MB
Timeout: 300 seconds
Environment Variables:
  - SENTINEL_API_KEY: ${SENTINEL_API_KEY}
  - S3_BUCKET: croppulse-data-lake
  - REGION: ap-south-1
Triggers:
  - CloudWatch Events: cron(0 6 * * ? *)
IAM Role:
  - S3: PutObject, GetObject
  - CloudWatch: PutMetricData, CreateLogStream
```

**Function: ndvi-calculator**
```yaml
Runtime: python3.11
Memory: 1024 MB
Timeout: 180 seconds
Layers:
  - numpy-scipy-layer
  - rasterio-layer
Triggers:
  - S3: ObjectCreated (satellite-raw/)
IAM Role:
  - S3: GetObject, PutObject
  - DynamoDB: PutItem
```

**Function: ml-inference-orchestrator**
```yaml
Runtime: python3.11
Memory: 256 MB
Timeout: 60 seconds
Environment Variables:
  - SAGEMAKER_ENDPOINT: croppulse-model-prod
Triggers:
  - CloudWatch Events: rate(6 hours)
IAM Role:
  - SageMaker: InvokeEndpoint
  - DynamoDB: Query, PutItem
  - S3: GetObject
```

**Function: alert-generator**
```yaml
Runtime: python3.11
Memory: 256 MB
Timeout: 30 seconds
Environment Variables:
  - DYNAMODB_ALERTS_TABLE: Alerts
  - SNS_TOPIC_ARN: ${SNS_TOPIC_ARN}
Triggers:
  - DynamoDB Stream (PlotNDVI table)
IAM Role:
  - DynamoDB: PutItem, Query
  - SNS: Publish
  - Polly: SynthesizeSpeech
```

#### 6.1.2 API Gateway

**REST API: CropPulse API**
```yaml
Name: croppulse-api
Stage: prod
Endpoint Type: Regional
Authentication: AWS_IAM + Custom Authorizer (JWT)

Resources:
  /plots:
    POST: Create new farm plot
    GET: List user's plots
  /plots/{plot_id}:
    GET: Get plot details and status
    PUT: Update plot information
  /plots/{plot_id}/alerts:
    GET: Get alert history for plot
  /alerts/{alert_id}/acknowledge:
    POST: Acknowledge alert
  /upload-image:
    POST: Upload crop image (future)
  /health:
    GET: API health check

Throttling:
  Rate: 100 requests/second
  Burst: 200 requests

CORS: Enabled for mobile app domain
```

#### 6.1.3 SageMaker

**Endpoint Configuration**:
```yaml
EndpointName: croppulse-model-prod
ModelName: croppulse-stress-classifier-v1
InstanceType: ml.t2.medium
InitialInstanceCount: 1
AutoScaling:
  MinCapacity: 1
  MaxCapacity: 5
  TargetValue: 70  # CPU utilization
  ScaleInCooldown: 300
  ScaleOutCooldown: 60
```

**Model Package**:
```
model.tar.gz
├── model/
│   ├── saved_model.pb
│   ├── variables/
│   └── assets/
├── code/
│   ├── inference.py
│   └── requirements.txt
└── model_metadata.json
```

#### 6.1.4 DynamoDB

**Table Configuration**:
```yaml
Tables:
  Users:
    BillingMode: PAY_PER_REQUEST
    PointInTimeRecovery: Enabled
    Encryption: AWS_MANAGED
    
  FarmPlots:
    BillingMode: PAY_PER_REQUEST
    GlobalSecondaryIndexes:
      - IndexName: user_id-index
        KeySchema: user_id (HASH)
        Projection: ALL
    
  Alerts:
    BillingMode: PAY_PER_REQUEST
    StreamEnabled: true
    StreamViewType: NEW_AND_OLD_IMAGES
    TimeToLiveAttribute: ttl
    GlobalSecondaryIndexes:
      - IndexName: user_id-timestamp-index
        KeySchema: 
          - user_id (HASH)
          - timestamp (RANGE)
```

#### 6.1.5 S3

**Bucket Configuration**:
```yaml
BucketName: croppulse-data-lake
Region: ap-south-1
Versioning: Enabled (for ml-models/ only)
Encryption: AES256
PublicAccessBlock: All blocked
CORS: Enabled for mobile app
Lifecycle Rules:
  - satellite-raw/: Glacier after 90 days
  - satellite-processed/: Glacier after 180 days
  - voice-advisories/: Delete after 30 days
Event Notifications:
  - ObjectCreated in satellite-raw/ → Lambda: ndvi-calculator
```

#### 6.1.6 CloudWatch

**Alarms**:
```yaml
Alarms:
  - Name: HighLambdaErrorRate
    Metric: Errors
    Threshold: 10 errors in 5 minutes
    Action: SNS notification to admin
    
  - Name: SageMakerEndpointLatency
    Metric: ModelLatency
    Threshold: >5000ms
    Action: SNS notification + auto-scale
    
  - Name: DynamoDBThrottling
    Metric: UserErrors
    Threshold: >5 in 1 minute
    Action: Increase provisioned capacity
    
  - Name: S3DataIngestionFailure
    Metric: Custom metric from Lambda
    Threshold: No data for 48 hours
    Action: SNS notification to admin
```

**Dashboards**:
- System health overview
- ML model performance metrics
- Alert delivery statistics
- User engagement metrics

### 6.2 Security Configuration

#### 6.2.1 IAM Roles and Policies

**Lambda Execution Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::croppulse-data-lake/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:ap-south-1:*:table/CropPulse*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sagemaker:InvokeEndpoint"
      ],
      "Resource": "arn:aws:sagemaker:ap-south-1:*:endpoint/croppulse-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "polly:SynthesizeSpeech"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

**SageMaker Execution Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::croppulse-data-lake/ml-models/*",
        "arn:aws:s3:::croppulse-data-lake/ml-features/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*"
    }
  ]
}
```

#### 6.2.2 Encryption

**Data at Rest**:
- S3: Server-side encryption (SSE-S3)
- DynamoDB: AWS managed encryption
- SageMaker: Encrypted ML storage volumes

**Data in Transit**:
- API Gateway: TLS 1.2+
- All AWS service communication: HTTPS
- Mobile app: Certificate pinning

#### 6.2.3 Authentication and Authorization

**API Authentication**:
```python
# Custom JWT Authorizer Lambda
def lambda_handler(event, context):
    token = event['authorizationToken']
    
    try:
        # Verify JWT token
        payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
        user_id = payload['user_id']
        
        # Generate IAM policy
        policy = generate_policy(user_id, 'Allow', event['methodArn'])
        
        # Add user context
        policy['context'] = {
            'user_id': user_id,
            'phone_number': payload['phone_number']
        }
        
        return policy
    except jwt.ExpiredSignatureError:
        raise Exception('Unauthorized: Token expired')
    except jwt.InvalidTokenError:
        raise Exception('Unauthorized: Invalid token')
```

**User Registration Flow**:
1. Farmer provides phone number
2. System sends OTP via SMS
3. Farmer enters OTP
4. System verifies OTP and creates user account
5. System generates JWT token (valid for 30 days)
6. Token stored securely on mobile device

---

## 7. Deployment Strategy

### 7.1 Infrastructure as Code

**Tool**: AWS CloudFormation or Terraform

**Stack Structure**:
```
croppulse-infrastructure/
├── network/
│   ├── vpc.yaml
│   └── security-groups.yaml
├── storage/
│   ├── s3-buckets.yaml
│   └── dynamodb-tables.yaml
├── compute/
│   ├── lambda-functions.yaml
│   └── sagemaker-endpoints.yaml
├── api/
│   └── api-gateway.yaml
├── monitoring/
│   ├── cloudwatch-alarms.yaml
│   └── cloudwatch-dashboards.yaml
└── iam/
    └── roles-and-policies.yaml
```

### 7.2 CI/CD Pipeline

**Pipeline Stages**:

```
1. SOURCE
   GitHub Repository → CodePipeline Trigger

2. BUILD
   - Install dependencies
   - Run unit tests
   - Build Lambda deployment packages
   - Build Docker images for SageMaker
   - Run linting and security scans

3. TEST (Staging Environment)
   - Deploy to staging
   - Run integration tests
   - Run model validation tests
   - Performance testing
   - Security testing

4. APPROVAL
   - Manual approval gate
   - Review test results
   - Review deployment plan

5. DEPLOY (Production)
   - Blue/green deployment for Lambda
   - SageMaker endpoint update with traffic shifting
   - Database migrations (if needed)
   - Smoke tests

6. MONITOR
   - CloudWatch metrics
   - Error rate monitoring
   - Rollback if issues detected
```

**Tools**:
- AWS CodePipeline
- AWS CodeBuild
- AWS CodeDeploy
- GitHub Actions (alternative)

### 7.3 Environment Strategy

**Environments**:

| Environment | Purpose | Configuration |
|-------------|---------|---------------|
| Development | Local development and testing | Minimal resources, mock data |
| Staging | Pre-production testing | Production-like, subset of data |
| Production | Live system | Full resources, real data |

**Environment Variables**:
```yaml
Development:
  SAGEMAKER_ENDPOINT: croppulse-model-dev
  S3_BUCKET: croppulse-data-lake-dev
  DYNAMODB_PREFIX: dev-
  LOG_LEVEL: DEBUG

Staging:
  SAGEMAKER_ENDPOINT: croppulse-model-staging
  S3_BUCKET: croppulse-data-lake-staging
  DYNAMODB_PREFIX: staging-
  LOG_LEVEL: INFO

Production:
  SAGEMAKER_ENDPOINT: croppulse-model-prod
  S3_BUCKET: croppulse-data-lake
  DYNAMODB_PREFIX: prod-
  LOG_LEVEL: WARNING
```

### 7.4 Deployment Checklist

**Pre-Deployment**:
- [ ] Code review completed
- [ ] Unit tests passing (>80% coverage)
- [ ] Integration tests passing
- [ ] Security scan completed (no critical issues)
- [ ] Performance testing completed
- [ ] Documentation updated
- [ ] Rollback plan prepared

**Deployment**:
- [ ] Deploy infrastructure changes
- [ ] Deploy Lambda functions
- [ ] Update SageMaker endpoints
- [ ] Run database migrations
- [ ] Update API Gateway configuration
- [ ] Verify health checks

**Post-Deployment**:
- [ ] Smoke tests passing
- [ ] Monitor error rates (15 minutes)
- [ ] Verify alert delivery
- [ ] Check CloudWatch metrics
- [ ] Notify stakeholders
- [ ] Update runbook

### 7.5 Rollback Strategy

**Automated Rollback Triggers**:
- Error rate >5% for 5 minutes
- API latency >5 seconds (p95)
- SageMaker endpoint failures >10%

**Rollback Process**:
1. Trigger rollback (manual or automated)
2. Revert Lambda function versions
3. Shift SageMaker traffic to previous endpoint
4. Revert API Gateway deployment
5. Verify system health
6. Investigate root cause

---

## 8. Scalability and Performance

### 8.1 Horizontal Scaling

**Lambda Auto-Scaling**:
- Concurrent executions: 1000 (reserved)
- Provisioned concurrency: 10 (for critical functions)
- Automatic scaling based on invocation rate

**SageMaker Auto-Scaling**:
```python
# Auto-scaling policy
{
    "TargetTrackingScalingPolicyConfiguration": {
        "TargetValue": 70.0,  # Target CPU utilization
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "SageMakerVariantInvocationsPerInstance"
        },
        "ScaleInCooldown": 300,
        "ScaleOutCooldown": 60
    }
}
```

**DynamoDB Auto-Scaling**:
- On-demand billing mode (automatic scaling)
- No capacity planning required
- Handles traffic spikes automatically

### 8.2 Performance Optimization

**Caching Strategy**:
```
1. API Gateway Cache
   - Cache GET requests for plot status (TTL: 5 minutes)
   - Cache weather data (TTL: 1 hour)

2. Lambda Layer Caching
   - Pre-load ML model artifacts
   - Cache frequently accessed S3 objects

3. DynamoDB DAX (Future)
   - Cache frequently queried data
   - Reduce read latency to microseconds
```

**Data Processing Optimization**:
- Batch processing for multiple plots
- Parallel Lambda invocations
- Efficient NDVI calculation using vectorized operations
- Compressed data transfer (gzip)

**Database Query Optimization**:
- Use GSIs for efficient queries
- Batch read/write operations
- Consistent read only when necessary
- Projection expressions to fetch only required attributes

### 8.3 Geographic Scaling

**Multi-Region Strategy (Future)**:

```
Primary Region: ap-south-1 (Mumbai)
  - Serves: Northeast India, Eastern India
  
Secondary Region: ap-south-2 (Hyderabad) - Future
  - Serves: Southern India
  
Disaster Recovery Region: ap-southeast-1 (Singapore)
  - Backup and failover
```

**Data Replication**:
- S3 Cross-Region Replication for critical data
- DynamoDB Global Tables for multi-region access
- SageMaker endpoints in multiple regions

### 8.4 Load Testing Results (Target)

**Expected Load**:
- 10,000 active farmers
- 50,000 farm plots
- 500 alerts per day
- 2,000 API requests per hour

**Performance Targets**:
- API response time: <500ms (p95)
- ML inference: <3 seconds
- Alert delivery: <2 hours from detection
- System availability: 99.5%

---

## 9. Monitoring and Observability

### 9.1 Metrics and KPIs

**System Metrics**:
- Lambda invocation count, duration, errors
- SageMaker endpoint invocations, latency, errors
- API Gateway request count, latency, 4xx/5xx errors
- DynamoDB read/write capacity, throttles
- S3 request count, data transfer

**Business Metrics**:
- Alerts generated per day
- Alert acknowledgment rate
- Average time to acknowledgment
- Farmer active users (DAU, MAU)
- Plot coverage and data quality

**ML Model Metrics**:
- Prediction distribution
- Average confidence scores
- Inference latency
- Model version usage
- Farmer feedback on accuracy

### 9.2 Logging Strategy

**Log Levels**:
- ERROR: System failures, exceptions
- WARNING: Degraded performance, retries
- INFO: Normal operations, state changes
- DEBUG: Detailed diagnostic information (dev only)

**Structured Logging**:
```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def log_event(event_type, details):
    log_entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "event_type": event_type,
        "details": details,
        "environment": os.environ.get("ENVIRONMENT"),
        "function_name": os.environ.get("AWS_LAMBDA_FUNCTION_NAME")
    }
    logger.info(json.dumps(log_entry))

# Usage
log_event("alert_generated", {
    "plot_id": "MZ-AZ-001-P123",
    "prediction": "Early Stress",
    "confidence": 0.82
})
```

**Log Aggregation**:
- CloudWatch Logs for all Lambda functions
- CloudWatch Insights for log analysis
- Log retention: 30 days (production), 7 days (dev)

### 9.3 Alerting

**Critical Alerts** (Immediate notification):
- System down or unavailable
- Data ingestion failure >24 hours
- ML endpoint errors >10%
- Database throttling or errors

**Warning Alerts** (Notification within 1 hour):
- High latency (>5 seconds)
- Low alert acknowledgment rate (<50%)
- Model confidence degradation
- Unusual prediction distribution

**Info Alerts** (Daily digest):
- Daily summary of system health
- Usage statistics
- Cost optimization recommendations

### 9.4 Dashboards

**Operations Dashboard**:
- System health overview
- Real-time error rates
- API and Lambda performance
- Database metrics

**Business Dashboard**:
- Active users and plots
- Alerts generated and delivered
- Farmer engagement metrics
- Geographic distribution

**ML Dashboard**:
- Model performance metrics
- Prediction distribution
- Confidence score trends
- Feature importance

---

## 10. Cost Optimization

### 10.1 Cost Breakdown (Estimated Monthly)

**For 10,000 farmers, 50,000 plots**:

| Service | Usage | Estimated Cost (USD) |
|---------|-------|---------------------|
| Lambda | 5M invocations, 512MB, 30s avg | $25 |
| SageMaker | 1 ml.t2.medium instance | $50 |
| DynamoDB | 10M reads, 2M writes | $15 |
| S3 | 500GB storage, 10M requests | $15 |
| API Gateway | 2M requests | $7 |
| CloudWatch | Logs and metrics | $10 |
| SNS/SMS | 15,000 SMS/month | $150 |
| Polly | 500,000 characters | $20 |
| Data Transfer | 100GB | $10 |
| **Total** | | **~$300/month** |

**Cost per Farmer**: ~$0.03/month

### 10.2 Cost Optimization Strategies

**Compute**:
- Use Lambda instead of EC2 (pay per use)
- Right-size SageMaker instances
- Use Spot instances for training
- Implement Lambda provisioned concurrency only for critical functions

**Storage**:
- S3 lifecycle policies (move to Glacier)
- Compress data before storage
- Delete temporary files
- Use S3 Intelligent-Tiering

**Database**:
- DynamoDB on-demand billing (no over-provisioning)
- Use TTL to auto-delete old data
- Optimize query patterns to reduce reads

**Data Transfer**:
- Minimize cross-region transfers
- Use CloudFront for static content (future)
- Compress API responses

**Monitoring**:
- Optimize log retention periods
- Use metric filters instead of storing all logs
- Sample detailed logs (10% in production)

---

## 11. Disaster Recovery and Business Continuity

### 11.1 Backup Strategy

**Data Backup**:
- DynamoDB: Point-in-time recovery (35 days)
- S3: Versioning enabled for critical data
- ML models: Versioned in S3 with lifecycle policy

**Backup Schedule**:
- DynamoDB: Continuous (PITR)
- S3: Automatic versioning
- Configuration: Daily snapshots to separate bucket

### 11.2 Recovery Objectives

**RTO (Recovery Time Objective)**: 4 hours
- Time to restore system to operational state

**RPO (Recovery Point Objective)**: 1 hour
- Maximum acceptable data loss

### 11.3 Disaster Recovery Plan

**Scenario 1: Lambda Function Failure**
- Detection: CloudWatch alarm
- Action: Automatic rollback to previous version
- Recovery Time: <5 minutes

**Scenario 2: SageMaker Endpoint Failure**
- Detection: Health check failure
- Action: Failover to backup endpoint
- Recovery Time: <15 minutes

**Scenario 3: DynamoDB Table Corruption**
- Detection: Data validation errors
- Action: Restore from point-in-time backup
- Recovery Time: <2 hours

**Scenario 4: Region Outage**
- Detection: Multi-region health checks
- Action: Failover to DR region (manual)
- Recovery Time: <4 hours

### 11.4 Testing

**DR Testing Schedule**:
- Quarterly: Backup restoration test
- Bi-annually: Full DR failover test
- Annually: Region failover simulation

---

## 12. Future Enhancements and Scalability

### 12.1 Phase 2 Features (Post-MVP)

**Enhanced AI Capabilities**:
- Computer vision model for farmer-uploaded crop images
- Multi-modal fusion (satellite + images + weather)
- Pest and disease identification
- Yield prediction models
- Personalized recommendations based on farm history

**Advanced Features**:
- Offline mobile app with sync capability
- Community forum for farmer knowledge sharing
- Integration with agricultural input suppliers
- Market price integration for harvest timing
- Crop insurance claim automation
- WhatsApp Business API integration
- USSD support for feature phones

**Analytics and Insights**:
- Predictive analytics for seasonal planning
- Regional crop health dashboards
- Trend analysis and early warning systems
- Impact assessment and ROI calculation

### 12.2 Scalability Roadmap

**Year 1 (MVP)**:
- 1 district in Mizoram
- 100 farmers, 500 plots
- 2-3 crop types
- Basic alert system

**Year 2**:
- 5 districts across Northeast India
- 5,000 farmers, 25,000 plots
- 10 crop types
- Image analysis capability
- Mobile app launch

**Year 3**:
- 3 states (Mizoram, Manipur, Nagaland)
- 50,000 farmers, 250,000 plots
- 20 crop types
- Multi-language support (5 languages)
- Community features

**Year 5**:
- Pan-India coverage (hilly regions)
- 500,000 farmers, 2.5M plots
- All major crops
- Advanced AI models
- Integration with government systems

### 12.3 Technical Debt and Improvements

**Code Quality**:
- Comprehensive unit test coverage (>90%)
- Integration test automation
- Code documentation and API specs
- Refactoring for maintainability

**Architecture**:
- Microservices decomposition (if needed)
- Event-driven architecture with EventBridge
- GraphQL API for flexible queries
- Real-time updates with WebSockets

**Performance**:
- Edge computing for faster inference
- Model quantization for efficiency
- Database query optimization
- CDN for static content

**Security**:
- Regular security audits
- Penetration testing
- Compliance certifications
- Bug bounty program

### 12.4 Integration Opportunities

**Government Systems**:
- Integration with PM-KISAN database
- Soil Health Card data integration
- Agricultural census data
- Weather department APIs

**Agricultural Ecosystem**:
- Input suppliers (seeds, fertilizers)
- Agricultural extension services
- Crop insurance companies
- Market linkage platforms
- Financial institutions (credit)

**Research Institutions**:
- Data sharing for agricultural research
- Collaborative model development
- Field trial partnerships
- Knowledge dissemination

---

## 13. Assumptions and Constraints

### 13.1 Technical Assumptions

**Data Availability**:
- Sentinel-2 satellite data available every 5 days (cloud cover permitting)
- Weather data APIs accessible with <1 hour latency
- Mobile network coverage available in target regions (at least 2G)

**Infrastructure**:
- AWS services available in ap-south-1 region
- No significant AWS service outages during critical periods
- Internet connectivity for farmers (intermittent acceptable)

**Model Performance**:
- Sufficient labeled training data available (>5,000 examples)
- Model accuracy achievable with available features
- Transfer learning effective for crop stress detection

### 13.2 User Assumptions

**Farmer Behavior**:
- Farmers willing to adopt digital tools
- Farmers can access basic smartphones or feature phones
- Farmers trust AI-generated advisories (with validation)
- Farmers provide feedback on alert accuracy

**Extension Workers**:
- Extension workers available for ground validation
- Extension workers can use web dashboard
- Extension workers provide training to farmers

### 13.3 Operational Assumptions

**Pilot Phase**:
- 3-month pilot duration sufficient for validation
- Seasonal crop cycle captured during pilot
- Sufficient farmer participation (>50 farmers)

**Partnerships**:
- Government or NGO partnership for farmer outreach
- Agricultural experts available for advisory content
- Local language translators available

### 13.4 Constraints

**Budget Constraints**:
- MVP development within hackathon timeline
- Operational costs <$500/month for pilot
- Scalable cost model for expansion

**Technical Constraints**:
- Must use AWS services (hackathon requirement)
- Limited to freely available satellite data
- No expensive sensors or hardware
- Must work with low-bandwidth networks

**Regulatory Constraints**:
- Compliance with India's data protection laws
- No sharing of farmer data without consent
- Agricultural advisory regulations (if any)

**Resource Constraints**:
- Limited development team (2-3 developers)
- Limited ML expertise for advanced models
- Limited field testing resources

### 13.5 Risks and Mitigations

**Risk: Poor satellite data quality due to cloud cover**
- Mitigation: Use multi-temporal analysis, weather-based interpolation, combine with ground observations

**Risk: Low farmer adoption**
- Mitigation: Voice-first interface, local language support, extension worker training, community demonstrations

**Risk: Model accuracy insufficient**
- Mitigation: Start with high-confidence alerts only, combine with expert validation, continuous model improvement

**Risk: High operational costs**
- Mitigation: Serverless architecture, cost monitoring, optimization strategies, tiered service model

**Risk: Network connectivity issues**
- Mitigation: Offline capabilities, SMS fallback, retry mechanisms, local caching

**Risk: Data privacy concerns**
- Mitigation: Strong encryption, transparent policies, minimal data collection, farmer control over data

---

## 14. Success Criteria

### 14.1 Technical Success Criteria

- [ ] End-to-end data pipeline operational
- [ ] ML model deployed with >70% accuracy
- [ ] API response time <3 seconds (p95)
- [ ] System uptime >99% during pilot
- [ ] Alerts delivered within 2 hours of detection
- [ ] Voice advisories generated successfully in 2 languages

### 14.2 User Success Criteria

- [ ] >50 farmers onboarded during pilot
- [ ] >70% alert acknowledgment rate
- [ ] >4/5 user satisfaction score
- [ ] >60% farmers report actionable insights
- [ ] <20% false positive rate (farmer feedback)

### 14.3 Business Success Criteria

- [ ] Demonstrated reduction in crop loss (>10%)
- [ ] Cost per farmer <$0.05/month
- [ ] Positive feedback from agricultural experts
- [ ] Interest from government/NGO partners
- [ ] Scalability plan validated

### 14.4 Hackathon Demo Criteria

- [ ] Live demonstration of alert generation
- [ ] Voice advisory playback in local language
- [ ] Dashboard showing system metrics
- [ ] Mobile interface demonstration
- [ ] Architecture and design documentation complete
- [ ] Code repository with README and setup instructions

---

## 15. Appendices

### 15.1 Glossary

- **NDVI**: Normalized Difference Vegetation Index - measure of vegetation health
- **Sentinel-2**: European Space Agency satellite providing free imagery
- **SageMaker**: AWS managed machine learning service
- **Lambda**: AWS serverless compute service
- **DynamoDB**: AWS NoSQL database service
- **Polly**: AWS text-to-speech service
- **TTS**: Text-to-Speech
- **JWT**: JSON Web Token for authentication
- **SSML**: Speech Synthesis Markup Language
- **GSI**: Global Secondary Index (DynamoDB)
- **TTL**: Time To Live (auto-deletion)
- **PITR**: Point-In-Time Recovery

### 15.2 References

- AWS Well-Architected Framework
- SageMaker Best Practices
- Sentinel-2 User Guide
- India Meteorological Department APIs
- Digital Personal Data Protection Act, 2023
- Agricultural Extension Best Practices

### 15.3 Contact and Support

**Technical Lead**: [Name]
**Email**: [email]
**GitHub Repository**: [URL]
**Documentation**: [URL]

---

## Document Version Control

- **Version**: 1.0
- **Date**: February 15, 2026
- **Status**: Design Document for MVP
- **Authors**: CropPulse Development Team
- **Reviewers**: Technical Architect, ML Engineer, Product Manager
- **Next Review**: Post-MVP implementation

---

*End of Design Document*
