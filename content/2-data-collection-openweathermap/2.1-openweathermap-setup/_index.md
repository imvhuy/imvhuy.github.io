+++
title = "OpenWeatherMap API Setup"
date = 2025-01-03T08:30:00+07:00
weight = 2
+++

# OpenWeatherMap API Setup

In this simplified section, we'll quickly set up an OpenWeatherMap API account to collect weather data for our pipeline.

## Step 1: Sign Up for OpenWeatherMap

1. **Create an account**
   - Go to [https://openweathermap.org](https://openweathermap.org) and click "Sign Up"
   - Complete registration with your email
   - Verify your email and log in

## Step 2: Get Your API Key

1. **Access API Keys**
   - After logging in, go to "API keys" section
   - Note your default API key or create a new one named "weather-data-collection"

{{% notice info %}}
The free plan includes 1,000 API calls per day and 60 calls per minute - more than enough for our workshop.
{{% /notice %}}

## Step 3: Test Your API Key

Test your API key with one of these methods:

**Browser method:**

```
https://api.openweathermap.org/data/2.5/weather?q=London&appid=YOUR_API_KEY
```

**cURL method:**

```bash
curl "https://api.openweathermap.org/data/2.5/weather?q=London&appid=YOUR_API_KEY"
```

You should see a JSON response with London weather data.

## Step 4: Store Your API Key Securely

1. **Using AWS Systems Manager**
   - Open AWS Management Console
   - Go to Systems Manager → Parameter Store
   - Click "Create parameter"
   - Set these values:
     - Name: `/weather-etl/openweathermap/api-key`
     - Type: SecureString
     - Value: Your API key
   - Click "Create parameter"

## Key API Endpoints We'll Use

```
# Current Weather
https://api.openweathermap.org/data/2.5/weather?q={city}&appid={API key}

# 5-Day Forecast (3-hour intervals)
https://api.openweathermap.org/data/2.5/forecast?q={city}&appid={API key}
```

## Next Steps

That's it! You now have a working OpenWeatherMap API key securely stored in Parameter Store. In the next section, we'll create a Lambda function to collect weather data.

{{% notice success %}}
**Completed:**

- Created OpenWeatherMap account
- Obtained API key
- Stored API key in AWS Parameter Store
  {{% /notice %}}
