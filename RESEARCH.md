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

The detector reads a tissue crop, plus two versions that emphasize differences from the local average. Small fields of view preserve the shape of calcifications. Larger fields cover the structure of masses. Training uses lesion annotations from training patients to teach the detector. At prediction time a dense scan of the two whole views proposes candidate patches.

The breast-level classifier learns to combine patch evidence across views. Every training breast receives proposed patches from a detector that excluded its patient group. The strongest standalone version reached **0.7964 validation AUROC**. The blend search compared **524 recipes** and selected a new version with **0.8350 cross-fitted validation AUROC**.

| Same official test cohort | Published eight-model ensemble | New fine-texture candidate |
| --- | ---: | ---: |
| Breast-level AUROC | **0.8050** | 0.7882 |
| Average precision | **0.7762** | 0.7578 |
| Mass AUROC | **0.8602** | 0.8440 |
| Calcification AUROC | **0.7341** | 0.7180 |

The change was **−0.0168 AUROC**, with a paired 95% interval of **−0.0345 to +0.0002**. This experiment gives a concrete example of a higher validation score that fails to transfer to the familiar test cohort. The published ensemble remains the one described in the [main report](AUROC.md).

### Reproduce the fine-texture candidate

Use the same `$p`, `$cohort`, `$pixels1536`, `$hf` and `$runs` as the [main training guide](REPRODUCE.md).

```powershell
& $p -m medimage.auc_mammo_mil --cohort $cohort --pixels $pixels1536 --weights $hf `
    --output "$runs/mammo-fine-texture" --patch-epochs 8 --epochs 18 --seed 5183 --fine-texture
```

Add the candidate `fine-texture-mil=runs/mammo-fine-texture@mammo-spatial1536` to the existing candidate list. Write the new validation lock in another evaluation directory before scoring the official test.

## A second source of breast images

The Chinese Mammography Database contains digital mammograms and biopsy-confirmed labels. It gives the experiment a way to learn from a second source and to reserve separate patients for evaluation. The data use different acquisition equipment from CBIS-DDSM. Transfer between the two is a research question. [CMMD data and citation](https://doi.org/10.7937/tcia.eqde-4b16)
