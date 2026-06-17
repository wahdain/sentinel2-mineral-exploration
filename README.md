Sentinel‑2 Mineral Exploration Pipeline

Automated detection of iron‑oxide (gossans) and hydrothermal clay alteration
zones from Sentinel‑2 Level‑2A imagery. Uses unsupervised K‑Means clustering
to group spectral‑lithological units and extracts georeferenced vector targets
ready for field validation.

Core mineral indices (computed pixel‑wise after BOA reflectance conversion):

  Iron Oxide Index  = Red (B04) / Blue (B02)
  → Highlights ferric iron minerals associated with oxidation and weathering.

  Clay Alteration Index = SWIR‑1 (B11) / SWIR‑2 (B12)
  → Highlights phyllosilicate (clay) minerals typical of hydrothermal systems.

  Vegetation masking: NDVI = (NIR‑Red)/(NIR+Red) ; pixels with NDVI > 0.3
  are discarded to avoid false anomalies from lush cover.

Processing workflow (12 stages, fully automated):

1. Scans the SAFE folder and dynamically finds all required band files.
2. Loads bands B02, B03, B04 (10m) and B11, B12 (20m); resamples 10m bands
   to the 20m grid using bilinear interpolation. Optionally loads B08 (NIR)
   and the Scene Classification (SCL) layer.
3. Builds a valid‑pixel mask – excludes pixels with zero values, clouds,
   cloud shadows, and water (using SCL when available, else a blue‑band
   threshold).
4. Converts digital numbers to surface reflectance using the Sentinel‑2
   offset (1000) and scale (10000), clamping values to [0,1].
5. Computes Iron Oxide, Clay Alteration, and NDVI (if NIR present).
6. Flattens all bands/indices into a feature matrix and standardises it
   using StandardScaler (zero mean, unit variance).
7. Runs K‑Means clustering on the valid pixels. The number of clusters
   (NUM_CLUSTERS) is user‑set; an optional silhouette analysis suggests
   the optimal K for reference.
8. Ranks clusters by a combined score = (mean_iron + mean_clay) × log(pixel_count)
   to identify the most prospective spectral unit. Dynamic percentile
   thresholds (default 85th) define "high‑grade" pixels within that cluster.
9. Vectorises the high‑grade mask into polygons using two phases:
   (a) extracts polygon geometries and transforms centroids to WGS84;
   (b) computes mean iron/clay per polygon via scipy.ndimage.label for
       efficient statistics.
10. Exports GeoTIFFs (cluster map, iron oxide, clay alteration) with LZW
    compression and embedded metadata.
11. Generates CSV tables – all targets, high‑priority targets (area above
    threshold), cluster scores, and perimeter vertices of the champion target.
12. Creates a multi‑panel PNG figure and an interactive Leaflet HTML map
    with polygon pop‑ups and satellite basemap.

Input requirements:

  A Sentinel‑2 L2A SAFE product containing:
  - GRANULE/L2A_*/IMG_DATA/R10m/   (B02, B03, B04, B08 .jp2)
  - GRANULE/L2A_*/IMG_DATA/R20m/   (B11, B12, SCL .jp2)
  The script uses flexible pattern matching – no renaming needed.

Output files (written to the current working directory):

  geology_map.tif                  – cluster classification (uint8)
  iron_oxide.tif                   – continuous iron index (float32)
  clay_alteration.tif              – continuous clay index (float32)
  geology_results.png              – 2×3 summary figure
  field_exploration_targets.csv    – all polygons with ID, coords, area,
                                     iron_mean, clay_mean
  High_Priority_Campaign_Targets.csv – subset with area >= threshold
  champion_target_perimeter.csv    – boundary vertices for the largest target
  cluster_analysis.csv             – per‑cluster scores and averages
  interactive_target_map.html      – Leaflet map for field navigation

Key user‑configurable parameters (set at the top of the script):

  BASE_DIR                  path to your GRANULE/L2A_* folder
  NUM_CLUSTERS              number of K‑Means classes (default 6)
  MAX_KMEANS_SAMPLES        max pixels used for clustering (default 200k)
  AREA_THRESHOLD_METERS     min area (m²) for "high priority" (default 5000)
  MIN_POLYGON_AREA          absolute min polygon area to keep (default 100)
  HIGH_GRADE_IRON_PERCENTILE   threshold for high‑grade iron (default 85)
  HIGH_GRADE_CLAY_PERCENTILE   threshold for high‑grade clay (default 85)
  CLOUD_MASK_THRESHOLD      fallback cloud mask if SCL missing (default 0.15)
  BOA_OFFSET                Sentinel‑2 offset (default 1000)

Installation and execution:

  Install dependencies:
    pip install numpy rasterio scipy scikit‑learn matplotlib pandas pyproj folium shapely

  Run the pipeline:
    python lithos_s2_pipeline.py

  The script prints progress for each stage and a final summary with runtime,
  valid‑pixel percentage, target counts, and largest target area.

Typical runtime for a full 100×100 km tile is 2‑5 minutes on a standard
laptop. All outputs are GIS‑ and field‑ready – load the CSVs into a GPS
or open the HTML map on a tablet for direct navigation to priority targets.
