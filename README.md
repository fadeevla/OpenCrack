# OpenCrack

A consolidated, leakage-controlled benchmark for pavement-crack segmentation, in COCO format.

OpenCrack unifies dozens of separately-annotated public crack datasets into a single benchmark with a
shared four-class taxonomy, cross-split deduplication, and a fixed stratified split. It is released in
two regimes:

- **OpenCrack**: the in-domain, close-up crack imagery (asphalt, concrete, masonry, facades, bridge
  decks). This is the primary pixel-mask benchmark.
- **PaveSafe**: an out-of-domain deployment benchmark drawn from operational road-inspection systems
  with wider acquisition geometry, retaining non-crack distress classes (potholes, patches, manholes).

The two-regime design separates *in-domain accuracy* from *cross-regime transfer*, and the
consolidation removes the train/test image leakage that inflates scores on the large composite
crack datasets.

This repository is the benchmark's public reference page: the dataset roster, the model-comparison
results, and full source citations. See [Citing this work](#citing-this-work) to reference the
benchmark.

---

## At a glance

| | Images (dedup) | Annotations (dedup) | Images (full) | Annotations (full) |
|---|---:|---:|---:|---:|
| **OpenCrack** (in-domain) | 82,218 | 99,493 | 104,930 | 129,116 |
| **PaveSafe** (out-of-domain) | 48,770 | 105,583 | 50,522 | 110,987 |

- **38 public source datasets** consolidated: 32 into OpenCrack (under 28 COCO `dataset_source`
  labels, six nested in composite archives) and 6 into PaveSafe.
- **Mixed annotation granularity**: pixel masks, bounding boxes, and confirmed-negative (crack-free)
  sources, unified in a COCO instance format. Segmentation metrics are computed on the pixel-mask
  subset.
- **Confirmed-negative pool of 11,229 crack-free images** (8,034 from SDNET2018 uncracked splits +
  3,195 from the Bridge Crack Library noncrack set), reserved for false-positive-rate evaluation.
- **Deduplication** with DINOv2 image embeddings at a conservative cosine threshold (τ = 0.979), so no
  near-duplicate image straddles the train/validation/test boundary.
- **Stratified 70/15/15 split** on source dataset, crack presence, and crack-width band.

---

## Four-class taxonomy

The taxonomy is anchored to engineering inspection standards (ASTM/PCI, GOST), so the labels map to
the distress families that road authorities track and repair.

| ID | Class | Definition |
|---:|---|---|
| 1 | `material_discontinuity` | Cracks: surface fractures from exceeded tensile strength (longitudinal, transverse, diagonal, alligator, block, map-cracking), and sealed/repaired cracks |
| 2 | `surface_openings` | Structural material loss: potholes, spalling, voids where the surface is missing |
| 3 | `patching_and_repairs` | Prior maintenance interventions: surface and full-depth patches |
| 4 | `functional_infrastructure` | Non-distress structural components that generate false positives: manhole covers, drainage grates |

Sealed and repaired cracks are merged into class 1; the source label is preserved in an
`original_class` field on every annotation. OpenCrack populates mainly classes 1–2; classes 3–4 are
populated by PaveSafe's multi-class sources.

---

## Results: model comparison

Four models are evaluated on exactly the same held-out OpenCrack stratified test cohort (6,910
crack-pixel-annotated positives), with one harness and 1,000-resample bootstrap confidence intervals.
The compared models are three released baselines (**CrackSAM-adapter** [11], an adapter on Segment
Anything; **OmniCrack30k** [4], an nnU-Net trained on a prior composite; and **Hybrid-Segmentor**
[12], a CNN-transformer) and an **nnU-Net** [14] trained on the OpenCrack training split.

### In-domain (OpenCrack)

IoU and clIoU(τ=4) as image-weighted (micro) means over the 6,910 positives, with the per-source
(macro) mean alongside:

| Model | IoU micro [95% CI] | IoU macro | clIoU(τ=4) micro [95% CI] | clIoU macro |
|---|---:|---:|---:|---:|
| **nnU-Net (trained on OpenCrack)** | **0.539 [0.534, 0.545]** | **0.470** | **0.641 [0.634, 0.648]** | **0.691** |
| CrackSAM-adapter [11] | 0.470 [0.463, 0.476] | 0.391 | 0.606 [0.598, 0.613] | 0.669 |
| OmniCrack30k [4] | 0.426 [0.420, 0.433] | 0.384 | 0.557 [0.548, 0.565] | 0.633 |
| Hybrid-Segmentor [12] | 0.402 [0.395, 0.409] | 0.265 | 0.468 [0.460, 0.476] | 0.379 |

The trained nnU-Net leads on both metrics with confidence-interval-disjoint margins. The comparison
against **OmniCrack30k** is the informative one: it is the *same* nnU-Net v2 framework, architecture,
and 500-epoch budget, trained on a prior composite instead of OpenCrack. The **+0.11 IoU** gap
between them (and +0.058 clIoU macro, 0.691 vs 0.633) therefore measures the value of in-domain
training on the consolidated benchmark, with architecture held fixed.

### Full five-metric matrix (OpenCrack, n = 6,910)

The ranking is identical across all five per-image metrics, so no conclusion depends on the metric
choice.

| Model | IoU | Dice | Boundary F1 | NSD | clIoU(τ=4) |
|---|---:|---:|---:|---:|---:|
| **nnU-Net (this work)** | **0.539 [0.534, 0.545]** | **0.664 [0.658, 0.669]** | **0.608 [0.601, 0.615]** | **0.700 [0.694, 0.707]** | **0.641 [0.634, 0.648]** |
| CrackSAM-adapter [11] | 0.470 [0.463, 0.476] | 0.586 [0.578, 0.592] | 0.523 [0.516, 0.531] | 0.631 [0.623, 0.638] | 0.606 [0.598, 0.613] |
| OmniCrack30k [4] | 0.426 [0.420, 0.433] | 0.532 [0.524, 0.539] | 0.504 [0.496, 0.513] | 0.587 [0.578, 0.597] | 0.557 [0.548, 0.565] |
| Hybrid-Segmentor [12] | 0.402 [0.395, 0.409] | 0.493 [0.485, 0.501] | 0.463 [0.456, 0.471] | 0.537 [0.529, 0.545] | 0.468 [0.460, 0.476] |

### Cross-regime transfer (PaveSafe, n = 730)

Models trained or fine-tuned on close-up crack imagery do not transfer to PaveSafe's wider road
scenes: the in-domain leader falls from 0.539 to 0.142 IoU, and every crack model collapses into a
narrow 0.09–0.14 band. The ranking is regime-dependent, not a fixed ordering of model quality.

| Model | IoU | Dice | Boundary F1 | NSD | clIoU(τ=4) |
|---|---:|---:|---:|---:|---:|
| nnU-Net (this work) | 0.142 [0.128, 0.155] | 0.203 [0.184, 0.221] | 0.206 [0.189, 0.223] | 0.257 [0.236, 0.278] | 0.201 [0.182, 0.221] |
| OmniCrack30k [4] | 0.124 [0.110, 0.138] | 0.176 [0.157, 0.193] | 0.209 [0.189, 0.230] | 0.241 [0.219, 0.264] | 0.192 [0.173, 0.212] |
| CrackSAM-adapter [11] | 0.097 [0.084, 0.110] | 0.137 [0.120, 0.152] | 0.050 [0.043, 0.058] | 0.077 [0.066, 0.089] | 0.132 [0.117, 0.147] |
| Hybrid-Segmentor [12] | 0.090 [0.077, 0.104] | 0.126 [0.109, 0.143] | 0.048 [0.041, 0.054] | 0.072 [0.062, 0.081] | 0.053 [0.045, 0.061] |

Per-source breakdowns (by crack width and surface material), the hard-negative enrichment study, and
the reproduction of published baseline numbers are part of the full evaluation accompanying this
benchmark.

---

## The trained model

The supervised baseline is the self-configuring **nnU-Net v2** (2D configuration) [14], trained from
random initialisation on the OpenCrack class-1 training split:

- Plain convolutional U-Net selected by the nnU-Net planner (seven stages, feature widths
  `[32, 64, 128, 256, 512, 512, 512]`), InstanceNorm + LeakyReLU, **≈92.5 M parameters** (counted
  from the released checkpoint).
- 256×256 patches, per-image z-normalisation, nnU-Net default augmentation; SGD default schedule.
- A **fixed training budget of about 500 epochs**, single fold (fold 0), no early stopping and no
  multi-fold ensembling, matched to the single-fold OmniCrack30k so the head-to-head isolates
  training data from architecture.

**Weights:** the trained checkpoint and planner-generated configuration are on Hugging Face:
[`fadeevla/opencrack-nnunet`](https://huggingface.co/fadeevla/opencrack-nnunet).

---

## Data and reproduction

The source datasets are **not redistributed** in this repository: each retains its own license and
must be obtained from its original provider (see [References](#references)). What is published here is
the consolidated benchmark's specification, roster, and results.

The consolidation pipeline (taxonomy mapping, mask-to-RLE conversion, DINOv2 deduplication, stratified
splitting) and the unified evaluation harness are maintained in the project's development repository.

---

## OpenCrack source datasets

32 source datasets under 28 COCO `dataset_source` labels (six are nested inside composite archives).
Image counts are for the deduplicated build. All sources provide a pixel mask per image **except**
CCIC and SDNET2018, which are image-level crack/non-crack sources and supply most of the
confirmed-negative pool.

| Source | Images | Source | Images |
|---|---:|---|---:|
| CCIC [22] | 17,763 | DeepCrack [20] | 365 |
| SDNET2018 [9] | 17,170 | Facade390 † | 364 |
| CrackVision12K [12] | 9,484 | BuildCrack [26] | 354 |
| Conglomerate [5] | 7,834 | Masonry [8] | 239 |
| Bridge Crack Library [16] | 7,745 | CrackLS315 [20] | 194 |
| TopoDS [23] | 7,047 | CrackSC † | 168 |
| CRACK500 [30] | 3,280 | CrackNJ156 [29] | 155 |
| Concrete3k [19] | 2,891 | CrackTree260 [31] | 150 |
| Asphalt3k [19] | 2,273 | Ceramic [17] | 100 |
| UDTIRI-Crack [13] | 2,229 | AEL [1] | 57 |
| DIC [25] | 530 | CRKWH100 [20] | 44 |
| S2DS [3] | 497 | Khanh11k † | 41 |
| SegCODEBRIM [15] | 441 | CrackSeg9k [18] | 7 |
| Road420 † | 415 | GAPS384 [10] | 381 |

† Dataset release with no associated publication; distributed as data only (Road420, Facade390,
CrackSC, Khanh11k).

## PaveSafe source datasets

| Dataset | Images | Annotation | Surface | Geography |
|---|---:|---|---|---|
| RDD2022 [2] | 28,000 | Bounding box (4 classes) | Asphalt road | 6 countries |
| SVRDD [24] | 7,999 | Bounding box (7 classes) | Asphalt road | China |
| Mosquitonet / MAPSIA [6] | 6,424 | Bounding box (13 classes) | Road | Spain |
| PavementScapes [27] | 3,305 | Pixel mask | Asphalt (top-down) | China |
| CrackSeg [28] | 2,366 | Pixel mask | Asphalt (oblique) | Mixed |
| UDTIRI pothole [13] | 676 | Pixel mask (pothole) | Road | China |

---

## Citing this work

If you use OpenCrack, please cite this work and credit each source dataset you rely on (the
references below).

```bibtex
@misc{fadeev2026opencrack,
  author       = {Fadeev, V. A.},
  title        = {OpenCrack: A Consolidated, Leakage-Controlled Benchmark for Pavement Crack Segmentation},
  year         = {2026},
  howpublished = {\url{https://github.com/fadeevla/opencrack}}
}
```

---

## License and attribution

The contents of this repository (the README, the roster, the compiled results, and the presentation)
are released under **CC-BY-4.0**. This license covers the benchmark specification and documentation
**only**; it does **not** relicense the underlying images or annotations. Every source dataset
remains under its own original license; obtain each from its provider and comply with its terms.

---

## References

1. Amhaz, R. Automatic Crack Detection on Two-Dimensional Pavement Images: An Algorithm Based on Minimal Path Selection / R. Amhaz, S. Chambon, J. Idier, V. Baltazart // IEEE Transactions on Intelligent Transportation Systems. — 2016. — Vol. 17, no. 10. — P. 2718–2729. — DOI 10.1109/TITS.2015.2477675.
2. Arya, D. RDD2022: A multi-national image dataset for automatic road damage detection / D. Arya, H. Maeda, S. K. Ghosh, D. Toshniwal, Y. Sekimoto // Geoscience Data Journal. — 2024. — DOI 10.1002/gdj3.260.
3. Benz, C. Image-Based Detection of Structural Defects Using Hierarchical Multi-scale Attention / C. Benz, V. Rodehorst // Pattern Recognition: 44th DAGM German Conference (DAGM GCPR 2022). — Springer, 2022. — P. 337–353. — DOI 10.1007/978-3-031-16788-1_21.
4. Benz, C. OmniCrack30k: A Benchmark for Crack Segmentation and the Reasonable Effectiveness of Transfer Learning / C. Benz, V. Rodehorst // 2024 IEEE/CVF CVPR Workshops (CVPRW). — IEEE, 2024. — P. 3876–3886. — DOI 10.1109/CVPRW63382.2024.00392.
5. Bianchi, E. Concrete Crack Conglomerate Dataset / E. Bianchi, M. Hebdon. — 2021. — DOI 10.7294/16625056.
6. Cano-Ortiz, S. An end-to-end computer vision system based on deep learning for pavement distress detection and quantification / S. Cano-Ortiz, L. Lloret Iglesias, P. Martinez Ruiz Del Árbol, P. Lastra-González, D. Castro-Fresno // Construction and Building Materials. — 2024. — Vol. 416. — Art. 135036. — DOI 10.1016/j.conbuildmat.2024.135036.
7. Carion, N. SAM 3: Segment Anything with Concepts / N. Carion [et al.]. — Meta AI, 2025.
8. Dais, D. Automatic crack classification and segmentation on masonry surfaces using convolutional neural networks and transfer learning / D. Dais, İ. E. Bal, E. Smyrou, V. Sarhosis // Automation in Construction. — 2021. — Vol. 125. — Art. 103606. — DOI 10.1016/j.autcon.2021.103606.
9. Dorafshan, S. SDNET2018: An annotated image dataset for non-contact concrete crack detection using deep convolutional neural networks / S. Dorafshan, R. J. Thomas, M. Maguire // Data in Brief. — 2018. — Vol. 21. — P. 1664–1668. — DOI 10.1016/j.dib.2018.11.015.
10. Eisenbach, M. How to get pavement distress detection ready for deep learning? A systematic approach / M. Eisenbach, R. Stricker, D. Seichter [et al.] // 2017 International Joint Conference on Neural Networks (IJCNN). — IEEE, 2017. — P. 2039–2047. — DOI 10.1109/IJCNN.2017.7966101.
11. Ge, K. Fine-tuning vision foundation model for crack segmentation in civil infrastructures / K. Ge, C. Wang, Y. Guo, Y. Tang, Z. Hu, H. Chen // arXiv. — 2024. — arXiv:2312.04233.
12. Goo, J. M. Hybrid-Segmentor: A Hybrid Approach to Automated Fine-Grained Crack Segmentation in Civil Infrastructure / J. M. Goo, X. Milidonis, A. Artusi, J. Boehm, C. Ciliberto // Automation in Construction. — 2025. — Vol. 170. — Art. 105960. — DOI 10.1016/j.autcon.2024.105960.
13. Guo, S. UDTIRI: An Online Open-Source Intelligent Road Inspection Benchmark Suite / S. Guo, J. Li, Y. Feng [et al.] // IEEE Transactions on Intelligent Transportation Systems. — 2024. — Vol. 25, no. 8. — P. 9920–9931. — DOI 10.1109/TITS.2024.3351209.
14. Isensee, F. nnU-Net: a self-configuring method for deep learning-based biomedical image segmentation / F. Isensee, P. F. Jaeger, S. A. A. Kohl, J. Petersen, K. H. Maier-Hein // Nature Methods. — 2021. — Vol. 18, no. 2. — P. 203–211. — DOI 10.1038/s41592-020-01008-z.
15. Jaziri, A. SegCODEBRIM / A. Jaziri, M. Mundt, A. F. Rodriguez, V. Ramesh. — 2023. — DOI 10.5281/zenodo.10071534.
16. Jin, T. Bridge Crack Library / T. Jin, Z. Li, Y. Ding, S. Ma, Y. Ou. — 2021. — DOI 10.7910/DVN/RURXSH.
17. Junior, G. Ceramic Cracks Segmentation with Deep Learning / G. Junior, J. Ferreira, C. Millán-Arias, R. Daniel, A. Casado, B. Fernandes // Applied Sciences. — 2021. — Vol. 11. — Art. 6017. — DOI 10.3390/app11136017.
18. Kulkarni, S. CrackSeg9k: A Collection and Benchmark for Crack Segmentation Datasets and Frameworks / S. Kulkarni, S. Singh, D. Balakrishnan, S. Sharma, S. Devunuri, S. C. R. Korlapati // arXiv. — 2022. — arXiv:2208.13054.
19. Li, Y. Real-time High-Resolution Neural Network with Semantic Guidance for Crack Segmentation / Y. Li, R. Ma, H. Liu, G. Cheng // Automation in Construction. — 2023. — Vol. 156. — Art. 105112. — DOI 10.1016/j.autcon.2023.105112.
20. Liu, Y. DeepCrack: A deep hierarchical feature learning architecture for crack segmentation / Y. Liu, J. Yao, X. Lu, R. Xie, L. Li // Neurocomputing. — 2019. — Vol. 338. — P. 139–153. — DOI 10.1016/j.neucom.2019.01.036.
21. Oquab, M. DINOv2: Learning Robust Visual Features without Supervision / M. Oquab, T. Darcet, T. Moutakanni [et al.] // arXiv. — 2023. — arXiv:2304.07193.
22. Özgenel, Ç. F. Performance Comparison of Pretrained Convolutional Neural Networks on Crack Detection in Buildings / Ç. F. Özgenel, A. G. Sorguç // 35th International Symposium on Automation and Robotics in Construction (ISARC). — 2018. — DOI 10.22260/ISARC2018/0094.
23. Pantoja-Rosero, B. G. TOPO-Loss for continuity-preserving crack detection using deep learning / B. G. Pantoja-Rosero, D. Oner, M. Kozinski [et al.] // Construction and Building Materials. — 2022. — Vol. 344. — Art. 128264. — DOI 10.1016/j.conbuildmat.2022.128264.
24. Ren, M. An annotated street view image dataset for automated road damage detection / M. Ren, X. Zhang, X. Zhi, Y. Wei, Z. Feng // Scientific Data. — 2024. — Vol. 11, no. 1. — Art. 407. — DOI 10.1038/s41597-024-03263-7.
25. Rezaie, A. Comparison of Crack Segmentation Using Digital Image Correlation Measurements and Deep Learning / A. Rezaie, R. Achanta, M. Godio, K. Beyer // Construction and Building Materials. — 2020. — Vol. 261. — Art. 120474. — DOI 10.1016/j.conbuildmat.2020.120474.
26. Srivastava, K. CrackUDA: Incremental Unsupervised Domain Adaptation for Improved Crack Segmentation in Civil Structures / K. Srivastava, D. Kancharla, R. Tahereen, P. Ramancharla, S. R. K. Sarvadevabhatla, H. Kandath // arXiv. — 2024. — arXiv:2412.15637.
27. Tong, Z. Pavementscapes: a large-scale hierarchical image dataset for asphalt pavement damage segmentation / Z. Tong, T. Ma, J. Huyan, W. Zhang // arXiv. — 2022. — arXiv:2208.00775.
28. TRMetaGroup. CrackSeg : dataset. — 2025. — URL: https://github.com/TRMetaGroup/CrackSeg.
29. Xu, Z. Pavement crack detection from CCD images with a locally enhanced transformer network / Z. Xu, H. Guan, J. Kang [et al.] // International Journal of Applied Earth Observation and Geoinformation. — 2022. — Vol. 110. — Art. 102825. — DOI 10.1016/j.jag.2022.102825.
30. Yang, F. Feature Pyramid and Hierarchical Boosting Network for Pavement Crack Detection / F. Yang, L. Zhang, S. Yu, D. Prokhorov, X. Mei, H. Ling // IEEE Transactions on Intelligent Transportation Systems. — 2020. — Vol. 21, no. 4. — P. 1525–1535. — DOI 10.1109/TITS.2019.2910595.
31. Zou, Q. CrackTree: Automatic crack detection from pavement images / Q. Zou, Y. Cao, Q. Li, Q. Mao, S. Wang // Pattern Recognition Letters. — 2012. — Vol. 33, no. 3. — P. 227–238. — DOI 10.1016/j.patrec.2011.11.004.
