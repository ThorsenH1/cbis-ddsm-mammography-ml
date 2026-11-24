# Improving the mammography model

The current selected ensemble reached **0.8050 breast-level AUROC**. Mass cases reached **0.8602** and calcification cases reached **0.7341**. Calcifications can occupy only a small part of an image. Resizing an entire breast can make their margins and distribution harder to see. [Current benchmark](AUROC.md)

## What a full-resolution external model contributed

GMIC combines a whole-image model with a second branch that reads proposed 256 × 256 pixel regions. Its layout lets it see fine detail without running the entire image through the expensive patch model. The pilot tested the first published checkpoint on 467 whole CC or MLO projections from 260 validation breasts. We aligned the chest wall, preserved the aspect ratio, and used image-wide standardization. Each view used a 2,944 × 1,920 pixel canvas. [GMIC source and weights](https://github.com/nyukat/GMIC) · [Published method](https://arxiv.org/abs/2002.07613)

| Pooling rule for the external checkpoint | Validation AUROC | Average precision |
| --- | ---: | ---: |
| Average of available view logits | 0.5864 | 0.5823 |
| Maximum over available view logits | 0.5615 | 0.5741 |

This pilot measures transfer from one published checkpoint to this cohort with the stated preparation. The low score motivated a detector trained on our own development images. The official test was unused in this pilot.

## Learn small details from project training data

The next experiment uses a different patch pipeline. The detector reads a tissue crop, plus two versions that emphasize differences from the local average. Small fields of view preserve the shape of calcifications. Larger fields cover the structure of masses. Training uses lesion annotations from training patients to teach the detector. At prediction time a dense scan of the two whole views proposes candidate patches.

The breast-level classifier learns to combine patch evidence across views. Every training breast receives proposed patches from a detector that excluded its patient group. Validation compares the scoring model after fitting from those proposals. Selection needs to compare both the overall AUROC and the calcification subgroup on the frozen cohort. [Current evaluation protocol](AUROC.md#training-and-model-selection)
