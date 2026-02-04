# cell-painting-exercise
 Take a parquet file with image embedding data from cell painting (Bray et al, 2016)  images of pooled primary human hepatocytes treated with perturbations for 48 hours. 

# Column name information:

## Metadata (plate/well identifiers)
* well_id: position on a 382-well plate (ex. "A01", "H12")
* plate: unique plate identifier
* source: Batch ID for the screen (technical batch) - plates processed together
* batch: another batch grouping (possibly sub-batches within sources)
* library: which compound library the molecule came from (oasis)

## Perturbation
* compound_id: Name/ID of drug. "DMSO" = vehicle control (no drug)
* compound_concentration_um: Dose in micromolar (uM)
* compound_target: What protein(s) the drug hits (multi-label, ex "EGFR; HER2")
* compound_pathway: What biological pathway(s) it affects (ex. "Apoptosis; Cell Cycle")
* compound_biological_activity: free-test description from vendor

## Assay readouts (measurements)
* mtt_ridge_norm: Viability assay. 1 = healthy/viable, 0 = dead. Measures metabolic activity (MT-Glo assay). "Ride norm" means it's been normalized to remove well-position effects
* ldh_ridge_norm: Cytotoxicity assay. 0 = healthy, 1 = max toxicity. Measures lactate dehydrogenase release (cells leak this enzyme when they die)

## Image derived features (from microscopy):

Comes from segmenting cell painting images and measuring morphology:

### Nuclei features:
* mean_nuclei_count: Average # nuclei per FOV. Fewer nuclei = cells died or didn't grow
* mean_nuclei_count_ridge_norm: Same but normalized to remove well-position effects
* mean_nuclei_count_ridge_norm_inv: Inverse transformed (1/normalized count)
* mean_nuclei_area_mean: Average area of each nucleus (larger = cell stress or polyploidy)
* mean_nuclei_area_mean_ridge_norm_inv: Inverse of normalized version

### Cytoplasm features:
* cyto_area_ratio: Ratio of cytoplasm area to total cell area (cells can swell/shrink under stress)
* cyto_area_ratio_ridge_norm: Normalized version

### Organelle features:
* vacuole_sum: Total vacuole area (vacuoles = stress/autophagy markers)
* vacuole_ratio: Fraction of cell occupied by vacuoles
* mito_puncta_ratio: Fraction of mitochondria that are "punctate" (fragmented/small). Fission = stress/apoptosis
* mito_puncta_ratio_ridge_norm_sub: Normalized by subtracting well-position effects
* ros_stress_granules_sum_mean_norm_ridge_norm_corr: Reactive oxygen species/stress granules (markers of oxidative stress). Normalized.

## Embeddings:
* brightfield: 768-dimensional DINOv2 embedding from brightfield channel only (gray scaled, no flourescent stains). Shape: (N, 768). Raw, un-normalized.
* pca_embedding_raw: DINOv2 embeddings from all 5 flourescent channels (nucleus, ER, RNA, actin/Golgi, mitochondria) concatenated end-to-end (5 x 768 = 3,840 dims), then PCA reduced to 128 dims. PCA was fit using first 10 sources - Shape (N, 128). Not normalized for batch effects.
* pca_embedding_normalize: Same as pca_embedding_raw but batch-corrected per source using DMSO controls
(raw - mean_dmso_in_source) / std_dmso_in_source

## Misc:
* bioactivity_svc_tvn_pca_platenorm_coral_regnorm_all_n_components_256: A pre-computed feature vector of 256 dims from a different embedding approach