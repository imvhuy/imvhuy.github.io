+++
title = "2.1 OpenWeatherMap API Setup"
date = 2025-01-03T08:30:00+07:00
weight = 2
+++

In this simplified section, we will quickly set up an OpenWeatherMap API account to collect weather data for our pipeline.

## Step 1: Sign up for OpenWeatherMap

1. **Create an account**
   - Go to [https://openweathermap.org](https://openweathermap.org) and click "Sign Up"
   - Complete the registration with your email
   - Verify your email and log in

![Create account](/images/data-collection/21b1.png)

## Step 2: Get API Key

1. **Access API Keys**
   - After logging in, go to the "API keys" section
   - Note down the default API key or create a new one named "weather-data-collection"

![Get API key](/images/data-collection/21b2.png)

{{% notice info %}}
The free plan includes 1,000 API calls per day and 60 calls per minute.
{{% /notice %}}

## Step 3: Test API Key

Test your API key using one of the following methods:

**Browser method:**

```
https://api.openweathermap.org/data/2.5/weather?lat=10.7769&lon=106.7009&appid=YOUR_API_KEY
```

![check API key](/images/data-collection/21b31.png)

**cURL method:**

```bash
curl "https://api.openweathermap.org/data/2.5/weather?lat=10.7769&lon=106.7009&appid=YOUR_API_KEY"
```

![check API key](/images/data-collection/21b32.png)

You should see a JSON response with the current weather data for Ho Chi Minh City.

## Main API Endpoints to be Used

```
# Current weather
https://api.openweathermap.org/data/2.5/weather?q={city}&appid={API key}
```

## Next Steps

That's it! You now have a working OpenWeatherMap API key securely stored in Parameter Store. In the next section, we will create a Lambda function to collect weather data.

**Completed:**

- Created OpenWeatherMap account
- Retrieved API key
- Stored API key in AWS Parameter Store
