---
title: "Data Visualization with QuickSight"
date: 2025-01-07T09:00:00+00:00
weight: 50
chapter: false
pre: "<b>5. </b>"
---

Once the weather data is stored and queryable via Athena, it's time for visualization. In this section, we will use Amazon QuickSight to build an interactive dashboard that brings the weather data to life.

## Overview

Amazon QuickSight is AWS's business intelligence (BI) service that makes it easy to create and publish interactive dashboards. You will connect QuickSight to your Athena data source and create visualizations that reveal patterns and insights from the weather data.

## Step 1: Set up Amazon QuickSight

### 1.1 Sign up for QuickSight

1. Access QuickSight
   - Log in to the AWS Console.
   - In the search bar, type QuickSight and select Amazon QuickSight.
2. Click **Sign up for QuickSight**
3. Choose **Standard Edition** (includes a 30-day free trial)
4. Enter your account information:
   - **Account name**: `weather-analytics-[your-name]`
   - **Notification email**: Your email address
5. Click **Finish**

![Quicksight](/images/analytics-visualization/5b1.png)
![Quicksight](/images/analytics-visualization/5b2.png)

### 1.2 Configure QuickSight Permissions

1. In QuickSight, click the **profile icon** (top right)
2. Select **Manage QuickSight**
3. Choose **Security & permissions**
4. Click **Manage**
5. Enable the following services:
   - **Amazon Athena**
   - **Amazon S3**
6. For S3, click **Select S3 buckets**
7. Select your weather data buckets:
   - `your-weather-processed-bucket`
   - `your-athena-query-results-bucket`
8. Click **Save**

![Quicksight](/images/analytics-visualization/5b3.png)

## Step 2: Create Data Source and Dataset

### 2.1 Connect to Athena

1. On the QuickSight homepage, click **Datasets**
2. Click **New dataset**
3. Select **Athena** as the data source
4. Configure the connection:
   - **Data source name**: `Weather-Data-Athena`
   - **Athena workgroup**: `primary` (default)
5. Click **Create data source**

![Quicksight](/images/analytics-visualization/5b4.png)

### 2.2 Create Dataset from Weather Table

1. Select database: `weather_analytics`
2. Select table: `current_weather`
3. Choose **Directly query your data**
4. Click **Visualize**

![Quicksight](/images/analytics-visualization/5b5.png)
![Quicksight](/images/analytics-visualization/5b6.png)

{{% notice tip %}}
If your dataset is small (< 1GB), you can choose "Import to SPICE" for better performance. SPICE is QuickSight's in-memory calculation engine.
{{% /notice %}}

## Step 3: Create Weather Visualizations

### 3.1 Temperature Trend Line Chart

1. **Create a new analysis**:

   - Click **+ Add** → **Add visual**
   - Choose **Line chart**

2. **Configure the chart**:

   - **X-axis**: Drag `data_collection_date` to the X-axis
   - **Value**: Drag `temperature_celsius` to Value
   - **Color**: Drag `city_name` to Color

3. **Customize the chart**:
   - Click the **visual** → **Format visual**
   - **Title**: "Temperature Trends by City"
   - **Y-axis label**: "Temperature (°C)"
   - **Legend**: Place at the bottom

![Quicksight](/images/analytics-visualization/5b7.png)

### 3.2 Weather Conditions Pie Chart

1. **Add a new visual**:

   - Click **+ Add** → **Add visual**
   - Choose **Pie chart**

2. **Configure the chart**:

   - **Group/Color**: Drag `weather_main` to Group/Color
   - **Value**: Drag `city_name` to Value
   - **Aggregate**: Change to **Count**

3. **Customize**:
   - **Title**: "Weather Conditions Distribution"
   - **Legend**: Show percentages

![Quicksight](/images/analytics-visualization/5b8.png)

### 3.3 City Temperature Comparison Bar Chart

