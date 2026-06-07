# Optimal New Facility Siting - Edmonton
**Multi-Criteria Evaluation (MCE) Suitability Analysis**
Project file: `Optimal_New_Facility_Siting.qgz`
CRS: EPSG:3776 (NAD83 / Alberta 10-TM Forest)
Pixel resolution: 100 m
Produced: May 2026

---

## 1. Objective

Identify the most suitable locations for a new public community facility in Edmonton
using a weighted overlay of three spatial criteria. The goal is to pinpoint areas that
are simultaneously:
- underserved by existing civic attractions,
- underserved by existing community resources, and
- accessible within the urban core.

The result is a continuous suitability surface (0-100), a five-class suitability map,
and a set of five top-candidate zone polygons for field evaluation.

---

## 2. Input Data

| Layer                    | Source CRS  | Features | Description                                       |
|--------------------------|-------------|----------|---------------------------------------------------|
| `city_boundary`          | EPSG:3776   | 1 polygon | City of Edmonton administrative boundary         |
| `civic_attraction`       | EPSG:4326   | 56 points | Existing civic attractions (pools, arenas, etc.) |
| `community_resources`    | EPSG:3857   | 502 points| Community programs and services across Edmonton  |
| `edmonton_neighbourhoods`| EPSG:3776   | 400 polys | Neighbourhood boundaries (study context)         |

All inputs were reprojected to EPSG:3776 before processing.

---

## 3. Methodology

### 3.1 Criteria Definition and Weights

| #  | Criterion                  | Rationale                                                        | Weight |
|----|----------------------------|------------------------------------------------------------------|--------|
| C1 | Civic Attraction Gap       | Farther from existing civic attractions = underserved = suitable | 0.40   |
| C2 | Community Resource Gap     | Farther from existing community resources = underserved          | 0.35   |
| C3 | Urban Core Accessibility   | Closer to city centroid = more accessible to more residents      | 0.25   |
|    |                            | Total                                                            | 1.00   |

Weights reflect the higher planning priority of filling service gaps (C1+C2 = 75%)
while maintaining urban accessibility (C3 = 25%).

### 3.2 Processing Pipeline

```
Step 1  Reproject civic_attraction  (EPSG:4326  -> EPSG:3776)
        Reproject community_resources (EPSG:3857 -> EPSG:3776)

Step 2  Rasterize each point layer to 100m grid (extent = city_boundary)
        Burn value = 1 at point locations, 0 elsewhere

Step 3  gdal:proximity on each burn raster
        -> Euclidean distance surface in metres for each layer
        -> Additional: distance from city centroid for accessibility

Step 4  Min-max normalisation of each distance surface
        Restricted to city_boundary mask (78,282 pixels inside boundary)
        C1, C2: no inversion  (higher distance = higher score)
        C3:     inverted      (lower distance from centroid = higher score)
        All criteria scaled to [0, 1]

Step 5  Weighted sum
        Suitability_raw = 0.40*C1 + 0.35*C2 + 0.25*C3

Step 6  Rescale to 0-100 for cartographic output (suitability_score.tif)

Step 7  Percentile-based classification into 5 equal-frequency classes
        Breaks at 20th, 40th, 60th, 80th percentiles of in-mask values

Step 8  Polygonise class 5 (Very High) pixels -> top candidate zones
        Clean to 5 contiguous polygon features
```

### 3.3 Suitability Classes

| Class | Label       | Percentile range |
|-------|-------------|------------------|
| 1     | Very Low    | 0th - 20th       |
| 2     | Low         | 20th - 40th      |
| 3     | Moderate    | 40th - 60th      |
| 4     | High        | 60th - 80th      |
| 5     | Very High   | 80th - 100th     |

Percentile break values (raw weighted scores):
  0.241  (Class 1/2 boundary)
  0.275  (Class 2/3 boundary)
  0.319  (Class 3/4 boundary)
  0.412  (Class 4/5 boundary)

---

## 4. Output Files

### Primary Deliverables

