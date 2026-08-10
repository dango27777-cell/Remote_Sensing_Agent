# Remote Sensing Image Processing Agent

## System Prompt

```
You are an expert agent in high-resolution satellite image processing. Your knowledge covers:

1. Pan-sharpening — Gram-Schmidt, NNDiffuse, Brovey
2. Band standardization (BGRNIR ↔ RGBNIR)
3. Geometric registration (MSS → PAN)
4. Radiometric harmonization (Color Balancing, Bundle Adjustment, DOS)
5. Multi-scene mosaicking (Mosaic, Seamlines, Blending, cblend)
6. Quality assurance (Metadata, Statistics, Quicklook)

Toolchain: GDAL, Rasterio, NumPy, SciPy, ArcPy (ArcGIS Pro integration)

Your responses must:
- Provide a SINGLE definitive approach, not a list of alternatives
- Distinguish data quality from display rendering — never conflate them
- Prefer deterministic methods (same input → same output)
- Flag which steps require ArcGIS Pro licensing vs. pure Python
- Reference validated parameter values from real production experience
```

---

## 1. Sensor Data Characteristics (Verified on VHR Broadband Sensor)

| Property | PAN | MSS |
|----------|-----|-----|
| Resolution | 0.5m | 2m |
| Bands | 1 (broadband, ~450-900nm) | 4 (Blue, Green, Red, NIR) |
| Data type | UInt16 | UInt16 |
| Geolocation | RPC (Rational Polynomial Coefficients) | RPC |
| CRS | LOCAL_CS / EngineeringCRS (no EPSG) | Same |
| Native band order | — | B1=Blue, B2=Green, B3=Red, B4=NIR |
| GIS-standard order | — | B1=Red, B2=Green, B3=Blue, B4=NIR |

**Key insight**: PAN and MSS are captured simultaneously from the same platform. Their pixels are spatially aligned in local sensor coordinates. MSS GeoTransform Y-axis sign may differ from PAN.

---

## 2. Recommended Processing Pipeline

```
Step 1: Metadata Check         → metadata_report.csv            [Pure Python/GDAL]
Step 2: MSS → RGBNIR           → gdal.Translate bandList        [Pure GDAL]
Step 3: Geometric Registration → RegisterRaster + CopyRaster     [Needs ArcPy]
Step 4: Pan-sharpen            → Gram-Schmidt (identity mapping)  [Needs ArcPy]
Step 5: Mosaic                 → GDAL BuildVRT + Warp + cblend   [Pure GDAL]
Step 6: Quality Control        → Z-score + Quicklook             [Pure Python]
```

**Only Steps 3-4 require ArcGIS Pro licensing.**

---

## 3. Step-by-Step Reference

### Step 1: Metadata Check

Automatically scan all input TIFF files:

```python
fields = ["file", "width", "height", "bands", "dtype", "crs_wkt", "pixel_size"]
```

Checks: 16-bit depth, 4-band MSS / 1-band PAN, CRS consistency, PAN-MSS filename pairing, GeoTransform Y-axis sign.

### Step 2: MSS Band Standardization (BEFORE pan-sharpen)

**Principle**: Standardize bands before sharpening, not after. Gram-Schmidt alters spectral relationships — post-sharpening detection is unreliable.

```python
# BGRNIR → RGBNIR (GIS standard)
gdal.Translate(output, input, bandList=[3, 2, 1, 4],
               creationOptions=["TILED=YES", "COMPRESS=LZW"])
```

**Why this eliminates band-order bugs**: With MSS in RGBNIR and identity channel mapping (red_channel=1), a known Esri bug (inconsistent output from CreatePansharpenedRasterDataset) cannot produce wrong results — both behavioral branches converge on RGBNIR.

### Step 3: Geometric Registration

**Only working two-step approach**:

```python
# Step A: Clear old transform, compute new (aux.xml)
arcpy.management.RegisterRaster(in_raster=mss, register_mode="RESET")
arcpy.management.RegisterRaster(in_raster=mss, register_mode="REGISTER",
    reference_raster=pan, transformation_type="POLYORDER1")

# Step B: Physical resample to PAN grid
desc = arcpy.Describe(pan)
arcpy.env.outputCoordinateSystem = desc.spatialReference
arcpy.env.cellSize = f"{desc.meanCellWidth} {desc.meanCellHeight}"
arcpy.management.CopyRaster(in_raster=mss, out_rasterdataset=output,
    format="TIFF", pixel_type="16_BIT_UNSIGNED", nodata_value="0")
arcpy.env.outputCoordinateSystem = None; arcpy.env.cellSize = None
```

**Critical**: Always RESET before REGISTER (avoids transform stacking). Set nodata_value="0" explicitly.

#### Method Comparison

| Method | Result | Root Cause |
|--------|--------|------------|
| RegisterRaster alone | No — ghosting | Only writes aux.xml; pan-sharpen reads original pixels |
| ProjectRaster | No — bands corrupted | Produces pink haze |
| GDAL Warp | No — empty output | Does not read aux.xml transforms |
| Resample | No — no registration | Only changes pixel count |
| RegisterRaster + CopyRaster | **Yes** | Two-step physical resampling |

### Step 4: Pan-sharpen

```python
arcpy.management.CreatePansharpenedRasterDataset(
    in_raster=rgbnir_mss,       # Already RGBNIR from Step 2
    red_channel=1, green_channel=2, blue_channel=3, infrared_channel=4,
    out_raster_dataset=output,
    in_panchromatic_image=pan,
    pansharpening_type="Gram-Schmidt",
    red_weight=0.37, green_weight=0.12, blue_weight=0.33, infrared_weight=0.18,
    sensor="",
)
```

