# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

specutils is an Astropy-affiliated package providing shared Python representations of astronomical spectra (`Spectrum`, `SpectrumList`, `SpectrumCollection`, `SpectralRegion`) plus analysis, fitting, manipulation, and file I/O tools. Python >= 3.11; core deps are numpy, scipy, astropy, gwcs, asdf/asdf-astropy, and ndcube.

## Commands

```sh
pip install -e '.[test]'           # dev install (add ',jwst' for stdatamodels-based JWST loaders)

pytest                              # runs specutils/ and docs/ (testpaths in setup.cfg)
pytest specutils/tests/test_regions.py
pytest specutils/tests/test_regions.py -k test_invert
pytest --remote-data=any            # also run tests that download files (tests marked with remote_access / remote_data)

tox -e codestyle                    # flake8 (config in setup.cfg [flake8], max line length 100)
tox -e py314-test                   # isolated run as CI does; other factors: -devdeps, -oldestdeps, -predeps, -cov
```

Pytest config (in `setup.cfg`) that matters:
- `filterwarnings = error` — any new, unfiltered warning fails a test. Use `pytest.warns` or add a targeted filter rather than letting warnings escape.
- `--doctest-rst` and `doctest_plus` are on, so the `.rst` files in `docs/` and docstring examples are executed as tests.
- `xfail_strict = true`.
- ASDF schemas in `specutils/io/asdf/schemas` are validated as tests.

CI (`.github/workflows/ci_workflows.yml`) covers Python 3.11–3.14. `oldestdeps` runs on the oldest Python using the pins in `tox.ini`, and `devdeps` (nightly wheels) runs on Python 3.15 as an allowed failure. Job names describe roles (oldest/intermediate/current/latest Python) rather than versions, so a Python bump only changes the `python`/`toxenv` values.

## Architecture

### Core data classes (`specutils/spectra/`)
- `Spectrum` (`spectrum.py`) subclasses `ndcube.NDCube` plus astropy `NDIOMixin`/`NDArithmeticMixin`, with `OneDSpectrumMixin` and `RedshiftMixin` from `spectrum_mixin.py`. It can be N-dimensional; `spectral_axis_index` tells it which axis is spectral, and `move_spectral_axis` can reorder it on construction. The spectral axis comes from either an explicit `spectral_axis` (bin centers or bin edges) or a WCS (FITS WCS or GWCS). Much of the constructor logic is reconciling these inputs.
- `Spectrum1D` is a deprecated alias of `Spectrum` (renamed in 2.0). New code and docs should use `Spectrum`.
- `SpectralAxis` (`spectral_axis.py`) subclasses astropy `SpectralCoord`.
- `SpectrumList` is a plain `list` subclass that multi-spectrum loaders return. It supports lazy loading, where spectra are read on access through a loader callable with an LRU cache (`is_lazy`, `cache_size`).
- `SpectrumCollection` holds several same-shape spectra as one set of arrays.
- `SpectralRegion` / `CompoundSpectralRegion` (`spectral_region.py`) define spectral sub-ranges. Analysis functions accept them.

### I/O (`specutils/io/`)
- Readers are registered with astropy's unified I/O registry through the `@data_loader(label, identifier=..., dtype=..., extensions=..., priority=...)` decorator in `io/registers.py` (writers use `@custom_writer`). Users call `Spectrum.read(path, format=label)`. Without `format`, the identifier functions pick the loader, and `priority` breaks ties.
- A `data_loader` for `dtype=Spectrum` automatically registers a matching `SpectrumList` reader as well, unless `autogenerate_spectrumlist=False`. For `SpectrumList` loaders, `lazy_loader=` provides the lazy path, which is triggered by `SpectrumList.read(..., lazy_load=True)`.
- Every module in `io/default_loaders/` is imported automatically (its `__init__.py` globs `*.py`), so adding a loader only means adding a file there. The loader table in the docs is generated from the registry (`io/_list_of_loaders.py`).
- Identifier functions that raise are caught and treated as "not this format", which makes identifier bugs silent. Pass `verbose=True` to debug them.
- ASDF serialization: `io/asdf/` (converters, schemas, and manifests registered via entry points in `setup.cfg`).

### Analysis / manipulation / fitting
- `analysis/`: scalar measurements (line flux, centroid, FWHM, moments, SNR, correlation, template comparison). Most take `(spectrum, regions=None, ...)` and send the region handling through `analysis.utils.computation_wrapper`, which returns one result per region when given a list of regions.
- `manipulation/`: resamplers (a `ResamplerBase` subclass with `FluxConserving`/`LinearInterpolated`/`SplineInterpolated` implementations), smoothing, region extraction, uncertainty estimation, and model replacement.
- `fitting/`: continuum and line fitting on top of `astropy.modeling`. `utils/quantity_model.py` wraps models so they work with Quantities.

## Tests
- Unit tests live in `specutils/tests/`; loader-specific tests live in `specutils/io/default_loaders/tests/` and `specutils/io/asdf/tests/`.
- Shared fixtures (e.g. `simulated_spectra`, `spectral_examples`) are in `specutils/conftest.py`.
- Tests that need downloaded data use `remote_access([{'id': <zenodo id>, 'filename': ...}])` from `specutils/tests/conftest.py`. They only run with `--remote-data`.

## Changelog
Every PR adds an entry to `CHANGES.rst` under the unreleased version, in the right section (New Features / Bug Fixes / Other Changes and Additions), ending with `[#PR]`.