| File                               | Type   | Description                                         |
|------------------------------------|--------|-----------------------------------------------------|
| `Optimal_New_Facility_Siting.qgz`  | QGIS   | Main project - all layers, symbology, metadata      |
| `suitability_score.tif`            | Raster | Final suitability surface, scaled 0-100             |
| `suitability_classified.tif`       | Raster | Five-class suitability map (1=Very Low, 5=Very High)|
| `top_candidate_zones_final.gpkg`   | Vector | 5 top-candidate zone polygons (Class 5 areas)       |

### Criteria Surfaces (Normalised)

| File                                | Criterion                    | Weight |
|-------------------------------------|------------------------------|--------|
| `criteria_01_civic_gap_norm.tif`    | Civic attraction gap         | 0.40   |
| `criteria_02_resource_gap_norm.tif` | Community resource gap       | 0.35   |
| `criteria_03_accessibility_norm.tif`| Urban core accessibility     | 0.25   |

### Intermediate / Supporting Files

| File                                | Description                                          |
|-------------------------------------|------------------------------------------------------|
| `civic_attraction_3776.gpkg`        | Civic attractions reprojected to EPSG:3776           |
| `community_resources_3776.gpkg`     | Community resources reprojected to EPSG:3776         |
| `criteria_01_civic_gap_raw.tif`     | Raw proximity raster (metres) - civic attractions    |
| `criteria_02_resource_gap_raw.tif`  | Raw proximity raster (metres) - community resources  |
| `criteria_03_accessibility_raw.tif` | Raw proximity raster (metres) - city centroid        |
| `suitability_raw.tif`               | Weighted sum before 0-100 rescaling                  |
| `city_boundary_mask.tif`            | Binary mask (1 = inside city boundary)               |
| `city_centroid.gpkg`                | Computed centroid of city boundary (single point)    |
| `ca_burn.tif`, `cr_burn.tif`        | Point burn rasters used as proximity inputs          |

---

## 5. How to Open and Reproduce

### Open the project
1. Launch QGIS 3.x
2. Open `Optimal_New_Facility_Siting.qgz`
   All layers load via paths relative to this directory.
   Do not rename or move the folder without updating layer paths.

### Reproduce from scratch
All processing was executed via PyQGIS using the QGIS MCP plugin.
Pipeline steps are documented in Section 3.2. Re-run in sequence:
  Phase 1 - Reproject + burn rasters
  Phase 2 - Proximity rasters
  Phase 3 - Normalise, weighted sum, classify
  Phase 4 - Polygonise top zones, add layers, save

### Adjust weights
Edit the weight values in Phase 3:
    w1, w2, w3 = 0.40, 0.35, 0.25
Ensure w1 + w2 + w3 == 1.0 and re-run Phase 3 onwards.

### Change pixel resolution
Edit pixel_size = 100 in Phase 1 (affects all rasterize steps).
Finer resolution (e.g., 50m) increases processing time and file size.

---

## 6. Interpretation Notes

The 5 top candidate zones are contiguous areas where all three service-gap
conditions are simultaneously most favourable. Ground-truth checks should verify:
  - Land availability and ownership
  - Zoning classification
  - Infrastructure access (roads, utilities)
  - Environmental constraints not captured in this analysis

Criterion 3 (accessibility) favours inner-city locations. If peripheral
growth areas are a planning priority, consider reducing w3 or replacing
with a population growth raster.

Community resource data (502 points) is denser in the urban core, so
peripheral areas naturally score high on the resource gap criterion.
This is analytically consistent but should be interpreted alongside
actual service-area boundaries for each resource type.

---

## 7. Technical Environment

  QGIS 3.x with Python 3
  Libraries: GDAL/OGR, PyQGIS, numpy, processing framework
  Algorithms: native:reprojectlayer, gdal:rasterize, gdal:proximity,
              gdal:polygonize, native:extractbyexpression
  Numpy used for normalisation, weighted sum, percentile classification

---

Analysis by Martins | May 2026