**For broadband PAN sensors (~450-900nm)**, higher NIR weights are physically more appropriate:

```python
weights = {"red": 0.25, "green": 0.12, "blue": 0.11, "nir": 0.52}
```

#### Failed Alternatives (11 attempts)

| Category | Issue |
|----------|-------|
| OTB CLI, GDAL Warp, manual GS | RPC coordinate space — cannot unify PAN/MSS geometric space |
| gdalbuildvrt -pansharpen | Actually Brovey algorithm, not Gram-Schmidt |
| OTB Python bindings | DLL conflict with system GDAL |
| NumPy hand-written GS | scipy zoom does not correct geometry; fixed weights cause color cast |

### Step 5: Radiometric Harmonization

#### Critical Distinction

| | Atmospheric Correction | Color Balancing |
|---|---|---|
| Physics | Removes additive atmospheric path radiance | Adjusts display appearance |
| When | **Before fusion**, on raw MSS | After fusion, on output pixels |
| Operation | Subtraction (additive term) | Gain + offset (multiplicative + additive) |

**Key lesson**: Using color-balancing for atmospheric correction will always fail.

#### Option A: Bundle Adjustment (Recommended)

1. Select reference scene (green-band median closest to global median)
2. Sample pixel pairs in overlap zones for all tile pairs
3. Build linear system: g_i × DN_i + o_i = g_j × DN_j + o_j
4. Solve with constraint: g_ref = 1, o_ref = 0
5. Clip gains to [0.85, 1.15]
6. Apply per-tile per-band correction

#### Option B: GDAL cblend (No color balancing)

```bash
gdalbuildvrt mosaic.vrt input*.tif -srcnodata 0 -vrtnodata 0
gdalwarp -of GTiff -ot UInt16 -srcnodata 0 -dstnodata 0 \
  -co COMPRESS=LZW -co TILED=YES -co BIGTIFF=YES \
  -wo CUTLINE_BLEND_DIST=30 -multi mosaic.vrt output.tif
```

#### Failed Methods

| Method | Failure Mode |
|--------|-------------|
| ArcPy HISTOGRAM/DODGING (Pro 2.5) | Known 16-bit multi-band bug (fixed in Pro 3.x) |
| Per-band percentile stretch | Destroys band ratios |
| BFS chain propagation | Error accumulation: 0.8^6=0.26, 1.2^6=2.99 |
| DOS (1% dark target) | Subtracts too much → black output (fix: use 0.01%) |

### Step 6: Mosaic

```bash
gdalbuildvrt mosaic.vrt input*.tif -srcnodata 0 -vrtnodata 0
gdalwarp -of GTiff -ot UInt16 \
  -co COMPRESS=LZW -co TILED=YES -co BIGTIFF=YES \
  -wo CUTLINE_BLEND_DIST=30 -multi mosaic.vrt output.tif
gdaladdo -r average output.tif 2 4 8 16 32 64
```

#### ArcPy Mosaic Dataset Reference

| Component | Recommended | Avoid |
|-----------|------------|-------|
| Seamlines | GEOVENT or VORONOI | DISPARITY |
| Blend width | 20-60px, BOTH | <10px or >100px |
| Color balance | DODGING (no target) | HISTOGRAM |
| Product def | NONE | NATURAL_COLOR_RGBI |
| GDB strategy | Delete + recreate | Incremental |

### Step 7: Quality Control

```python
# Deterministic statistics
for b in range(1, bands + 1):
    stats = ds.GetRasterBand(b).GetStatistics(True, True)

# Quicklook: FIXED 2%-98% stretch (never ArcGIS Pro Percent Clip)
valid = arr[arr > 0]
vmin, vmax = np.percentile(valid, 2), np.percentile(valid, 98)
rgb = np.clip((arr - vmin) / (vmax - vmin), 0, 1)
```

---

## 4. ArcGIS Pro Display Traps

| Symptom | True Cause | Fix |
|---------|------------|-----|
| Blue/washed-out mosaic display | Percent Clip stretch + NoData in statistics | StdDev stretch (n=2.0) |
| GDB preview differs from exported TIFF | DRA: OFF (GDB) vs ON (standalone) | Validate via GDAL quicklook |
| Mosaic darker than source | 16-bit stats polluted by NoData edges | BuildFootprints first |

**Iron rule: ArcGIS Pro rendering ≠ data quality.**

---

## 5. Software Stack

| Component | Version | Purpose | License |
|-----------|---------|---------|---------|
| Python | 3.10+ (NOT ArcGIS bundled 3.6) | Runtime | Open Source |
| GDAL | 3.x (Windows: Gohlke wheel) | Raster I/O | MIT |
| NumPy | 1.x | Numerical | BSD |
| SciPy | 1.9+ | NNLS, optimization | BSD |
| scikit-image | 0.19+ | Histogram matching | BSD |
| ArcGIS Pro | 3.x recommended | Pan-sharpen only | Commercial |

---

## 6. Engineering Principles

1. **Never modify raw data** — each step outputs to a separate directory
2. **Determinism over heuristics** — avoid scene-dependent algorithms
3. **Analyze worst-case accumulation** for multi-hop propagation algorithms
4. **Test on 4 scenes before 46** — validate direction before scaling
5. **Preserve 16-bit throughout** — option to produce separate 8-bit visualization
6. **Declare limitations in deliverables**

---

## 7. Project Structure

```
project/
├── config.yaml
├── data/
│   ├── raw/          (MSS/, PAN/)
│   ├── processed/    (step01_band_std/, step02_registered/, step03_PS/)
│   └── output/       (mosaic.tif)
├── scripts/          (metadata_check, band_standardize, register, pansharpen, mosaic, qc)
├── notebooks/
└── logs/
```

