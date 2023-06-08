# natural capital footprint impact
[![License](https://img.shields.io/badge/License-BSD_3--Clause-blue.svg)](https://opensource.org/licenses/BSD-3-Clause)

Company footprint impact workflow (eventually to be made public)

This script calculates metrics of the impact of man-made structures on certain ecosystem services, based on their physical footprint on the landscape.

- Asset: A unit of physical infrastructure that occupies space on the surface of the earth, such as an office, a restaurant, a cell tower, a hospital, a pipeline, or a billboard.
- Footprint: The area on the earth surface taken up by an asset.

Asset location data is usually available as point coordinates. The real footprint of an asset is usually not available. We approximate the footprint by drawing a circle around the point. If actual footprint data is available, you may substitute that in.
Footprint sizes vary widely, but correlate with the type of asset (for example, power plants take up more space than restaurants). We categorize assets using the S&P "facility category" designations.

## Data you must provide
The instructions below assume that you begin with the following information about the assets of interest:
- its coordinate point location
- its category
- its owner

This data should be formatted into a table where each row represents an asset.
Coordinate locations of each asset are in the `latitude` and `longitude` columns.
The `category` column determines footprint size. Other attributes, like the name of the ultimate parent company, may be used to aggregate data.

| latitude | longitude | facility_category | ultimate_parent_name    |
|----------|-----------|-------------------|-------------------------|
| 81.07    | 33.55     | Bank Branch       | XYZ Corp                |
| ...      | ...       | ...               | ...                     |

## Data provided for you

### footprint data by asset category
A table where each row represents an asset category.
The first column is named `category`. The category values will be cross-referenced with the categories in the asset table.
The second column is named `area`. This is the size (in square meters) of footprint to draw for assets of this category.

| facility_category | area |
|-------------------|----------------|
| Bank Branch       | 549.7          |
| ...               | ...            |

This data was derived by manually estimating the footprint area of real assets on satellite imagery. We took the median of a small sample from each category. You may modify or replace this table if you wish to use different data.

### ecosystem service data
Each row represents an ecosystem service.
Columns are:
- `es_id`: An identifier for the ecosystem service
- `es_value_path`: Path to a global raster map of the ecosystem service
- `flag_threshold`: Flagging threshold value for the ecosystem service. Pixels with a value greater than this threshold will be flagged.

| es_id    | es_value_path         | flag_threshold         |
|----------|-----------------------|------------------------|
| sediment | gs://foo-sediment.tif | 123                    |
| ...      | ...                   | ...                    |

You may modify or replace this table if you wish to use different data.

## Installation

```
git clone https://github.com/natcap/natural-capital-footprint-impact.git
conda create -n footprint python gdal
conda activate footprint
pip install .
```
The command `natural-capital-footprint-impact` should now be available.

## Workflow
1. If your asset point data is in CSV format, convert it to a GDAL-supported vector format such as GPKG:
```
ogr2ogr -s_srs EPSG:4326 -t_srs EPSG:4326 -oo X_POSSIBLE_NAMES=longitude -oo Y_POSSIBLE_NAMES=latitude assets.gpkg assets.csv
```

2. If your asset data is in a geographic (non-projected) coordinate system, such as latitude/longitude, reproject it to a projected coordinate system, such as Eckert IV:
```
ogr2ogr -t_srs ESRI:54012 assets_eckert.gpkg assets.gpkg
```
This can also be done in QGIS with the Warp tool, and ArcGIS using the Project tool. Other projected coordinate systems may be used, such as UTM, as long as they are supported by OGR (?? Emily, is this correct?)

3. Run the workflow:
```
natural-capital-footprint-impact -e ECOSYSTEM_SERVICE_TABLE {points,polygons} [--buffer-table BUFFER_TABLE] asset_vector
```

## Modes of operation

```
usage: natural-capital-footprint-impact [-h] -e ECOSYSTEM_SERVICE_TABLE [-b BUFFER_TABLE]
                                        {points,polygons} asset_vector footprint_results_path company_results_path

positional arguments:
  {points,polygons}     mode of operation. in points mode, the asset vector contains point geometries. in polygons mode, it contains polygon geometries.
  asset_vector          path to the asset vector
  footprint_results_path
                        path to write out the asset results vector
  company_results_path  path to write out the aggregated results table

options:
  -h, --help            show this help message and exit
  -e ECOSYSTEM_SERVICE_TABLE, --ecosystem-service-table ECOSYSTEM_SERVICE_TABLE
                        path to the ecosystem service table
  -b BUFFER_TABLE, --buffer-table BUFFER_TABLE
                        buffer points according to values in this table
```


### mode 1
`natural-capital-footprint-impact -e <ecosystem service table path> points <asset point vector path> <output vector path> <output table path>`

In mode 1, you provide the assets as coordinate points. The asset footprint is not known or modeled. Statistics are calculated under each point only.

### mode 2
`natural-capital-footprint-impact -e <ecosystem service table path> points --buffer-table <buffer table path> <asset point vector path> <output vector path> <output table path>`

In mode 2, you provide the assets as coordinate points. The asset footprint is modeled by buffering each point to a distance determined by the asset category. Statistics are calculated under each footprint.

### mode 3
`natural-capital-footprint-impact -e <ecosystem service table path> polygons <asset polygon vector path> <output vector path> <output table path>`

In mode 3, you provide the assets as footprint polygons. This mode is preferred if asset footprint data is available. Statistics are calculated under each footprint.


## Input formats

### point vector
Point data may be provided in a [GDAL-supported vector format](https://gdal.org/drivers/vector/index.html). All points must be in the first layer. All features in the layer must be of the `Point` type. `MultiPoint`s are not allowed. No attributes are required. Any attributes that there are will be preserved in the output.

### polygon vector
Polygon data may be provided in a [GDAL-supported vector format](https://gdal.org/drivers/vector/index.html). All polygons must be in the first layer. All features in the layer must be of the `Polygon` or `MultiPolygon` type. Any attributes that there are will be preserved in the output.

## Output formats

The ecosystem services are:
- `coastal_risk_reduction_service`
- `nitrogen_retention_service`
- `sediment_retention_service`
- `nature_access`
- `endemic_biodiversity`
- `redlist_species`
- `species_richness`
- `kba`

### CSV and point vector
Mode 1 produces a CSV and a point vector in geopackage (.gpkg) format. Both contain the same data. These are copies of the input data with additional columns added. There is one column added for each ecosystem service. This column contains the ecosystem service value at each point, or `NULL` if there is no data available at that location.

### polygon vector
Modes 2 and 3 produce a polygon vector in geopackage (.gpkg) format. It is a copy of the input vector with additional columns added to the attribute table. There is one column added for each combination of ecosystem service and statistic.

