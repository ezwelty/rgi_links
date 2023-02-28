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

Load RGI6 and RGI7 outlines and write them to parquet for fast read access.

```py
import helpers

rgi6 = helpers.load_rgi6_outlines(path='nsidc0770_00.rgi60.complete.zip')
rgi6.to_parquet('rgi6.parquet')

rgi7 = helpers.load_rgi7_outlines(path='rgi7')
rgi7.to_parquet('rgi7.parquet')
```

Read RGI6 and RGI7 outlines, compute overlaps, and write the results.

```py
import geopandas as gpd
import helpers

rgi7 = gpd.read_parquet('rgi7.parquet', columns=['geometry', 'rgi_id'])

# --- Compute RGI7 self overlaps (~ 95 s)
overlaps = helpers.compute_self_overlaps(rgi7.geometry)
overlaps['i'] = rgi7['rgi_id'].iloc[overlaps['i']].values
overlaps['j'] = rgi7['rgi_id'].iloc[overlaps['j']].values
overlaps.to_parquet('rgi7_self_overlaps.parquet')

# --- Compute RGI7-RGI6 overlaps (~ 605 s)
rgi6 = gpd.read_parquet('rgi6.parquet', columns=['geometry', 'RGIId'])
overlaps = helpers.compute_cross_overlaps(rgi7.geometry, rgi6.geometry)
overlaps['i'] = rgi7['rgi_id'].iloc[overlaps['i']].values
overlaps['j'] = rgi6['RGIId'].iloc[overlaps['j']].values
overlaps.to_parquet('rgi7_rgi6_overlaps.parquet')
```

Resolve RGI7 self overlaps.

```py
import geopandas as gpd
import helpers
import pyproj

overlaps = gpd.read_parquet('rgi7_self_overlaps.parquet')
rgi7 = (
  gpd.read_parquet('rgi7.parquet', columns=['geometry', 'rgi_id'])
  .set_index('rgi_id')
)

# --- Resolve RGI7 self overlaps
resolved = helpers.resolve_self_overlaps(
  overlaps=overlaps,
  geoms=rgi7.geometry,
  min_area=1e5,
  transformer=pyproj.Transformer.from_crs(
    'EPSG:4326', {'proj': 'cea'}, always_xy=True
  )
)
resolved.reset_index(name='geometry').to_parquet('rgi7_fixes.parquet')

# --- Confirm that remaining overlaps are rounding artifacts
rgi7.geometry[resolved.index] = resolved
remaining = helpers.compute_self_overlaps(rgi7.geometry)
assert remaining['area'].lt(1e-16).all()
```


Further inspect RGI7-RGI7 overlaps.

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

# --- Total (380998)
overlaps
# Max area fraction < 0.05 (127936)
overlaps[overlaps['max_area_fraction'].lt(0.05)]

# --- 1:1 RGI7:RGI6 (103840)
is_unique = (
  ~overlaps['i'].duplicated(keep=False) &
  ~overlaps['j'].duplicated(keep=False)
)
overlaps[is_unique].sort_values('i_area_fraction')
# Min area fraction < 0.5 (6221)
overlaps[is_unique & overlaps['min_area_fraction'].lt(0.5)]
# Max area fraction < 0.5 (591)
overlaps[is_unique & overlaps['max_area_fraction'].lt(0.5)]

# --- 1:N RGI7:RGI6 (81192 RGI7)
has_multiple = overlaps.groupby('i')['j'].count().gt(1)
overlaps[overlaps['i'].isin(has_multiple[has_multiple].index)]
# RGI7 area fraction > 0.1 (12586 RGI7)
mask = overlaps['i_area_fraction'].gt(0.1)
has_multiple = overlaps[mask].groupby('i')['j'].count().gt(1)
overlaps[overlaps['i'].isin(has_multiple[has_multiple].index)]

# --- N:1 RGI7:RGI6 (81744 RGI6)
has_multiple = overlaps.groupby('j')['i'].count().gt(1)
overlaps[overlaps['j'].isin(has_multiple[has_multiple].index)]
# RGI6 area fraction > 0.1 (15740 RGI6)
mask = overlaps['j_area_fraction'].gt(0.1)
has_multiple = overlaps[mask].groupby('j')['i'].count().gt(1)
overlaps[overlaps['j'].isin(has_multiple[has_multiple].index)]

# --- 1:0 RGI7:RGI6 (51893)
rgi7_ids = pd.read_parquet('rgi7.parquet', columns=['rgi_id']).iloc[:, 0]
pd.Index(rgi7_ids).difference(overlaps['i'])

# --- 0:1 RGI7:RGI6 (11466)
rgi6_ids = pd.read_parquet('rgi6.parquet', columns=['RGIId']).iloc[:, 0]
pd.Index(rgi6_ids).difference(overlaps['j'])
```
