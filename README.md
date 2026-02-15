# CropPulse 🌾

AI-Powered Early Crop Stress Detection for Smallholder Farmers

---

## Overview

CropPulse is an intelligent agricultural monitoring system designed to help smallholder farmers in India's hilly and remote regions detect crop stress early and take timely action. By combining satellite imagery, weather data, and AI-powered analysis, CropPulse delivers actionable advisories in local languages through voice and SMS.

## Problem Statement

Farmers in northeastern states like Mizoram face chronic yield losses due to:
- Delayed detection of crop stress and diseases
- Limited access to agricultural experts
- Fragmented landholdings and irregular connectivity
- Rapidly changing micro-climates
- Manual monitoring that is unreliable and reactive

## Solution

CropPulse uses AI to analyze:
- **Satellite NDVI data** (vegetation health indices)
- **Weather signals** (temperature, rainfall, humidity)
- **Optional farmer images** (future enhancement)

The system generates early warnings and delivers voice-based advisories in local languages, enabling farmers to act before significant crop damage occurs.

## Key Features

- **Automated Satellite Monitoring**: Daily analysis of crop health using free Sentinel-2 imagery
- **AI-Powered Detection**: Machine learning model classifies crop stress levels with confidence scores
- **Voice Advisories**: Text-to-speech in local languages (Mizo, Hindi, English) for low-literacy users
- **Low-Bandwidth Delivery**: SMS and lightweight mobile interface for 2G/3G networks
- **Advisory History**: Track alerts and recommendations over time
- **Privacy-First**: Farmer data encrypted and never shared without consent

## Technology Stack

### AWS Services
- **Amazon SageMaker**: ML model training and inference
- **AWS Lambda**: Serverless data processing and orchestration
- **Amazon S3**: Data lake for satellite imagery and features
- **Amazon DynamoDB**: NoSQL database for alerts and user data
- **Amazon API Gateway**: RESTful API for mobile and web interfaces
- **Amazon Polly**: Text-to-speech for voice advisories
- **Amazon CloudWatch**: Monitoring and logging

### Languages & Frameworks
- **Python**: Backend services and ML models
- **TensorFlow/PyTorch**: Deep learning frameworks
- **FastAPI**: API development

## Architecture

```
Satellite Data → Data Processing → AI Model → Alert Engine → Farmer
     ↓              ↓                ↓            ↓           ↓
   AWS S3      AWS Lambda      SageMaker    DynamoDB    SMS/Voice
```

See [design.md](design.md) for detailed architecture diagrams and component specifications.

## MVP Scope (Hackathon)

The initial prototype focuses on:
- One district in Mizoram (pilot region)
- 2-3 common crop types (rice, maize, vegetables)
- 50-100 sample farm plots
- Satellite NDVI data ingestion and processing
- Weather data integration
- Basic ML model for crop stress classification
- Voice advisory delivery in 2 languages
- SMS-based alerts
- Extension worker dashboard



## Usage

### For Farmers
1. Register via SMS or mobile app with phone number
2. Add farm plot details (location, crop type)
3. Receive automatic alerts when crop stress is detected
4. Listen to voice advisories in your language
5. Acknowledge alerts and provide feedback

### For Extension Workers
1. Access web dashboard
2. View all alerts in your region
3. Validate AI predictions with field observations
4. Add manual observations to improve model accuracy

### For Administrators
1. Monitor system health via CloudWatch dashboards
2. Review alert delivery statistics
3. Manage user accounts and farm plots
4. Trigger model retraining when needed

## Roadmap

### Phase 1 (MVP) - Current
- Basic satellite monitoring and alerts
- Voice advisories in 2 languages
- SMS delivery

### Phase 2 (3-6 months)
- Farmer image upload and analysis
- Mobile app with offline support
- 5 additional languages
- Expansion to 5 districts

### Phase 3 (6-12 months)
- Multi-modal AI (satellite + images + weather)
- Pest and disease identification
- Yield prediction
- Integration with government systems

### Phase 4 (1-2 years)
- Pan-India coverage for hilly regions
- Community knowledge sharing
- Market price integration
- Crop insurance automation

## Documentation

- [Requirements Specification](requirements.md) - Detailed functional and non-functional requirements
- [System Design](design.md) - Architecture, components, and AWS integration


## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

## Acknowledgments

- Farmers and extension workers in Mizoram for their invaluable feedback
- European Space Agency for free Sentinel-2 satellite data
- India Meteorological Department for weather data
- AWS for cloud infrastructure support

