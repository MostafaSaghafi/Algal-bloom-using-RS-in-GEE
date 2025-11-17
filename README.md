# Algal-bloom-using-RS-in-GEE

## GEE Link: https://code.earthengine.google.com/a0bd008514186b8a2a4d03c31fbf5786

# 🌊 Sentinel-2 NDCI Water Quality Monitoring (Lake Houston)

This GEE script analyzes the **Normalized Difference Chlorophyll Index (NDCI)** for **Lake Houston, Texas, USA**, using Sentinel-2 satellite imagery. The goal is to monitor water quality by tracking changes in chlorophyll-a concentration over time.

---

## 🧐 Line-by-Line Code Explanation

This section breaks down the GEE script, explaining the purpose of every major line and code block.

### Step 1 & 2: Define Study Area and Map Setup

| Line(s) | Code Snippet | Explanation |
| :--- | :--- | :--- |
| `1-10` | `var WaterBodyArea = ee.Geometry.Polygon(...)` | **Defines the Area of Interest (AOI):** Creates an `ee.Geometry.Polygon` object using specific longitude/latitude coordinates to delineate the rectangular study area over Lake Houston. |
| `13` | `Map.centerObject(WaterBodyArea, 10);` | **Map Centering:** Sets the initial view of the GEE map viewer to the AOI at a zoom level of **10**. |
| `14` | `Map.addLayer(WaterBodyArea, {}, 'Study Area: ...', false);` | Adds the AOI boundary to the map as a layer, initially set to be **invisible** (`false`). |

---

### Step 3: Define Time Range

| Line(s) | Code Snippet | Explanation |
| :--- | :--- | :--- |
| `17-18` | `var Startyear = '2025-01-01';` | **Time Range:** Defines the specific start and end dates used for filtering the satellite imagery collection. |

---

### Step 4 & 5: Data Loading, Filtering, and Processing

| Line(s) | Code Snippet | Explanation |
| :--- | :--- | :--- |
| `21` | `var sentinelCollection = ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")` | **Loads Data:** Accesses the **Sentinel-2 Level-2A (Surface Reflectance)** image collection. This data is atmospherically corrected. |
| `22` | `.filterDate(Startyear, Endyear)` | **Time Filter:** Keeps only images acquired within the defined `Startyear` and `Endyear`. |
| `23` | `.filterBounds(WaterBodyArea)` | **Spatial Filter:** Keeps only images that intersect the defined `WaterBodyArea`. |
| `24` | `.filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20))` | **Cloud Filter:** Excludes images that have a cloud cover percentage of **20% or more**. |
| `25` | `.map(function(image){ ... })` | **Image Processing:** Applies the subsequent function to **every image** in the filtered collection. |
| `26` | `var scaledBands = image.select('B.*').multiply(0.0001);` | **Scaling:** Converts Sentinel-2's integer reflectance values (typically $0-10000$) to **true reflectance** values ($0-1$) for calculations. |
| `27` | `var ndwi = scaledBands.normalizedDifference(['B3', 'B8']).rename('NDWI');` | **Calculates NDWI:** Computes the **Normalized Difference Water Index** ($\frac{B3-B8}{B3+B8}$) using Green (B3) and Near-Infrared (B8) to distinguish water. |
| `28` | `var waterMask = ndwi.gt(0.1);` | **Mask Creation:** Creates a binary mask where pixels with **NDWI greater than 0.1** are considered **water**. |
| `29` | `var chlorophyllIndex = scaledBands.normalizedDifference(['B5', 'B4']).rename('NDCI');` | **Calculates NDCI:** Computes the **Normalized Difference Chlorophyll Index** ($\frac{B5-B4}{B5+B4}$) using Red Edge (B5) and Red (B4) bands, which is sensitive to chlorophyll-a. |
| `31` | `return chlorophyllIndex.updateMask(waterMask)` | **Applies Mask:** Returns the calculated NDCI band, but applies the `waterMask` so only water pixels are retained (non-water pixels are transparent). |
| `32` | `.copyProperties(image, ['system:time_start', 'system:time_end']);` | **Preserves Metadata:** Copies essential date properties to the new NDCI image for later charting and export. |
| `35` | `print('Filtered Sentinel-2 Collection:', sentinelCollection);` | **Logging:** Prints a summary of the final filtered image collection to the GEE **Console** tab. |

---

### Step 6: Visualization and Compositing

| Line(s) | Code Snippet | Explanation |
| :--- | :--- | :--- |
| `38` | `var ndciComposite = sentinelCollection.mean().clip(WaterBodyArea);` | **Annual Composite:** Computes the **mean (average) NDCI** value across the **entire** filtered time range, creating a single composite image, and clips it to the AOI. |
| `41` | `Map.addLayer(ndciComposite, {min: -1, max:1, palette: [...]}, 'NDCI', false);` | **Adds Annual Layer:** Displays the mean NDCI composite on the map using a blue (low NDCI) to red (high NDCI) color palette. Set to be **initially invisible** (`false`). |
| `44-45` | `var periodeStart = '2025-05-01';` | **Seasonal Range:** Defines a specific sub-period (e.g., summer/algal bloom season) for comparison. |
| `47-49` | `var ndciPeriode = sentinelCollection.filterDate(...).mean().clip(...);` | **Seasonal Composite:** Filters the collection by the seasonal range, computes the **mean NDCI** for this specific period, and clips it to the AOI. |
| `52` | `Map.addLayer(ndciPeriode, {min: -1, max:1, palette: [...]}, 'NDCI periode');` | **Adds Seasonal Layer:** Displays the seasonal mean NDCI composite, which is visible by default. |

---

### Step 7 & 8: Time Series Charting

| Line(s) | Code Snippet | Explanation |
| :--- | :--- | :--- |
| `55-66` | `var chart = ui.Chart.image.series({...}).setOptions({...});` | **Time Series Chart:** Generates a line chart. It calculates the **mean NDCI** (`ee.Reducer.mean()`) for the entire `WaterBodyArea` at the acquisition date of each image in the collection, showing the trend over time. |
| `69` | `print(chart);` | **Displays Chart:** Prints the generated interactive chart to the GEE **Console** tab. |

---

### Step 9 & 10: Data Export Tasks

| Line(s) | Code Snippet | Explanation |
| :--- | :--- | :--- |
| `72-80` | `Export.image.toDrive({...});` | **Export Image:** Initiates a task to export the **annual mean NDCI composite image** (`ndciComposite`) as a GeoTIFF file to a Google Drive folder named `Water_Chlorophyll`. |
| `83-95` | `var timeSeries = sentinelCollection.map(function(image) {...});` | **Data Preparation:** Iterates through the collection again. For each image, it uses `image.reduceRegion` to extract a single **mean NDCI value** over the entire AOI, creating a feature collection of date-value pairs. |
| `97-101` | `Export.table.toDrive({...});` | **Export Table:** Initiates a task to export the prepared time series data (`timeSeries`) as a **CSV file** to Google Drive, suitable for external spreadsheet or statistical analysis. |
