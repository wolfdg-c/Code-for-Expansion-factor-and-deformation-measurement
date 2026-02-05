# Code-for-Expansion-factor-and-deformation-measurement
#codes are adapted from the following references, please cite them once use our codes: Jurriens, D., van Batenburg, V., Katrukha, E. A. & Kapitein, L. C. in Expansion Microscopy for Cell Biology Vol. 161 (eds. Guichard, P. & Hamel, V.) 105–124 (Academic Press, 2021). Damstra, H. G. et al. GelMap: intrinsic calibration and deformation mapping for expansion microscopy. Nat. Methods 20, 1573–1580 (2023). Zhang, H., Ding, L., Hu, A. et al. TEMI: tissue-expansion mass-spectrometry imaging. Nat Methods 22, 1051–1058 (2025). https://doi.org/10.1038/s41592-025-02664-9.

Tissue Expansion Mass spectrometry Imaging (TEMI) for high spatial resolution multi-omics molecular mapping
Expansion Factor and Deformation Analysis Pipeline
This pipeline quantifies global expansion, local nonuniform deformation, and measurement error as a function of length scale by registering pre-expansion and post-expansion tissue images using manually defined landmarks.

Pipeline Overview
1. Manual landmark selection (BigWarp)
Corresponding landmarks are manually selected between:
	Moving image: pre-expansion tissue
	Fixed/target image: post-expansion tissue
Landmarks are saved as landmarks.csv.
Recommended landmarks
	Distinct, reproducible structural features visible in both images
(e.g., isolated puncta, holes, or unique local features)
	Landmarks should be spatially distributed across the field of view
    While a minimum of ~20 landmarks is typically sufficient for stable registration, the required number of landmarks depends on the image resolution and the spatial scale of features of interest. After landmark placement and registration, users should visually inspect the alignment quality across the entire field of view to confirm that corresponding structures are well aligned down to the resolution scale relevant to their analysis. If local misalignments are observed at the scale of interest, additional landmarks should be added in those regions and the registration repeated until satisfactory alignment is achieved.

2. Global (uniform) model: similarity transform
The Fiji Groovy script bigwarpSimilarityPart.groovy fits a SimilarityModel 2D to the landmark pairs, estimating:
	rotation
	translation
	a single isotropic scale factor
The average expansion factor is derived from the determinant of the fitted transform:
	2D:
avgscale = sqrt(det)
Rationale
The similarity model isolates the best-fit uniform expansion, allowing separation of the overall expansion factor from local nonuniform deformation.

3. Nonrigid model: thin-plate spline (TPS)
A nonrigid transformation is computed from the same landmarks using a Thin Plate Spline (TPS) model.
The Groovy function buildTransform():
	builds the TPS transform
	applies it to a regular grid of sample points
	exports transformed coordinates to CSV

4. Deformation field computation
MATLAB loads:
	coords_similarity.csv – regular grid under the similarity transform
	coords_thin_plate.csv – grid after TPS mapping
The local deformation field is defined as the residual displacement:
diffvect = xySim - xySpline;
This residual captures nonuniform expansion and local warping after removing global scale, rotation, and translation.

5. Measurement error vs. length scale
For all pairs of grid points within the valid region, the pipeline computes:
	Expected distance under uniform expansion
"measLength"=∥x_j^sim-x_i^sim∥

	Distance error due to nonuniform deformation
"measError"=∥d_j-d_i∥

where d_iis the local displacement vector
Measurement errors are then binned by distance, and the mean ± SD is plotted as a function of length scale.

Key Analysis Decisions and Parameter Choices
A. Sampling grid definition
Grid bounds are derived from the fixed (post-expansion) landmarks:
XMax/XMin/YMax/YMin from pointsFixed
dXYstep = round( sqrt(XRange * YRange) / 30 )
Decision
Grid spacing scales with image size, yielding ~30 samples per image dimension.
Why
This maintains comparable spatial sampling across datasets while keeping computational cost manageable.
Tradeoffs
	Smaller dXYstep → higher accuracy but O(N²) runtime increases rapidly
	Larger dXYstep → faster but may under-sample high-frequency deformation
Methods text
“Grid spacing was chosen as √(area)/30 to yield ~30 samples per image dimension, balancing spatial coverage with computational tractability.”

B. Convex-hull landmark mask
Only grid points inside the convex hull of landmarks are analyzed:
in_hull = inpolygon(...)
Why
TPS extrapolates poorly outside landmark support; restricting analysis prevents edge artifacts.
Tradeoff
Peripheral regions are excluded unless landmarks cover them.
Methods text
“Residual deformation was evaluated only within the convex hull of landmarks to avoid TPS extrapolation artifacts.”

C. Definition of deformation
Deformation is defined as:
diffvect = xySim - xySpline;
Interpretation
	Large ‖diffvect‖ → local distortion or nonuniformity

D. TPS inverse computation
When transforming moving → target, the TPS inverse is computed iteratively using:
	invTolerance
	maxIters
Why
TPS inversion is numerical and requires convergence criteria.
Troubleshooting
	NaNs or instability → increase maxIters, relax invTolerance, or improve landmark distribution

E. Physical coordinate scaling
Coordinates are converted from pixels to physical units before transformation:
scaledpt[i] = pt[i] * [sx, sy, sz]
Why
Ensures deformation magnitudes are reported in µm and supports anisotropic voxels.
Methods text
“All coordinates were converted to physical units using voxel sizes (sx, sy, sz) prior to warping.”

F. Pairwise error definition
Measurement error is defined as the difference in displacement vectors, not absolute displacement.
Why
Only relative displacement changes inter-point distances; common-mode shifts do not.

G. Error binning
Measurement errors are binned by distance:
dMeasStep = 10; % µm
Why
Binning reduces noise and yields smooth, interpretable curves.

Summary
This pipeline:
	separates global expansion from local deformation
	avoids TPS extrapolation artifacts
	quantifies how nonuniform expansion affects distance measurements across scales
	keeps computational cost tractable and results interpretable
