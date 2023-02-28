Temporary repository for the Randolph Glacier Inventory (RGI) 7 links team.
In the end, this repository will likely be closed and the code migrated to either [GLIMS-RGI/rgitools](https://github.com/GLIMS-RGI/rgitools) or [GLIMS-RGI/rgi7_scripts](https://github.com/GLIMS-RGI/rgi7_scripts).

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

Read RGI6 and RGI7 outlines, compute overlaps, and write the results to file.

```py
import geopandas as gpd
import helpers

rgi7 = gpd.read_parquet('rgi7.parquet', columns=['geometry', 'rgi_id'])

# Compute RGI7 self overlaps
overlaps = helpers.compute_self_overlaps(rgi7.geometry)
overlaps[['i', 'j']] = rgi7.iloc[overlaps[['i', 'j']]].values
overlaps.to_parquet('rgi7_self_overlaps.parquet')

# Compute RGI7-RGI6 overlaps
rgi6 = gpd.read_parquet('rgi6.parquet', columns=['geometry', 'RGIId'])
overlaps = helpers.compute_cross_overlaps(rgi7.geometry, rgi6.geometry)
overlaps['i'] = rgi7['rgi_id'].iloc[overlaps['i']].values
overlaps['j'] = rgi6['RGIId'].iloc[overlaps['j']].values
overlaps.to_parquet('rgi7_rgi6_overlaps.parquet')
```
