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

The CMMD cache starts from 5,202 DICOM files. We group images by patient and breast side and keep standard CC and MLO projections. A fixed patient split produced 1,242 training breasts, 275 validation breasts and 350 test breasts. The duplicate audit excluded five breasts from patients linked by repeated images. The CBIS-only EfficientNet starts from the checkpoint used in the main project. The CMMD-adapted version starts there too, then learns from CMMD training patients. CMMD validation selected epoch 6 of 7.

| Configuration on the same 350 CMMD breasts from 336 test patients | AUROC | 95% patient bootstrap interval | Average precision |
| --- | ---: | ---: | ---: |
| CBIS-only, external evaluation | 0.7271 | 0.6669–0.7862 | 0.8826 |
| CMMD-adapted, held-out patients from the adaptation dataset | **0.8415** | **0.7974–0.8807** | **0.9404** |

The paired change in AUROC was +0.1143 with a 95% interval of +0.0614 to +0.1703. This compares complete configurations at input heights of 1,536 and 1,280 pixels. The test set contains 260 malignant and 90 benign breasts, which gives average precision a prevalence baseline of 0.7429. The project's separate official CBIS-DDSM benchmark is reported above.

### Does CMMD adaptation improve the CBIS-DDSM model?

I started from the CMMD-adapted checkpoint and fitted it again on CBIS-DDSM training breasts at an input height of 1,536 pixels. The new branch reached **0.7785 validation AUROC**. A search across 675 configurations chose the final blend on patient-grouped validation. Its validation AUROC was **0.8332**, and its CMMD branch weight was **2.24%**. The complete model was locked before running the official CBIS-DDSM test.

| Metric on the same 336 breasts from 316 test patients | Published ensemble | CMMD-transfer blend |
| --- | ---: | ---: |
| AUROC | **0.8050** | 0.7882 |
| Average precision | **0.7762** | 0.7574 |
| Brier score, lower is better | **0.1745** | 0.1846 |
| Mass AUROC | **0.8602** | 0.8435 |
| Calcification AUROC | **0.7341** | 0.7174 |

The blend's 95% patient-bootstrap AUROC interval was **0.7359–0.8330**. The published CBIS-DDSM model remains the model with 0.8050 AUROC. The held-out CMMD gain above measures adaptation to CMMD patients. This experiment measures the result of transferring the adapted model back to CBIS-DDSM.

Using matched resamples of the same 316 test patients, the AUROC change was **−0.0168**, with a paired 95% interval of **−0.0349 to +0.0004**. Average precision changed by **−0.0188**, with a paired interval of **−0.0371 to −0.0013**.

### Reproduce the CMMD comparison

Download the DICOM release and its clinical spreadsheet from TCIA. Keep the `$p`, `$hf`, `$experiment` and `$runs` variables and the trained sharp checkpoint from the [main reproduction guide](REPRODUCE.md).

```powershell
$cmmdDicom = 'D:/data/CMMD/dicom'
$cmmdLabels = 'D:/data/CMMD/clinical.xlsx'
$cmmdCache = "$experiment/cmmd/cache1536"
& $p -m medimage.mammo_cmmd --dicom $cmmdDicom --labels $cmmdLabels --output $cmmdCache `
    --height 1536 --workers 5 --expected 5202
& $p -m medimage.auc_train mammo --cohort $cmmdCache --pixels $cmmdCache --backbone effb0 --weights $hf `
    --parent "$runs/mammo-sharp1536/best.pt" --domain-parent --output "$runs/cmmd-effb0" `
    --epochs 7 --batch 3 --accumulate 5 --resolution 1280 --depth 3 --lr 0.000065 --head-lr 0.00024 `
    --class-power 0.23 --dropout 0.38 --seed 5333
& $p -m medimage.mammo_cmmd_evaluate --cohort $cmmdCache --work $experiment --weights $hf `
    --output "$experiment/cmmd/evaluation"
