# Changes from upstream

Upstream: https://github.com/SnowEx/spicy-snow, tag `0.4.5` at commit 0686fb6a118765e3dc070cc60e15a475a770f1e7 (2026-06-19), by Zach Hoppinen, George Brencher and Ross Palomaki (SnowEx), MIT License (`LICENSE`, copyright 2023 SnowEx). The method is Lievens et al. (2019, 2022) as described in Hoppinen et al. (2024); please cite them and the data sets as the README asks.

This copy starts from upstream's file tree at 0686fb6 without its git history. It is unchanged except for the items below. The retrieval itself (`spicy_snow/processing/`, the A, B and C parameters) and the LICENSE are untouched. `.github/` was left out: it holds the test workflow and the release workflow that publishes to PyPI, which a copy should not do.

## 2026-10-24: Python version in environment.yml

`environment.yml` asked for `python>=3.9` with no upper bound, so a new environment got Python 3.14, while `setup.py` allows only up to 3.12 (and lists 3.9 to 3.11). `pip install .` then stopped with "requires a different Python", and `pip install spicy-snow` quietly installed the old 0.3.1 release. `environment.yml` now asks for `python>=3.9,<3.12`, the versions `setup.py` lists; upstream CI uses 3.10. Results do not change.

`setup.py` still says `>=3.9,<=3.12`, which strictly allows only 3.12.0 among the 3.12 releases. It was left as it is because 3.12 has not been tested here.

## 2026-10-24: dependencies

`tqdm` is imported at module level by `find_data`, `utils.download`, `processing.generate_dataarrays` and `hyp3_pipeline` but was not in `setup.py`; it only arrived through other packages. It is now listed. `environment.yml` also gained `geopandas`, `hyp3_sdk`, `pygeohydro` and `tqdm`, which the code imports, so they come from conda-forge with the rest of the stack instead of from pip. Results do not change.

## 2026-10-24: environment.yml uses conda-forge only

The file listed Anaconda's `defaults` channel after conda-forge. Every package is on conda-forge, and without access to repo.anaconda.com `conda env create` stopped with "HTTP 429 terms of service rate limit exceeded" before solving. `defaults` was removed. Results do not change.

## 2026-10-24: snow depth is labeled in meters

The output NetCDF had `units: %` on `snow_depth`, `snow_index`, `deltaGamma` and `fcf`, plus GeoTIFF tags (`TIFFTAG_*`, `AREA_OR_POINT`) and no-op `scale_factor = 1.0` and `add_offset = 0.0` attributes. They came from the NLCD canopy raster: xarray carries attributes through arithmetic, so the percent label followed `fcf` into the snow index and snow depth. `generate_forest_fraction_dataarray` now drops the source raster's attributes once `fcf` is on the 0 to 1 scale, and `calc_snow_index_to_snow_depth` sets `units: m` and `long_name: snow depth`. The README now calls `fcf` a fraction (0 to 1), not a percentage. A test checks the units.

This is a metadata correction only. Every value in the output is the same as before (see Check).

## 2026-10-24: no development folders on sys.path

`retrieval.py`, `utils/download.py` and `hyp3_pipeline.py` appended `../`, `/Users/zmhoppinen/Documents/spicy-snow` and `~/Documents/spicy-snow` to `sys.path` on import, left over from running from a checkout. With the package installed they do nothing useful and could let an unrelated module next to the working directory be imported. They were removed. Results do not change.

## 2026-10-24: README

How to install from a clone with conda, and which Python versions work. The coverage command now runs the tests with pytest: `unittest discover` collected none of them (0 tests), because they are pytest-style functions. The data citations and links now name the products the code downloads, OPERA RTC-S1 (not CSLC-S1) and VIIRS/JPSS1 VJ110A1F version 2 (not VNP10A1 version 1), with their DOIs from CMR.

## 2026-10-27: an explicit conda lock for Linux

`conda-linux-64.lock` is `environment.yml` solved on linux-64 and written out with `conda list --explicit --md5`: 350 conda-forge packages, each with its URL and MD5, so creating the environment needs no solver step. The solve had become slow and memory-hungry because nothing in `environment.yml` is pinned: on 2026-10-24 it took 116 s, and on 2026-10-27 it took 4 minutes on 4 cores and, in one run, 11 minutes and about 8.7 GB of memory. The README now creates the environment from the lock (`conda create -n spicy --file conda-linux-64.lock`), then runs `pip install .` as before. `environment.yml` stays the readable list of dependencies, with a comment at the top pointing to the lock, and is still the route for other platforms. To refresh the lock after changing `environment.yml`, solve it and run `conda list --explicit --md5 > conda-linux-64.lock` in the new environment. Results do not change.

Checked on 2026-10-27 on Linux x86_64 from a fresh `git clone`: the environment from the lock installed in 31 s (Python 3.11.18, numpy 2.4.6, pandas 3.1.0, xarray 2026.9.0, rasterio 1.4.4 with GDAL 3.12.3); `pip install .` added only spicy-snow and `pip check` found no broken requirements; pytest gave 77 passed and 2 skipped. The Idaho example below took 40 s and gave the same snow depth percentiles (0, 0, 0, 0, 0.31, 1.41, 2.78, 3.84 and 7.91 m), with 685 of 537,682 pixel-times above 5 m, as before.

## Check

On 2026-10-24, macOS on Apple silicon, from a fresh `git clone` of this repository:

```sh
conda env create -f environment.yml      # Python 3.11.17, all packages from conda-forge
conda activate spicy
pip install .                            # pulls nothing more from PyPI
conda install -c conda-forge pytest coverage
python -m coverage run -m pytest tests   # 77 passed, 2 skipped (integration tests, need RUN_INTEGRATION=1)
```

Then the README example over a smaller area around the Hyndman SNOTEL station in the Pioneer Mountains, Idaho:

```python
retrieve_snow_depth(aoi=[-114.24, 43.65, -114.13, 43.75], dates=['2020-10-01', '2021-04-01'],
                    work_dir=work, resolution=100, debug=False, source='opera', outfp=out_nc)
```

It downloads 348 OPERA RTC-S1 files (116 bursts, 1.5 GB) and 73 VIIRS VJ110A1F tiles (about 380 MB) with Earthdata Login through `~/.netrc`, and took 173 s with 3.7 GB peak memory. The same call with upstream 0.4.5 in the same environment gave a NetCDF (85 x 93 pixels, 73 dates, 15 variables) whose every variable and coordinate is identical, value for value, to this copy's. Only the attributes differ, as described above. Snow depth percentiles (min, 1, 5, 25, 50, 75, 95, 99, max) are 0, 0, 0, 0, 0.31, 1.41, 2.78, 3.84 and 7.91 m in both.

The package version comes from the `SPICY_SNOW_VERSION` environment variable at build time (upstream's release workflow sets it from the git tag); without it `setup.py` uses `0.0.0.dev0`. `spicy_snow.__version__` still says `0.3.0`. This copy is tagged `0.4.5.post2` (`0.4.5.post1` before the lock file); to install it with that version, run `SPICY_SNOW_VERSION=0.4.5.post2 pip install .`.
