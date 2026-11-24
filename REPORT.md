# Experiment report

The selected configuration is `incumbent+adapted-dino-224-0.25` with weights {"incumbent-global": 0.75, "adapted-dino-224": 0.25}.

| Metric | Initial | Selected | 95% paired delta interval |
| --- | ---: | ---: | ---: |
| AUROC | 0.6858 | 0.7468 | [+0.0298, +0.0988] |
| Average precision | 0.6372 | 0.7073 | [+0.0244, +0.1158] |
| Brier score | 0.2180 | 0.1998 | — |
| Expected calibration error | 0.0470 | 0.0729 | — |

The selected AUROC has a patient-cluster 95% CI of [0.6921, 0.7935] on 336 test units. The paired comparison uses 400 patient-cluster bootstrap draws. The test cohort was also used for the initial benchmark. The new model, blend, and calibration were locked before this round's test scoring.

## Validation candidates

| Model or blend | Selection score | AUROC | Average precision |
| --- | ---: | ---: | ---: |
| `incumbent+adapted-dino-224-0.25` | 0.77832 | 0.77832 | 0.74425 |
| `incumbent+adapted-dino-224-0.50` | 0.77174 | 0.77174 | 0.71857 |
| `incumbent+top-two-new` | 0.76517 | 0.76517 | 0.71712 |
| `incumbent+dino-224-0.25` | 0.76481 | 0.76481 | 0.73087 |
| `incumbent+effb0-160-0.25` | 0.75746 | 0.75746 | 0.75308 |
| `incumbent+effb0-224-0.25` | 0.75671 | 0.75671 | 0.75247 |
| `incumbent+effb0-224-0.50` | 0.75650 | 0.75650 | 0.72572 |
| `incumbent+effb0-160-0.50` | 0.75513 | 0.75513 | 0.73298 |
| `incumbent+adapted-dino-224-0.75` | 0.74765 | 0.74765 | 0.69522 |
| `incumbent+dino-224-0.50` | 0.74756 | 0.74756 | 0.69787 |
| `incumbent+r50-224-0.25` | 0.73911 | 0.73911 | 0.73188 |
| `incumbent-global` | 0.73830 | 0.73830 | 0.74218 |
| `incumbent+effb0-224-0.75` | 0.73504 | 0.73504 | 0.68877 |
| `incumbent+effb0-160-0.75` | 0.73325 | 0.73325 | 0.69106 |
| `top-two-new` | 0.73325 | 0.73325 | 0.67913 |
| `incumbent+dino-224-0.75` | 0.72835 | 0.72835 | 0.67312 |
| `adapted-dino-224` | 0.72216 | 0.72216 | 0.67812 |
| `incumbent+r50-224-0.50` | 0.72124 | 0.72124 | 0.70072 |
| `dino-224` | 0.71586 | 0.71586 | 0.65850 |
| `incumbent+r50-224-0.75` | 0.71114 | 0.71114 | 0.68318 |
| `effb0-224` | 0.70671 | 0.70671 | 0.65171 |
| `effb0-160` | 0.70552 | 0.70552 | 0.65730 |
| `r50-224` | 0.70498 | 0.70498 | 0.67017 |

## Search profile winners

| Profile | Epochs per refit | Head | Width | Learning rate | Weight decay | Dropout | Positive-weight exponent |
| --- | ---: | --- | ---: | ---: | ---: | ---: | ---: |
| `effb0-160` | 45 | linear | 0 | 0.003 | 0.015 | 0.12 | 0.5 |
| `r50-224` | 45 | attention | 160 | 0.00048 | 0.003 | 0.35 | 0.25 |
| `effb0-224` | 45 | linear | 0 | 0.003 | 0.015 | 0.12 | 0.5 |
| `dino-224` | 45 | mlp | 256 | 0.00035 | 0.0008 | 0.33 | 0.25 |

## Benchmark breakdown

| Finding or endpoint | Positive units | Initial AUROC | Selected AUROC | Initial AP | Selected AP |
| --- | ---: | ---: | ---: | ---: | ---: |
| malignancy in lesion enriched cohort | 139 | 0.6858 | 0.7468 | 0.6372 | 0.7073 |

| Subgroup | Units | AUROC | Average precision | Brier |
| --- | ---: | ---: | ---: | ---: |
| both views | 251 | 0.7694 | 0.7513 | 0.1873 |
| single view | 85 | 0.6447 | 0.5668 | 0.2368 |
| mass only | 190 | 0.7909 | 0.7432 | 0.1896 |
| calcification only | 146 | 0.6940 | 0.6680 | 0.2131 |

## Search protocol

Each of four encoder and resolution profiles starts with 12 recipes. The resource rungs use [8, 20, 45] epochs. Inner ranking uses 831 fit units and 208 held-out units grouped by patient. Each profile winner is refitted with two seeds on 1,039 units. Final choice and calibration use 260 validation units.

The radiography blend combines logits after correcting for positive-class weighting. The displayed curves use a fixed 101-point grid. Mammography reliability bins appear when they have at least eight samples. Aggregate metrics and trial summaries are in [metrics.json](metrics.json).
