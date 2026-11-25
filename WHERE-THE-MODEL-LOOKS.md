# See the small change and how the model responds

Look for the cluster of five pinpoints in the enlarged patch. A mammographic calcification can appear as a small bright spot, so the example uses a cluster that is easy to compare at the same scale. Begin with the two zooms, then use the colored tiles to see which changes affected the model's score. [NCI guide to reading a mammogram report](https://www.cancer.gov/types/breast/screening/mammograms)

![Five bright synthetic spots are added to one projection and tested with controlled blur](examples/where-model-looked.png)

1. The turquoise box marks a cluster of five small, bright spots added to the generated CC projection.
2. The two zooms show the exact same patch before and after adding those spots. The MLO projection stays fixed.
3. The last panel blurs one small region at a time and runs the trained **projection-supervised classifier** again. It measures the change in that component's malignancy score.

**Red** means that blurring the region lowered the score. **Blue** means that blurring it raised the score. Adding the dots changed the component's internal score from **−2.758 to −1.502**. The blue tile near the dots is also a measured effect. Blurring that tile changes the spots and the texture around them, so the two interventions can have different effects. [Edited synthetic input](examples/phantom-edited.png) · [All measured probes](examples/region-evidence.json)

The turquoise box is the known edit location. The heatmap measures how the model uses image texture under these changes. Neither is a location annotated by a radiologist. The [prediction walkthrough](WALKTHROUGH.md) explains how this component contributes to the full ensemble.

## Check the strongest local score against held-out lesion masks

I also measured whether the projection-supervised component can point to the lesion on real held-out mammograms. It scans its local scores for benign and malignant masses and calcifications, then picks the strongest grid cell within tissue. The choice uses only the model scores and a tissue mask derived from the input image. An overlap means the selected cell touches at least one pixel in the aligned lesion mask.

| Masked projections | Count | Strongest cell overlaps | Tissue center overlaps | Expected overlap for a random tissue cell |
| --- | ---: | ---: | ---: | ---: |
| All usable projections | 335 | **40.9%** | 6.0% | 3.0% |
| With a mass mask | 330 | **40.9%** | 6.1% | 3.0% |
| With a calcification mask | 5 | 2 of 5 | 0 of 5 | 1.4% |

The 335 projections came from 185 held-out patients. The 95% patient-bootstrap interval for the overall overlap rate was **35.4%–46.9%**. This checks a single location in one ensemble component and uses a coarser grid than the original images. For calcifications, the local archive matched full-size masks for only **5 of 294** linked annotation rows. Five examples are too few to establish how well the model can locate calcifications. The synthetic illustration above measures the change from local blur.

## Reproduce the test

Use the trained models and locked selection from the [main reproduction guide](REPRODUCE.md). Without input flags, the command recreates the same synthetic projections. For a local mammogram, pass `--cc` and optionally `--mlo`. The output stays with your local run.

```powershell
& $p -m medimage.visual_explain --selection "$evaluation/selection.json" --work $experiment `
    --weights $hf --component view-supervised --label malignancy_in_lesion_enriched_cohort `
    --output "$evaluation/region-probe"
& $p -m medimage.mammo_region_audit --root $data --cohort $cohort `
    --selection "$evaluation/selection.json" --work $experiment --weights $hf `
    --component view-supervised --output "$evaluation/region-audit.json"
```
