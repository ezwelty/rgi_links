# `rgi_links`

Temporary repository for the Randolph Glacier Inventory (RGI) 7 "links team".
In the end, the code in this repository will likely be merged into either [GLIMS-RGI/rgitools](https://github.com/GLIMS-RGI/rgitools) or [GLIMS-RGI/rgi7_scripts](https://github.com/GLIMS-RGI/rgi7_scripts).

## Installation

```sh
git clone https://github.com/ezwelty/rgi_links
```

## Example usage

Create the Python environment described in [`environment.yaml`](/environment.yaml) and open a Python console.

```sh
conda env create -f environment.yaml
conda activate rgi
python
```

### Prepare RGI outlines

Load RGI6 and RGI7 outlines and write them to parquet for fast read access.

```py
import helpers

rgi6 = helpers.load_rgi6_outlines(path='nsidc0770_00.rgi60.complete.zip')
rgi6.to_parquet('rgi6.parquet')

rgi7 = helpers.load_rgi7_outlines(path='rgi7')
rgi7.to_parquet('rgi7.parquet')
```

### Compute RGI7-RGI6 overlaps

Read RGI6 and RGI7 outlines, compute overlaps, and write the results.
Overlap polygons and the area fractions are computed with geographic coordinates
(computing the latter with projected coordinates had maximum 0.4% error),
but the overlap areas are computed in square meters with projected coordinates.

Overlaps smaller than 200 m<sup>2</sup> are dropped.
See https://github.com/ezwelty/rgi_links/issues/6 for how this number was chosen.

```py
import geopandas as gpd
import numpy as np
import scipy.interpolate
import helpers

rgi6 = gpd.read_parquet('rgi6.parquet', columns=['geometry', 'RGIId'])
rgi7 = gpd.read_parquet('rgi7.parquet', columns=['geometry', 'rgi_id'])

# --- Compute RGI7-RGI6 overlaps (~ 400 s)
overlaps = helpers.compute_cross_overlaps(rgi7.geometry, rgi6.geometry)
overlaps['i'] = rgi7['rgi_id'].iloc[overlaps['i']].values
overlaps['j'] = rgi6['RGIId'].iloc[overlaps['j']].values

# ---- Calculate area
equal_area_crs = {'proj': 'cea'}
overlaps['area'] = overlaps['geometry'].to_crs(equal_area_crs).area

# ---- Filter by minimum area
overlaps = overlaps[overlaps['area'] > 200]

# ---- Calculate relationships
# Count number of direct relatives (i.e. 1:1, n:1, 1:n, n:n)
overlaps['in'], overlaps['jn'] = helpers.count_pair_relations(
  overlaps['i'], overlaps['j']
)
# Label clusters of (directly and indirectly-related) pairs
overlaps['cluster'] = helpers.label_pair_clusters(overlaps['i'], overlaps['j'])
overlaps.to_parquet('rgi7_rgi6_overlaps.parquet')
```

### Inspect RGI7-RGI6 overlaps

```py
import numpy as np
import pandas as pd

# --- Read overlaps without geometries
overlaps = pd.read_parquet(
  'rgi7_rgi6_overlaps.parquet',
  columns=['i', 'j', 'i_area_fraction', 'j_area_fraction', 'area']
)

# --- Compute min and max area fraction
overlaps['min_area_fraction'] = (
  overlaps[['i_area_fraction', 'j_area_fraction']].min(axis=1)
)
overlaps['max_area_fraction'] = (
  overlaps[['i_area_fraction', 'j_area_fraction']].max(axis=1)
)

# --- Total
overlaps
# Max area fraction < 0.05
overlaps[overlaps['max_area_fraction'].lt(0.05)]

# --- 1:1 RGI7:RGI6
is_unique = overlaps['in'].eq(1) & overlaps['jn'].eq(1)
# Min area fraction < 0.5
overlaps[is_unique & overlaps['min_area_fraction'].lt(0.5)]
# Max area fraction < 0.5
overlaps[is_unique & overlaps['max_area_fraction'].lt(0.5)]

# --- 1:N RGI7:RGI6
has_multiple = overlaps.groupby('i')['j'].count().gt(1)
overlaps[overlaps['i'].isin(has_multiple[has_multiple].index)]
# RGI7 area fraction > 0.1
mask = overlaps['i_area_fraction'].gt(0.1)
has_multiple = overlaps[mask].groupby('i')['j'].count().gt(1)
overlaps[overlaps['i'].isin(has_multiple[has_multiple].index)]

# --- N:1 RGI7:RGI6
has_multiple = overlaps.groupby('j')['i'].count().gt(1)
overlaps[overlaps['j'].isin(has_multiple[has_multiple].index)]
# RGI6 area fraction > 0.1
mask = overlaps['j_area_fraction'].gt(0.1)
has_multiple = overlaps[mask].groupby('j')['i'].count().gt(1)
overlaps[overlaps['j'].isin(has_multiple[has_multiple].index)]

# --- 1:0 RGI7:RGI6
rgi7_ids = pd.read_parquet('rgi7.parquet', columns=['rgi_id'])['rgi_id']
pd.Index(rgi7_ids).difference(overlaps['i'])

# --- 0:1 RGI7:RGI6
rgi6_ids = pd.read_parquet('rgi6.parquet', columns=['RGIId'])['RGIId']
pd.Index(rgi6_ids).difference(overlaps['j'])
```

### Compute and fix RGI7 self-overlaps

```py
import geopandas as gpd
import helpers
import pyproj

rgi7 = gpd.read_parquet('rgi7.parquet', columns=['geometry', 'rgi_id'])
equal_area_crs = {'proj': 'cea'}

# --- Compute RGI7 self overlaps (~ 100 s)
overlaps = helpers.compute_self_overlaps(rgi7.geometry)
overlaps['area'] = overlaps['geometry'].to_crs(equal_area_crs).area
overlaps['i'] = rgi7['rgi_id'].iloc[overlaps['i']].values
overlaps['j'] = rgi7['rgi_id'].iloc[overlaps['j']].values
overlaps.to_parquet('rgi7_self_overlaps.parquet')

# --- Resolve RGI7 self overlaps
rgi7 = rgi7.set_index('rgi_id')
resolved = helpers.resolve_self_overlaps(
  overlaps=overlaps,
  geoms=rgi7.geometry,
  min_area=9500,
  transformer=pyproj.Transformer.from_crs(
    'EPSG:4326', {'proj': 'cea'}, always_xy=True
  )
)
resolved.reset_index(name='geometry').to_parquet('rgi7_fixes.parquet')

# --- Confirm that remaining overlaps are rounding artifacts
rgi7.geometry[resolved.index] = resolved
remaining = helpers.compute_self_overlaps(rgi7.geometry)
assert remaining['geometry'].to_crs({'proj': 'cea'}).area.lt(1e-6).all()
```