1. **Add a new visual**:

   - Choose **Vertical bar chart**

2. **Configure**:

   - **X-axis**: Drag `city_name` to the X-axis
   - **Value**: Drag `temperature_celsius` to Value
   - **Aggregate**: Change to **Average**

3. **Customize**:
   - **Title**: "Average Temperature by City"
   - **Y-axis label**: "Average Temperature (°C)"
   - Sort by value (descending)

![Quicksight](/images/analytics-visualization/5b9.png)

### 3.4 Line chart comparing temperature_celsius and feels_like_celsius

1. **Add a new visual**:

   - Choose **Line chart**

2. **Configure**:

   - **X-axis**: Drag `timestamp` to the X-axis
   - **Value**: Drag `temperature_celsius` and `feels_like_celsius` to Value
   - **Aggregate**: Change to **Average**

3. **Customize**:
   - **Title**: "Comparison of Outdoor Temperature and Feels Like Temperature"
   - **X-axis label**: "Time"

![Quicksight](/images/analytics-visualization/5b10.png)

### 3.5 Key Performance Indicators (KPIs)

Create KPI tiles for the latest weather data:

#### Current Temperature KPI

1. **Add new visual** → **KPI**
2. **Configure**:
   - **Value**: `temperature_celsius`
   - **Aggregate**: **Average**
3. **Filter**: Add a filter for the latest date
4. **Title**: "Current Average Temperature"

#### Humidity KPI

1. **Add KPI visual**
2. **Configure**:
   - **Value**: `humidity_percent`
   - **Aggregate**: **Average**
3. **Title**: "Current Average Humidity"

## Step 4: Build a Comprehensive Dashboard

### 4.1 Organize Dashboard Layout

1. **Resize and arrange visuals**:

   - Place KPIs at the top in a row
   - Temperature trend chart in the main area
   - Pie chart and bar chart side-by-side below

2. **Add dashboard title**:
   - Click **+ Add** → **Add title**
   - Text: "Weather Analysis Dashboard"
   - Style: Large, centered

### 4.2 Add Interactive Filters

1. **Add date filter**:

   - Click the **Filter** pane (left)
   - Click **Create one** → Select `data_collection_date`
   - Filter type: **Date range**
   - Default: Last 7 days

2. **Add city filter**:

   - Create a filter for `city_name`
   - Filter type: **Multi-select dropdown**
   - Show all cities by default

3. **Add weather condition filter**:
   - Create a filter for `weather_main`
   - Filter type: **Multi-select dropdown**

### 4.3 Apply Dashboard Styling

1. **Choose a color theme**:

   - Click **Themes** (top menu)
   - Choose **Midnight** or **Classic**

2. **Customize colors**:

   - For the temperature chart: Use a blue-red gradient
   - For weather conditions: Use distinct colors for each condition

3. **Add descriptive text**:
   - Click **+ Add** → **Add text box**
   - Add insights or instructions for dashboard users

## Step 5: Advanced Visualization

### Wind Direction Chart (Using Calculated Fields)

1. **Create a calculated field**:

   - Click **+ Add** → **Add calculated field**
   - **Name**: `wind_direction_category`
   - **Formula**:

   ```
   ifelse(
     wind_direction_deg >= 337.5 OR wind_direction_deg < 22.5, "N",
     wind_direction_deg >= 22.5 AND wind_direction_deg < 67.5, "NE",
     wind_direction_deg >= 67.5 AND wind_direction_deg < 112.5, "E",
     wind_direction_deg >= 112.5 AND wind_direction_deg < 157.5, "SE",
     wind_direction_deg >= 157.5 AND wind_direction_deg < 202.5, "S",
     wind_direction_deg >= 202.5 AND wind_direction_deg < 247.5, "SW",
     wind_direction_deg >= 247.5 AND wind_direction_deg < 292.5, "W",
     "NW"
   )
   ```

2. **Create wind direction chart**:
