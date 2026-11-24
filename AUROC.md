# High-resolution mammography experiment

![Official benchmark progress](figures/auroc-summary.png)

| Official test metric | Previous model search | First follow-up | High-resolution ensemble |
| --- | ---: | ---: | ---: |
| AUROC | 0.7468 | 0.7748 | **0.7904** |
| Average precision | 0.7073 | 0.7324 | **0.7542** |
| Brier score | 0.1998 | 0.1878 | **0.1819** |

The gain over the previous search was **+0.0435 AUROC**. The paired 95% interval was +0.0020 to +0.0850 across 1,000 patient bootstrap resamples. The high-resolution ensemble's AUROC interval was 0.7347–0.8364. The test comprises 336 breasts from 316 patients. The 0.80 point-estimate target remains open.

## Training and model selection

Training uses 1,039 breasts and 2,105 geometrically aligned mammogram lesion marks. Four benign and malignant morphology combinations teach the spatial head. The original split audit groups by patient and resolves conflicting records and image links. The evaluation takes whole-view pixels and view availability.

The first follow-up selected an equal blend of the previous model, a spatial EfficientNet-B0 and a Mammo-CLIP B5 model. It reached 0.8267 validation AUROC. The high-resolution round added a 1,536-pixel refinement trained with synchronized intensity and geometric augmentation. Validation compared 136 recipes and selected an equal blend of the previous model, Mammo-CLIP and the 1,536-pixel model. It reached 0.8352 AUROC and 0.8299 average precision on the 260 validation breasts.

![Validation AUROC by epoch for four fine-tuned models](figures/high-res-validation.png)

Each round locked its model, blend and calibration before official test scoring. The table presents both follow-up rounds on the same previously used benchmark. The interval describes uncertainty on this cohort.

## Performance by lesion type

| Reference lesion type | Breasts | AUROC |
| --- | ---: | ---: |
| Mass | 190 | 0.8466 |
| Calcification | 146 | 0.7196 |

The lesion categories in this table are evaluation subgroups. The selected classifier takes one or two full projections. The endpoint is malignancy in a lesion-enriched cohort.

## Reproduce and predict

Run the [original workflow](README.md#reproduce-the-experiment) to generate the initial model and cohort. The [shared protocol](https://github.com/ThorsenH1/medimage-ml-lab/blob/a290fb416fe8bce87541efabb15a4a017a362d2e/docs/auroc.md) describes lesion alignment, transfer settings and the validation lock.

```powershell
& $p -m medimage.auc_mammo_data --root $data --cohort artifacts/cache/mammo-full --output "$experiment/pixels/mammo-spatial1536" --height 1536 --workers 6
$parent = "$experiment/runs/mammo-spatial-effb0-768/best.pt"
& $p -m medimage.auc_train mammo --backbone effb0 --cohort artifacts/cache/mammo-full --pixels "$experiment/pixels/mammo-spatial1536" --weights $hf --parent $parent --output "$experiment/runs/mammo-sharp1536" --epochs 6 --batch 2 --accumulate 8 --resolution 1536 --depth 3 --auxiliary 0.38 --lr 0.000055 --head-lr 0.00015 --class-power 0.25 --dropout 0.35 --seed 4513 --block 128 --affine
& $p -m medimage.auc_predict --selection "$experiment/evaluation/mammo-locked-2/selection.json" --work $experiment --previous $search --original artifacts --weights $hf --cc examples/phantom-cc.png --mlo examples/phantom-mlo.png --output prediction.json
```

The [synthetic examples](examples/README.md) give an output for paired CC and MLO projections and another for CC alone.
