# Clinical image quality gate

I built a photo quality gate because I wanted to stop a small image classifier from answering when its input had been visibly damaged. I give it a photo, it measures focus, brightness, lost field area and dark-line obstruction, then either accepts it or abstains. When it abstains, it never runs the lesion classifier.

I tested this with public dermoscopic photos, which are close-up skin images taken through a special lens. I added blur, darkening, brightening, lost framing and artificial hair at recorded strengths. I compared simple measurement rules with a small trained gate, then checked how a separate lesion classifier performed on the photos each gate kept or rejected.

**This is a research demonstration, not a diagnostic tool.** An accepted photo is not a promise that a medical answer is correct. The labels describe synthetic damage, not a clinician's judgement of whether a photo is usable.

[Read the saved report with example images](reports/REPORT.md). It opens directly on GitHub without installing anything. I also saved a [standalone HTML report](reports/index.html); download it and open it locally to read it offline.

To try the gate, install [uv](https://docs.astral.sh/uv/getting-started/installation/), clone this repository, and run:

```sh
git clone https://github.com/jackaI1/clinical-image-quality-gate.git
cd clinical-image-quality-gate
uv sync --frozen
uv run image-qc download
uv run image-qc check data/images/ISIC_0024307.jpg
```

I can use `uv run image-qc check /path/to/photo.jpg --method rules` to try the simple checks, or leave out `--method` to use the learned gate. The JSON output includes `accept` or `abstain`, measurements, reasons and a lesion prediction only when accepted. Missing, unreadable or undersized images produce an error; flat images abstain. Both methods use the saved parameters in `reports/models.json`.

## What I found

<!-- BEGIN GENERATED RESULTS -->

I evaluated 1152 photo versions from 72 held-out lesions, using 360 source images in total. Without a gate, the baseline lesion accuracy was 688/1152 (59.7%); on clean held-out images it was 59/72 (81.9%).

| Gate | Damaged rejected | Acceptable retained | Photos accepted | Lesion accuracy: accepted | Lesion accuracy: rejected |
|---|---|---|---|---|---|
| rules | 683/720 (94.9%) | 400/432 (92.6%) | 437/1152 (37.9%) | 308/437 (70.5%) | 380/715 (53.1%) |
| learned | 703/720 (97.6%) | 392/432 (90.7%) | 409/1152 (35.5%) | 280/409 (68.5%) | 408/743 (54.9%) |

These numbers come from `uv run image-qc reproduce` and `uv run python scripts/update_readme.py`; the full counts are in [metrics.json](reports/metrics.json).

<!-- END GENERATED RESULTS -->

I kept the small gate because it catches more of the damage I defined, but it does not give the best accepted-image accuracy. The simple rules are a useful reference and easier to explain. Rejecting more images is not automatically better. The [full report](reports/REPORT.md) breaks this down by damage strength, shows clean-image false rejections and class differences, and gives uncertainty intervals that keep versions of the same lesion together.

## Run the experiment again

```sh
uv run image-qc reproduce --output reports
uv run python scripts/verify_evidence.py --report reports
uv run python scripts/update_readme.py
```

`reproduce` downloads missing originals from ISIC, checks every file against the frozen manifest, trains both classifiers, calibrates the gates, and writes the report. It uses the checked-in source list instead of asking a changing search endpoint for a new sample. I have not substituted generated images for the real-data experiment. The generated images used in unit tests are only test fixtures.

The output contains:

- [REPORT.md](reports/REPORT.md) and [index.html](reports/index.html): readable results and attributed image examples.
- [predictions.csv](reports/predictions.csv): each held-out version, its measurements, gate decisions and downstream prediction.
- [metrics.json](reports/metrics.json): counts, rates, source-bootstrap uncertainty and the acceptance/accuracy threshold curve.
- [models.json](reports/models.json) and [splits.json](reports/splits.json): numeric model parameters, thresholds and source partitions.

## How I built it

I split unique lesion sources before creating any damaged versions. Training, validation and test sets share no lesion IDs or exact image hashes. I did not establish patient independence or rule out visually similar images with different IDs.

The simple gate uses a focus measure based on changes between neighbouring pixels, histogram tails that count very dark or bright pixels, the fraction of the field that is not black, and a thin-dark-line response. I set its limits from acceptable validation images. The trained gate uses logistic regression, a small model that turns the same measurements into a damage score. I choose its cutoff using validation images only. A near-constant-image guard applies to both gates.

My public baseline lesion classifier is [implemented here](src/image_qc/models.py), trained from scratch on clean training images. It uses colour histograms and a coarse spatial colour grid to distinguish the two source diagnoses. It is a deliberately simple comparison model, not an external pretrained model. The benchmark scores rejected images only to measure what was discarded; the normal check command abstains before this classifier is called.

I fix the transformation settings and operational labels in [images.py](src/image_qc/images.py). Clean and mild versions count as acceptable; the stronger versions count as damaged. Framing means shifting part of the original field off the canvas and padding the gap black. This makes the retained field fraction known, but it is an easy proxy for poor framing, not a measurement of how much lesion remains visible. Synthetic hair is dark lines, not a realistic model of every hair pattern. The exposure factors multiply pixel values, not physical camera exposure stops.

## Data and licences

I use the public [ISIC HAM10000 collection](https://api.isic-archive.com/api/v2/collections/212/). The collection specifies [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/), which permits attributed noncommercial use and adaptations. My [frozen manifest](data/manifest.json) records each image's source URL, ISIC ID, public lesion group, diagnosis, checksum, licence and attribution returned by ISIC. I omit demographic fields. Displayed originals are resized and modified as described in the report. The image licence applies separately from my [MIT code licence](LICENSE).

Source citation: Tschandl, P., Rosendahl, C. & Kittler, H., *The HAM10000 dataset, a large collection of multi-source dermatoscopic images of common pigmented skin lesions*, Scientific Data (2018), [doi:10.1038/sdata.2018.161](https://doi.org/10.1038/sdata.2018.161). Collection attribution includes the ViDIR group and the original contributors identified by the archive; per-image attributions are preserved in the manifest and example captions. I retrieve image records through the [official ISIC API](https://api.isic-archive.com/api/docs/swagger/).

This is a deliberately balanced convenience sample of nevus and melanoma records, taking the first eligible unique lesions in each diagnosis from the archive's search order when I froze the manifest. It does not represent diagnosis prevalence or the whole archive. The sample has no private clinical data or access to private systems.

## Limits I would keep in mind

The clean originals can already have defects. Known corruption strength gives a controlled test target, not clinical ground truth. Real photos can have several problems at once, and neither gate has been validated on those, ordinary phone photos, different devices or different skin tones. The downstream classifier is particularly sensitive to framing, including versions labelled mild. It can still make mistakes after the gate accepts an image.

Accepted and rejected groups contain different damage and class mixtures. Their accuracy difference is descriptive, not proof that gating causes better diagnosis. I would need expert quality ratings, patient-level separation, broader data and a stronger independently validated downstream model before drawing conclusions about clinical use.

## Checks and project layout

```sh
uv run pytest -q
uv run ruff check .
uv run ruff format --check .
```

The [CI workflow](.github/workflows/ci.yml) runs the complete test suite and formatting checks, downloads the frozen public sample, reruns the real experiment and verifies its numerical evidence against the saved results. The offline tests exercise controlled damage, provenance, source separation, held-out data isolation, empty groups, checksum failures, safe JSON model export and abstention before lesion inference. Dependency versions and the Python requirement are recorded in `pyproject.toml` and `uv.lock`.

`src/image_qc/` holds the data loader, image transformations, models, experiment, report writer and CLI. `tests/` holds the offline regression tests. `scripts/` checks saved evidence and updates the results above from the generated report. [docs/design.md](docs/design.md) explains the evaluation decisions in more detail.
