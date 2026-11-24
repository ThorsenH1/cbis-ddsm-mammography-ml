# Reproduce the selected mammography ensemble

Run the [original workflow](README.md#reproduce-the-experiment) from this repository. Keep its Python, dataset and Hugging Face cache paths in `$p`, `$data` and `$hf`. The following commands use the default `artifacts` and `search-artifacts` directories. Use a local volume with sufficient space for both pixel caches and the model checkpoints.

The transfer runs consume two files from the Mammo-CLIP authors. The original B5 file is `Pre-trained-checkpoints/b5-model-best-epoch-7.tar` with SHA-256 `0baccb0cc19cd6093ff8252f7c2b7a6efbc32973498640abcb311210c46fd02c`. The downstream archive `Downstream_evalualtion_b5_fold0.zip` contains `classification/Models.zip`, whose member `Models/Classifier/fine_tune/cancer/upmc_breast_clip_det_b5_period_n_ft_seed_10_fold0_best_aucroc_ver084.pth` has SHA-256 `743bfaef035b5997c0d70bd7964edc3f29d5a3a099ff96a4386539a426b71727`. Training verifies these file hashes and imports the image encoder. The authors release the weights under CC BY-NC-SA 4.0. [Pretraining files](https://huggingface.co/shawn24/Mammo-CLIP/tree/main/Pre-trained-checkpoints) · [Downstream files](https://huggingface.co/shawn24/Mammo-CLIP/tree/main/Downstream-checkpoints) · [Author training script](https://github.com/batmanlab/Mammo-CLIP/blob/main/src/scripts/classifier_fine_tune_b5.sh)

## Prepare pixels and train the candidates

```powershell
$PSNativeCommandUseErrorActionPreference = $true
$ErrorActionPreference = 'Stop'
$experiment = 'D:/experiments/mammography-discrimination'
$mammoClip = 'D:/weights/b5-model-best-epoch-7.tar'
$rsnaEncoder = 'D:/weights/upmc_breast_clip_det_b5_period_n_ft_seed_10_fold0_best_aucroc_ver084.pth'
$search = 'search-artifacts'
$cohort = 'artifacts/cache/mammo-full'
$pixels1024 = "$experiment/pixels/mammo-spatial-v2"
$pixels1536 = "$experiment/pixels/mammo-spatial1536"
$runs = "$experiment/runs"

& $p -m medimage.auc_mammo_data --root $data --cohort $cohort --output $pixels1024 --height 1024 --workers 8
& $p -m medimage.auc_mammo_data --root $data --cohort $cohort --output $pixels1536 --height 1536 --workers 6

& $p -m medimage.auc_train mammo --cohort $cohort --backbone effb0 --pixels $pixels1024 --weights $hf `
    --output "$runs/mammo-spatial-effb0-768" --epochs 12 --batch 8 --accumulate 2 --resolution 768 --depth 3 `
    --auxiliary 0.55 --lr 0.000115 --head-lr 0.00055 --class-power 0.25

& $p -m medimage.auc_train mammo --cohort $cohort --backbone mammo-clip --pixels $pixels1024 --weights $mammoClip `
    --output "$runs/mammo-clip-b5-768" --epochs 10 --batch 6 --accumulate 3 --resolution 768 --depth 2 `
    --auxiliary 0.30 --lr 0.000055 --head-lr 0.00036 --class-power 0.25 --dropout 0.40

& $p -m medimage.auc_train mammo --cohort $cohort --backbone effb0 --pixels $pixels1536 --weights $hf `
    --parent "$runs/mammo-spatial-effb0-768/best.pt" --output "$runs/mammo-sharp1536" `
    --epochs 6 --batch 2 --accumulate 8 --resolution 1536 --depth 3 --auxiliary 0.38 --lr 0.000055 `
    --head-lr 0.00015 --class-power 0.25 --dropout 0.35 --seed 4513 --block 128 --affine

& $p -m medimage.auc_mammo_mil --cohort $cohort --pixels $pixels1024 --weights $hf `
    --output "$runs/mammo-mil" --patch-epochs 7 --epochs 15

& $p -m medimage.auc_train mammo --cohort $cohort --backbone effb0 --pixels $pixels1536 --weights $hf `
    --parent "$runs/mammo-sharp1536/best.pt" --output "$runs/mammo-specialist1536" `
    --epochs 6 --batch 2 --accumulate 8 --resolution 1536 --depth 3 --auxiliary 0.34 --lr 0.000040 `
    --head-lr 0.00019 --class-power 0.25 --dropout 0.40 --seed 4797 --block 128 --affine --specialize

& $p -m medimage.auc_mammo_texture --cohort $cohort --pixels $pixels1536 `
    --output "$runs/mammo-texture" --epochs 35 --workers 4

& $p -m medimage.auc_tta --cohort $cohort --pixels $pixels1536 --weights $hf `
    --parent "$runs/mammo-sharp1536" --output "$runs/mammo-sharp1536-tta"

& $p -m medimage.auc_train mammo --cohort $cohort --backbone mammo-clip-rsna --pixels $pixels1536 `
    --weights $rsnaEncoder --output "$runs/mammo-rsna-transfer" `
    --epochs 8 --batch 4 --accumulate 4 --resolution 1024 --depth 3 --auxiliary 0.28 --lr 0.000075 `
    --head-lr 0.00042 --class-power 0.24 --dropout 0.43 --seed 4981 --block 128 --affine

& $p -m medimage.auc_train mammo --cohort $cohort --backbone effb0 --pixels $pixels1536 --weights $hf `
    --parent "$runs/mammo-sharp1536/best.pt" --output "$runs/mammo-view-supervised" `
    --epochs 6 --batch 3 --accumulate 5 --resolution 1280 --depth 3 --auxiliary 0.29 --lr 0.000045 `
    --head-lr 0.00016 --class-power 0.21 --dropout 0.40 --seed 5003 --block 126 --affine --view-supervision
```

## Lock on validation, then run the official test

The lock includes the previous ensemble from the original workflow. It stores the selected blend, validation calibrators, and file hashes in `selection.json`. The test command checks those hashes and writes aggregate metrics and a local prediction array. GPU and library versions can affect the result of retraining. [Shared protocol](https://github.com/ThorsenH1/medimage-ml-lab/blob/bbf538c89c6b686ccbfb5a202ae887ef20628d8a/docs/auroc.md)

```powershell
$evaluation = "$experiment/evaluation/mammo-locked-4"
$paths = @('--work', $experiment, '--previous', $search, '--cohort', $cohort, '--output', $evaluation)
& $p -m medimage.auc_selection lock mammo @paths --pairwise --stable-blend --crossfit-only `
    --candidate 'mammo-clip=runs/mammo-clip-b5-768@mammo-spatial-v2' `
    --candidate 'sharp1536=runs/mammo-sharp1536@mammo-spatial1536' `
    --candidate 'patch-mil=runs/mammo-mil@mammo-spatial-v2' `
    --candidate 'specialist1536=runs/mammo-specialist1536@mammo-spatial1536' `
    --candidate 'calc-texture=runs/mammo-texture@mammo-spatial1536' `
    --candidate 'sharp1536-tta=runs/mammo-sharp1536-tta@mammo-spatial1536' `
    --candidate 'rsna-transfer=runs/mammo-rsna-transfer@mammo-spatial1536' `
    --candidate 'view-supervised=runs/mammo-view-supervised@mammo-spatial1536'

& $p -m medimage.auc_selection test mammo @paths --weights $hf --bootstrap 1000
& $p -m medimage.auc_report --output $evaluation --previous $search
```

## Run the synthetic inputs through the final model

```powershell
$predict = @('--selection', "$evaluation/selection.json", '--work', $experiment,
    '--previous', $search, '--original', 'artifacts', '--weights', $hf)
& $p -m medimage.auc_predict @predict --cc examples/phantom-cc.png --mlo examples/phantom-mlo.png --output "$evaluation/phantom-paired.json"
& $p -m medimage.auc_predict @predict --cc examples/phantom-cc.png --output "$evaluation/phantom-cc-only.json"
```

[Example outputs](examples/README.md) · [Official benchmark report](AUROC.md)
