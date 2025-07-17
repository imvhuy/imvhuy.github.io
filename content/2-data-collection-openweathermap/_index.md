---
title: "Data Collection with OpenWeatherMap"
date: 2025-01-07T09:00:00+00:00
weight: 20
chapter: false
pre: "<b>2. </b>"
---

In this section, we will learn how to set up automated weather data collection using the OpenWeatherMap API and AWS Lambda. This is the foundation of our weather analytics ETL pipeline, where we will build a reliable serverless data collection system.

### [2.1 OpenWeatherMap Setup](2.1-openweathermap-setup/)

**API and Credentials Setup**

Set up an OpenWeatherMap account, get an API key, and configure Systems Manager Parameter Store to securely store credentials. You will learn how to manage API keys and test connectivity.

### [2.2 Lambda Weather Collector](2.2-lambda-weather-collector/)

**Building Data Collection Functions**

Create Lambda functions to collect current and forecast weather data from the OpenWeatherMap API. This includes IAM roles, S3 bucket setup, and function code with error handling.

### [2.3 Automated Scheduling](2.3-automated-scheduling/)

**Automated Scheduling with CloudWatch Events**

Set up CloudWatch Events to run Lambda functions on an automated schedule. Configure monitoring, alarms, and notifications to ensure the system runs smoothly.

### [2.4 Testing and Monitoring](2.4-testing-monitoring/)

**Comprehensive Testing and Monitoring**

Establish a testing strategy including manual testing, data quality validation, performance testing, and automated health checks. Create a dashboard to monitor the system.

## Architecture Overview

```mermaid
graph TD
    A[OpenWeatherMap API<br/>Current Weather Data] --> B[Lambda Function<br/>weather-current-collector]
    C[EventBridge Rule<br/>Every Hour] --> B
    B --> D[S3 Raw Storage<br/>current-weather/]
    B --> E[CloudWatch Logs<br/>Monitoring]
    B --> F[CloudWatch Metrics<br/>Success/Error Counts]

    G[6 Vietnamese Cities<br/>HCM, Hanoi, Danang<br/>GiaLai, CanTho, Hue] --> A

    style A fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style B fill:#ff9900,stroke:#232f3e,stroke-width:3px
    style C fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    style D fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style E fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    style F fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style G fill:#f1f8e9,stroke:#558b2f,stroke-width:2px
```

## Data Types Collected

Collect current weather data for **6 Vietnamese cities/provinces**: Ha Noi, Ho Chi Minh City, Da NangGia Lai, Can Tho, Hue

**Data Collected**:

- Temperature (°C, °F), humidity, pressure
- Wind speed, wind direction, cloud cover
- Weather description, weather condition
- Metadata: timestamp, location, collection info

## Collection Schedule

**Current Weather**: Every hour (24 times/day × 6 cities = 144 data points/day)

## Cost Estimation

| Service            | Usage                 | Cost             |
| ------------------ | --------------------- | ---------------- |
| OpenWeatherMap API | 144 calls/day         | **Free**         |
| Lambda Executions  | 720 invocations/month | **Free Tier**    |
| S3 Storage         | 500 MB data           | **Free Tier**    |
| CloudWatch Logs    | 2 GB logs             | $1.00            |
| **Total**          |                       | **~$1.00/month** |

{{% notice tip %}}
OpenWeatherMap provides 1,000 free API calls per day, which is sufficient for this workshop.
{{% /notice %}}

## Expected Outcomes

After completing this module, you will have:

- A 24/7 operational serverless weather data collection system
- Structured weather data stored in S3
- A complete monitoring and alerting system
- Knowledge of AWS Lambda, CloudWatch Events, and S3 integration

## Getting Started

Ready to build a weather data collection system? Start with **[2.1 OpenWeatherMap Setup](2.1-openweathermap-setup/)** to set up your API and credentials.