& $p -m medimage.auc_train mammo --cohort $cohort --pixels $pixels1536 --backbone effb0 --weights $hf `
    --parent "$runs/cmmd-effb0/best.pt" --domain-parent --output "$runs/mammo-cmmd-transfer" `
    --epochs 6 --batch 2 --accumulate 8 --resolution 1536 --depth 3 --auxiliary 0.34 `
    --lr 0.000042 --head-lr 0.00015 --class-power 0.24 --dropout 0.38 --seed 5365 --block 128 --affine
```

Extend the [published validation search](REPRODUCE.md#lock-on-validation-then-run-the-official-test) with the fine-texture candidate above and `cmmd-transfer=runs/mammo-cmmd-transfer@mammo-spatial1536`. Use a new evaluation directory and keep the patient-grouped calibration and blend settings.

## Keeping small grayscale changes from the original images

The current model reads 8-bit JPEGs. The public archive also has the original mammograms. In a pilot, I matched 24 training CC views to their DICOM originals and checked the same annotated regions in both versions. The files had the same image dimensions and orientation in all 24 pairs. All originals stored 16-bit pixels.

| Median within the annotated region | Original DICOM | JPEG |
| --- | ---: | ---: |
| Distinct gray levels in 12 mass views | 949 | 80 |
| Distinct gray levels in 12 calcification views | 1,067.5 | 97 |

After matching the brightness scale, the median local texture correlation was 0.791 for mass regions and 0.811 for calcification regions. The pilot measures differences in the image signal. The breast-level evaluation measures whether the extra signal improves predictions. [Original CBIS-DDSM archive](https://www.cancerimagingarchive.net/collection/cbis-ddsm/) · [NCI Imaging Data Commons](https://datacommons.cancer.gov/repository/imaging-data-commons)

### Reproduce the high-precision candidate

The complete native-precision cache contains **1,635 breasts**. I verified the file sizes for all **2,897 original DICOM projections** before processing. The registry assigned 1,843 projections to training, 467 to validation, and 587 to the official test using the existing split. The preparation command keeps native precision through alignment and resizing, and adds a smaller contrast-enhanced component for compatibility with the starting model. It matched **1,007 of 1,007** training mass annotations and **1,101 of 1,102** training calcification annotations. One calcification annotation had no unambiguous match. The raw data require around 96 GB and the 16-bit cache requires around 10 GB.

I verified the full reader, training and prediction path on four original projections from the project's training partition. Running inference directly from the two DICOM files used for smoke validation reproduced both cached logits. Full-cohort performance is measured separately on the published CBIS-DDSM split.

```powershell
& $p -m pip install 'idc-index==0.13.5' 'idc-index-data==25.0.5'
$nativeRoot = "$experiment/native-mammograms"
$dicomPlan = "$nativeRoot/registry.json"
$nativeDicom = "$nativeRoot/dicom"
$nativePixels = "$experiment/pixels/mammo-native1536"
& $p -m medimage.mammo_native_registry plan --cohort $cohort --project-root $data --plan $dicomPlan
& $p -m medimage.mammo_native_registry download --plan $dicomPlan --destination $nativeDicom
& $p -m medimage.mammo_native --cohort $cohort --plan $dicomPlan --dicom $nativeDicom `
    --project-root $data --output $nativePixels --height 1536 --workers 4
& $p -m medimage.auc_train mammo --cohort $cohort --pixels $nativePixels --backbone effb0 --weights $hf `
    --parent "$runs/mammo-view-supervised/best.pt" --output "$runs/mammo-native1536" --view-supervision `
    --epochs 6 --batch 2 --accumulate 8 --resolution 1536 --depth 3 --auxiliary 0.31 `
    --lr 0.000040 --head-lr 0.000145 --class-power 0.21 --dropout 0.38 --seed 5701 --block 112 --affine
```

Add `native1536=runs/mammo-native1536@mammo-native1536` to the validation candidate list. The locked test command reads the matching original DICOM projections for this component. To reproduce a mixed-format prediction from files, supply the matching DICOM projections through `--native-cc` and `--native-mlo`, along with the JPEG projections required by the other components.
