---
mediawiki: SPIM_Registration_Method
title: SPIM Registration Method
extensions: ["mathjax"]
---

## Citation

Please note that the SPIM registration plugin available through Fiji, is based on a publication. If you use it successfully for your research please be so kind to cite our work:

-   S. Preibisch, S. Saalfeld, J. Schindelin and P. Tomancak (2010) "Software for bead-based registration of selective plane illumination microscopy data", *Nature Methods*, **7**(6):418-419. [Webpage](http://www.nature.com/nmeth/journal/v7/n6/full/nmeth0610-418.html) [PDF](/media/nmeth0610-418.pdf) [Supplement](/media/nmeth0610-418-s1.pdf)

## Important Note

<span style="color:#A52A2A"> ***For details about the SPIM registration, fusion & deconvolution please have a look at the [Multiview Reconstruction Plugin](/plugins/multiview-reconstruction). It is much more powerful, flexible and completely integrated with the [BigDataViewer](/plugins/bdv). Documentation on the outdated [SPIM Registration](/plugins/spim-registration) is still available.*** </span>

## Introduction

### SPIM principles

{% include thumbnail src='/media/plugins/spim-registration/spimscheme.png' title='<b>Figure 1:</b> Schematic drawing of a Selective Plane Illumination Microscope'%} A Selective Plane Illumination Microscope[^1] ([Figure 1](/media/plugins/spim-registration/spimscheme.png)), achieves optical sectioning by focusing the excitation laser into a thin laser light sheet that reaches its minimal thickness in the middle of the field of view. The light sheet enters the water filled sample chamber and illuminates the sample which is embedded in an agarose column. The agarose protrudes from the end of a glass capillary attached to a motor that rotates the sample. The objective lens is arranged perpendicular to the light sheet. The perpendicular orientation of the illumination and detection optics ensures that only a section of the specimen in-focus is illuminated, minimizing photo-bleaching and laser damage of the living samples and allowing for very long time-lapse recordings. Two-dimensional images of emitted fluorescent light are captured by a CCD camera focussed on the center of the light-sheet. The CCD camera captures the light-sheet-illuminated section in a single exposure enabling a very fast acquisition rate important for capturing dynamic developmental events. In order to acquire 3d image stacks, the sample is moved through the light sheet in increments of 0.5 μm to 5 μm depending on the objective and the light sheet thickness.

The SPIM instrument can, in principle, achieve an isotropic, high resolution along $$x$$, $$y$$ and $$z$$-axis allowing *in toto* imaging of large 3d specimens. In order to achieve an isotropic resolution uniformly across the sample volume in all three dimensions, it is necessary to rotate the sample and record image stacks of the same specimen from different angles (usually 3 to 12, see [Video 1](/media/plugins/spim-registration/supplementary-video-1-spim-in-action.mov)).

### Related work

Multi-view microscopy techniques, such as tilted confocal acquisitions,[^2] SPIM[^3] and Ultramicroscopy[^4] (2 views), can increase the resolution along the $$z$$-axis and thus enable the analysis of objects smaller than the axial resolution limit.[^5][^6] Image reconstruction based on the image intensities requires significant overlap of image content which is often difficult to achieve particularly in live imaging of dynamically changing samples.

The idea to incorporate fiduciary markers to facilitate sample independent reconstruction is widely used in medical imaging[^7][^8][^9][^10][^11] and electron tomography.[^12][^13] Due to the low amount of fiduciary markers available for registration, research is focused on error analysis rather than efficiency of matching of thousands of markers with only partial overlap.[^14]

In contrast, in the robotics and automation field, there is interest in localization of large amounts of different objects. Points of interest are extracted from photos and checked against databases to determine their type and orientation.[^15][^16] To enable real time object recognition, Lamdan et al.[^17] introduced 'geometric hashing' which uses an intrinsic invariant local coordinate system to match objects against database entries in a viewpoint independent manner. The geometric hashing principle is reused in the fields of astronomy[^18] and protein structure alignment and comparison[^19][^20][^21][^22] where efficient searching in massive point clouds is required.

The use of local descriptors instead of complete scenes for matching is proposed in many fields comprising image registration,[^23][^24] robotics and autonomous systems,[^25][^26] and computer vision.[^27]

Matula et al.[^28] suggest segmentation based approaches for reconstruction of multi-view microscopy images. The center of mass of the cloud of segmented objects is used as a reference point for a cylindrical coordinate system facilitating the registration between two views. Similarly to intensity based approaches, this method requires significant overlap between the images and furthermore supports alignment of only two stacks at a time.

Our approach combines the idea of using fiduciary markers, local descriptors and geometric hashing and applies global optimization. It can register an arbitrary number of partially overlapping point clouds. It is robust with respect to the amount of incorporated beads, bead distribution, amount of overlap, and can reliably detect non-affine disturbances (e.g. abrupt agarose movement) that might occur during imaging ([Table 1](#Table1)).

## The Method

### Bead Segmentation

The incorporated sub-resolution beads appear as the Point Spread Function (PSF) of the microscopic system in each image $$I(x,y,z)$$. To detect beads, one would ideally convolve it with the impulse response (PSF) of the microscope yielding highest correlation at the sub-pixel location of each bead. However, the PSF is not constant over different experiments due to changing exposure times, laser power, bead types, objectives and agarose concentration. Furthermore, the PSF is not constant across the field of view due to the concavity of the light sheet and thus the convolution operation is computationally very demanding.

We found that an appropriately smoothed 3d LaPlace filter $$\nabla^2$$ detects all beads with sufficient accuracy while effectively suppressing high frequency noise. As suggested in the Computer Vision literature,[^29][^30] we approximate $$\nabla^2I$$ by the difference of two Gaussian convolutions (DoG) of the image $$I$$ with a standard deviation $$\sigma$$ of 1.4 px and 1.8 px respectively. All local minima in a 3×3×3 neighborhood in $$\nabla^2I$$ represent intensity maxima whose sub-pixel location is then estimated by fitting a 3d quadratic function to this neighbourhood.[^31] The DoG detector identifies beads even if they are close to each other, close to the sample or those with an unexpected shape. It also massively oversegments the image detecting 'blob-like' structures, corners and various locations alongside edges or planes within the imaged sample. However, those detections do not interfere with the registration process as the descriptors that incorporate them are filtered out by local descriptor matching (see [Figure 2](/media/plugins/spim-registration/descriptorbuildup.png)). Only beads are repeatably detected in different views.

### Establishing Bead Correspondences

{% include thumbnail src='/media/plugins/spim-registration/descriptorbuildup.png' title='<b>Figure 2:</b> Rotation invariant local geometric descriptor'%} To register two views $$A$$ and $$B$$ the corresponding bead pairs $$(\vec{a},\vec{b})$$ have to be identified invariantly to translation and rotation. To this end, we developed a *geometric local descriptor* . The local descriptor of a bead is defined by the locations of its 3 nearest neighbors in 3d image space ordered by their distance to the bead. To efficiently extract the nearest neighbors in image space we use the *kd*-tree implementation of the WEKA framework.[^32] Translation invariance is achieved by storing locations relative to the bead. That is, each bead descriptor is an ordered 3d point cloud of cardinality 3 with its origin $$\vec{o} = (0,0,0)^T$$} being the location of the bead.

Local descriptor matching is performed invariantly to rotation by mapping the ordered point cloud of all beads $$\vec{a}\in A$$ to that of all beads $$\vec{b}\in B$$ individually by means of least square point mapping error using the closed-form unit quaternion-based solution.[^33] The similarity measure $$\epsilon$$ is the average point mapping error. Each candidate in $$A$$ is matched against each candidate in $$B$$. Corresponding descriptors are those with minimal $$\epsilon$$. This approach, however, is computationally very demanding as it has a complexity of $$O(n^2)$$ regarding the number of detections.[^34]

We therefore employed a variation of geometric hashing[^35] to speed up the matching process. Instead of using one reference coordinate system for the complete scene we define a local coordinate system for each of the descriptor as illustrated and described in [Figure 2](/media/plugins/spim-registration/descriptorbuildup.png). All remaining bead coordinates not used for defining the local coordinate system become rotation invariant which enables us to compare descriptors very efficiently using *kd*-trees to index remaining bead coordinates in the local coordinate system. Again, we employ the *kd*-tree implementation of the WEKA framework[^36] on a six-dimensional tree to identify nearest neighbors in the descriptor space, i.e. descriptors which are most similar. The most similar descriptors that are significantly better (10×) than the second nearest neighbor in descriptor space are designated correspondence candidates.[^37]

Descriptors composed of only four beads are not completely distinctive and similar descriptors can occur by chance. Increasing the number of beads in the descriptor would make it more distinctive to the cost of less identified correspondences and increased computation time. All true correspondences agree on one transformation model for optimal view registration, whereas each false correspondence supports a different transformation. Therefore, we used the minimal descriptor size (4 beads) and rejected false correspondences from candidate sets with the Random Sample Consensus (RANSAC)[^38] on the affine transformation model followed by robust regression.

### Global Optimization

The identified set of corresponding beads $$C_{AB} = \{(\vec{a}_i,\vec{b}_i) : i=\{1,2, \dots, \left| C\right|\}\}$$ for a pair of views $$A$$ and $$B$$ defines an affine transformation $$\mathbf{T}_{AB}$$ that maps $$A$$ to $$B$$ by means of least square bead correspondence displacement

<center>

$$ \arg\min_{\mathbf{T}_{AB}}\sum_{(\vec{a},\vec{b})\in C_{ab}}{\left\\|\mathbf{T}_{AB}\vec{a} - \vec{b}\right\\|^2}. $$

</center>

We use an affine transformation to correct for the anisotropic $$z$$-stretching of each view introduced by the differential refraction index mismatch between water and agarose[^39][^40] as the sample is never perfectly centered in the agarose column.

Registration of more than two views requires groupwise optimization of the configuration $$T_{VF} = \{\mathbf{T}_{AF} : A,F\in V\}$$ with $$V$$ being the set of all views and $$F$$ being a fixed view that defines the common reference frame. Then, the above Equation extends to

<center>

$$ \arg\min_{T_{VF}}\sum_{A\in V\setminus\{F\}}\left(\sum_{B\in V\setminus\{A\}}\left(\sum_{(\vec{a},\vec{b})\in C_{AB}}{\left\\|\mathbf{T}_{AF}\vec{a} - \mathbf{T}_{BF}\vec{b}\right\\|^2}\right)\right) $$

</center>

with $$C_{AB}$$ being the set of bead correspondences $$(\vec{a},\vec{b})$$ between view $$A$$ and view $$B$$ whereas $$\vec{a}\in A$$ and $$\vec{b}\in B$$. This term is solved using an iterative optimization scheme. In each iteration, the optimal affine transformation $$\mathbf{T}_{AF}$$ for each single view $$A\in V\setminus\{F\}$$ relative to the current configuration of all other views is estimated and applied to all beads in this view. The scheme terminates on convergence of the overall bead correspondence displacement. This solution allows us to perform the global optimization with any transformation model in case the microscopy set-up has different properties (e. g. translation[^41], rigid[^42]).

### Time-lapse registration

During extended time-lapse imaging, the whole agarose column may move. To compensate the drift, we used the bead-based registration framework to register individual time-points to each other. We select a single view $$A_t$$ from an arbitrary time-point $$t$$ in the middle of the series as reference. Subsequently, we use the stored DoG detections to identify the true corresponding local geometric descriptors for all pairs of views $$A_t$$ and $$A_{a\in{\{1,2, \dots \}\setminus\{t\}}}$$ and calculate an affine transformation $$\mathbf{T}_{A_aA_t}$$ that maps $$A_a$$ to $$A_t$$ by means of least square bead correspondence displacement. The identified transformation matrices are then applied to all remaining views in the respective time-points resulting in a registered time series ([Figure 4a](/media/plugins/spim-registration/erroranalysis.png)).

### Image Fusion and blending

{% include thumbnail src='/media/blending.jpg' title='<b>Figure 3:</b> Image fusion'%} The registered views can be combined to create a single isotropic 3d image. An effective fusion algorithm must ensure that each view contributes to the final fused volume only useful sharp image data acquired from the area of the sample close to the detection lens. Blurred data visible in other overlapping views should be suppressed. We use Gaussian Filters to approximate the image information at each pixel in the contributing views ([Figure 3c,f](/media/blending.jpg)).[^43]

For strongly scattering and absorbing specimen like *Drosophila* , we typically do not image the entire specimen in each single view, but instead stop at roughly two thirds of its depth as the images become increasingly blurred and distorted. In the reconstructed 3d image, this introduces line artifacts in areas where a view ends abruptly ([Figure 3a,d](/media/blending.jpg)). To suppress this effect for the purposes of data display, we apply non-linear blending close to the edges of each border between the views ([Figure 3b,e](/media/blending.jpg)).[^44]

Precise registration of multi-view data is the prerequisite for multi-view deconvolution of the reconstructed image which can potentially increase the resolution.[^45][^46][^47] Having sub-resolution fluorescent beads around the sample facilitates the estimation of a spatially dependent point spread function and validates the deconvolution results.

### Strategies for bead removal

The presence of sub-resolution fluorescent beads used for the registration of the views might interfere with subsequent analysis of the dataset. To computationally remove the beads from each view we compute the average bead shape by adding the local image neighborhood of all true correspondences (beads used for registration) of the respective view. The acquired template is subsequently used to identify other beads; to speed up the detection we compare this template only to all maxima detected by the Difference-of-Gaussian operator during the initial bead segmentation step. These DoG-detections contain all image maxima and therefore all beads of the sample. The beads are then removed by subtracting a normalized, Gaussian-blurred version of the bead template. This method reliably removes beads which are clearly separated from the sample judged by the average intensity in the vicinity of the detected bead. Therefore, some beads that are positioned very close to the sample are not removed as the bead-subtraction would interfere with the samples' intensities.

To completely remove all beads from the sample we adapt the intensities of the beads to the imaged sample. Therefore we simply embed beads excitable by a different wavelength then the fluorescent maker in the sample and use a long-pass filter for detection ([Figure 6e,g](/media/showcase.jpg)). In such acquisition the intensity of the beads is around 2-4.

## Evaluation

### Evaluation of the performance of the bead-based registration framework

{% include thumbnail src='/media/plugins/spim-registration/erroranalysis.png' title='<b>Figure 4:</b> Analysis of the registration error'%} We created a visualization of the optimization procedure. For each view, we display its bounding box and the locations of all corresponding descriptors in a 3d visualization framework[^48]. Correspondences are color coded logarithmically according to their current displacement ranging from red (&gt;100 px) to green (&lt;1 px). The optimization is initialized with a configuration where the orientation of the views is unknown; all views are placed on top of each other and thus the corresponding descriptor displacement is high (red). As the optimization proceeds, the average displacement decreases (yellow) until convergence at about one pixel average displacement (green) is achieved. [Video 2](/media/plugins/spim-registration/supplementary-video-2-global-opt.mov) shows the optimization progress for an 8 angle acquisition of fixed *C.elegans* . The outline of the worm forms in the middle (grey), since many worm nuclei were segmented by the DoG detector but discarded during establishment of bead correspondences.

The global optimization scheme can be seamlessly applied to tiled multi-view acquisitions. In such a set-up, different parts of a large 3d sample are scanned with a high-magnification lens from multiple angles separately. All such acquisitions can be mixed, discarding all information about their arrangement and the global optimization recovers the correct configuration. An example of such optimization is shown in [Video 3](/media/plugins/spim-registration/supplementary-video-3-global-opt-with-tiling.mov) containing a fixed *Drosophila* embryo imaged from 8 angles across two or three tiles per angle on a single photon confocal microscope with a 40×/0.8 Achroplan objective.

To prove the accuracy of the bead-based registration framework, we created a simulated bead-only dataset with beads approximated by a Gaussian filter response with σ=1.5 px. We generated 8 different views related by an approximately rigid affine transformation with isotropic resolution. The reconstruction of this dataset yielded an average error of 0.02 px ([Table 1](#Table1)). For real-life datasets, the registration typically results in errors of about 1 px or slightly lower ([Figure 4b](/media/plugins/spim-registration/erroranalysis.png) and [Table 1](#Table1)) where the remaining error is introduced by the localization accuracy of the bead detector and small non-affine disturbances induced by elastic deformation of the agarose. In the $$xy$$-plane of each view, the beads can be localized very precisely, however, alongside $$z$$, the localization accuracy drops due to a lower sampling rate and an asymmetric PSF. This is supported by the dimension dependent error shown in [Figure 4c,d](/media/plugins/spim-registration/erroranalysis.png).

[Table 1](#Table1) shows that for the registration we usually use significantly more beads than necessary to solve an affine transformation (4 beads). It is necessary to use many beads for registration due to the following reasons:

-   If the overlap between stacks is—as in many examples shown—very small, it has to be ensured that in these small overlapping areas there are still enough fiduciary markers to align the stacks.
-   If two stacks are extensively overlapping, the beads have to be evenly distributed around the sample to ensure an even error distribution throughout the sample. Otherwise, small errors in, for example, the lower left corner do not control the registration error in the upper right corner where there are no fiduciary markers.
-   The localization error of the beads is normally distributed as shown in [Figure 4c,d](/media/plugins/spim-registration/erroranalysis.png). That means, the more beads are included in the registration, the more accurate is the average localization of the beads. Consequently, the affine transformation for each individual stack yields lower residual error the more beads are used.

Due to the optical properties of the sample it might occur that beads are distorted and aberrated and therefore detected in the wrong location. Typically, such beads are excluded as they do not form repeatable descriptors between the different stacks. If the distortions are minor, they will contribute to the residual error of the affine transformation. However, the contribution is small as those beads represent a very small fraction of all beads. This is supported by the maximal transfer error of  1 px of the affine model (see also inset of [Figure 5c,d](/media/plugins/spim-registration/intensity-vs-beads.jpg)).

Examples of within and across time-points registered time-series of *Drosophila* embryonic development are shown in [Video 4](/media/plugins/spim-registration/supplementary-video-4-drosophila-gastrulation.mov) and [Video 5](/media/plugins/spim-registration/supplementary-video-5-drosophila-embryogenesis.mov). The videos show 3d renderings of the developing embryos expressing His-YFP in all cells. The 3d rendered single embryo is shown from four and three arbitrary angles to highlight the complete coverage of the specimen. [Video 4](/media/plugins/spim-registration/supplementary-video-4-drosophila-gastrulation.mov) shows in 42 time-points (7 angles) the last two synchronous nuclear divisions followed by gastrulation. [Video 5](/media/plugins/spim-registration/supplementary-video-5-drosophila-embryogenesis.mov) captures in 249 time-points (5 angles) *Drosophila* embryogenesis from gastrulation to mature embryo when muscle activity effectively prevents further imaging.

### Performance of the bead-based registration framework in comparison with intensity-based methods

{% include thumbnail src='/media/plugins/spim-registration/intensity-vs-beads.jpg' title='<b>Figure 5:</b> Comparison of bead-based and intensity-based multi-view reconstruction on 7-view acquisition of Drosophila embryo expressing His-YFP'%} Existing muti-view SPIM registration approaches that use sample intensities to iteratively optimize the quality of the overlap of the views do not work reliably and are computationally demanding.[^49][^50][^51] Alternatively, the registration can be achieved by matching of segmented structures, such as cell nuclei, between views[^52] However, such approaches are not universally applicable, as the segmentation process has to be adapted to the imaged sample.

To evaluate the precision and performance of the bead-based registration framework we compared it against the intensity-based registration method that we developed previously.[^53] This method identifies the rotation axis common to all views by iterative optimization of FFT-based phase correlation between adjacent views. We applied both methods (bead-based and intensity-based) to a single time-point of live 7-view acquisition of *Drosophila* embryo expressing His-YFP in all cells embedded in agarose with beads. We chose a time-point during blastoderm stage where the morphology of the embryo changes minimally over time. We evaluated the precision of both methods by the average displacement of the corresponding beads and concluded that the bead-based registration framework clearly outperformed the intensity-based registration in terms of bead registration accuracy (0.98 px versus 6.91 px, see [Table 1](#Table1), [Figure 5](/media/plugins/spim-registration/intensity-vs-beads.jpg)). The increased precision in the bead alignment achieved by the bead-based registration framework is reflected in noticeably improved overlap of the nuclei in the sample (see [Figure 5c–h](/media/plugins/spim-registration/intensity-vs-beads.jpg)). Moreover, the intensity-based method required approximately 9 hours of computation time compared to 2.5 minutes for the bead-based registration framework executed on the same computer hardware (Intel Xeon E5440 with 64GB of RAM), i.e. the bead-based framework is about 200  faster for this dataset.

------------------------------------------------------------------------

{::nomarkdown}
<table>
  <caption>
    style="margin-top:1em; margin-bottom:1em;" |<strong>Table 1:</strong> Statistics of multi-view registration of various datasets
  </caption>
  <thead>
    <tr class="header">
      <th>
        <p>Dataset</p>
      </th>
      <th>
        <p>min/avg/max error [px]</p>
      </th>
      <th>
        <p>DoG detections</p>
      </th>
      <th>
        <p>True correspondence number (ratio)</p>
      </th>
      <th>
        <p>processing time [min:sec]</p>
      </th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <p>Fixed <em>C.elegans</em>, 8 views<br>
        SPIM 40×/0.8NA</p>
      </td>
      <td>
        <p>1.02/1.12/1.31</p>
      </td>
      <td>
        <p>4566</p>
      </td>
      <td>
        <p>1717 (98%)</p>
      </td>
      <td>
        <p>11:09</p>
      </td>
    </tr>
    <tr>
      <td>
        <p>Live <em>Drosophila</em>, 5 views<br>
        SPIM 20×/0.5NA</p>
      </td>
      <td>
        <p>0.76/0.81/1.31</p>
      </td>
      <td>
        <p>9267</p>
      </td>
      <td>
        <p>1459 (97%)</p>
      </td>
      <td>
        <p>2:31</p>
      </td>
    </tr>
    <tr>
      <td>
        <p>Fixed <em>Drosophila</em>, 10 views<br>
        SPIM 20×/0.5NA</p>
      </td>
      <td>
        <p>0.65/0.78/0.97</p>
      </td>
      <td>
        <p>9035</p>
      </td>
      <td>
        <p>1301 (93%)</p>
      </td>
      <td>
        <p>20:10</p>
      </td>
    </tr>
    <tr>
      <td>
        <p>Fixed <em>Drosophila</em>, 11 views<br>
        Spinning Disc 20&amp;times/0.5NA</p>
      </td>
      <td>
        <p>1.10/1.33/1.86</p>
      </td>
      <td>
        <p>6309</p>
      </td>
      <td>
        <p>978 (92%)</p>
      </td>
      <td>
        <p>6:15</p>
      </td>
    </tr>
    <tr>
      <td>
        <p>Simulated Dataset, 8 views<br>
        Isotropic Resolution</p>
      </td>
      <td>
        <p>0.02/0.02/0.02</p>
      </td>
      <td>
        <p>2594</p>
      </td>
      <td>
        <p>2880 (96%)</p>
      </td>
      <td>
        <p>15:54</p>
      </td>
    </tr>
    <tr>
      <td>
        <p>Live <em>Drosophila</em>, 7 views<br>
        SPIM 20×/0.5NA<br>
        <strong>bead-based</strong></p>
      </td>
      <td>
        <p>0.87/0.98/1.17</p>
      </td>
      <td>
        <p>6232</p>
      </td>
      <td>
        <p>603 (97%)</p>
      </td>
      <td>
        <p>2:27</p>
      </td>
    </tr>
    <tr>
      <td>
        <p>Live <em>Drosophila</em>, 7 views<br>
        SPIM 20×/0.5NA<br>
        <strong>intensity-based</strong></p>
      </td>
      <td>
        <p>0.93/6.91/9.59</p>
      </td>
      <td>
        <p>n.a.</p>
      </td>
      <td>
        <p>n.a.</p>
      </td>
      <td>
        <p>515:10</p>
      </td>
    </tr>
  </tbody>
</table>
{:/}

style="margin-top:1em; margin-bottom:1em;" \|**Table 1:** Statistics of multi-view registration of various datasets

We show minimal, average and maximal displacement of all true correspondences (beads) after convergence of the global optimization. The simulated dataset shows very low registration errors. The total number of DoG detections is typically much higher than the number of extracted correspondence candidate pairs although a DoG detection can participate in more than one correspondence pair. The ratio of true correspondences versus correspondence candidates is typically above 90%. Lower ratios indicate a registration problem, for example caused by movement of the agarose during stack acquisition. The processing time (segmentation and registration) was measured on a dual quad-core Intel Xeon E5440 system.

------------------------------------------------------------------------

## Samples imaged by SPIM

### Overview of imaged specimens

{% include thumbnail src='/media/showcase.jpg' title='<b>Figure 6:</b> Examples of various reconstructed multi-view datasets'%} We demonstrated the performance of our registration framework on multi-view *in toto* imaging of fixed and living specimen of various model organisms ([Figure 6](/media/showcase.jpg)), in particular *Drosophila* . Fixed *Drosophila* embryos were stained with Sytox-Green to label all nuclei. For live imaging, we used a developing *Drosophila* embryo expressing fluorescent His-YFP under the control of endogenous promoter visualizing all nuclei. *Drosophila* specimens were imaged with a SPIM prototype equipped with a Zeiss 20×/0.5 Achroplan objective.

### Sample mounting for SPIM

We matched the fluorescence intensity of the beads to the signal intensity of the sample. For live imaging of His-YFP which is relatively dim and requires longer exposure times (0.3 s), we used red or yellow fluorescent beads that are suboptimal for the GFP detection filter set and therefore typically less bright than the sample. Conversely, for the imaging of the bright, fixed specimen, we used green fluorescent beads which give adequate signal at very short exposure times (0.01 s).

Despite the fact that our algorithm is robust with respect to the amount of beads available for registration, too many beads unnecessarily increase the computation time, while too few beads may result in an inadequate number of correspondences due to incomplete overlap of the views. Therefore, we determined empirically the optimal concentration of beads for each magnification (ideally 1,000–2,000 beads per imaged volume). We prepared a 2× stock solution of beads (13 μl of concentrated bead solution (Estapor Microspheres FXC050))

### Sample independent registration

We applied the bead-based registration framework to various samples derived from major model organisms. These include *Drosophila* embryo, larva ([Figure 6a–c](/media/showcase.jpg)) and oogenesis ([Figure 6d](/media/showcase.jpg)), *C. elegans* adult (data not shown), larval stages ([Figure 6e](/media/showcase.jpg)) and early embryo ([Figure 6f](/media/showcase.jpg)), whole mouse embryo ([Figure 6g](/media/showcase.jpg)), and dual color imaging of zebrafish embryo ([Figure 6h](/media/showcase.jpg)). Despite the fact that the samples range significantly in their size, fluorescent labeling, optical properties and mounting formats the bead-based registration framework was invariably capable of achieving the registration. Therefore, we conclude that our method is sample independent and is universally applicable for registration of any multi-view SPIM acquisition where the sample movement does not disturb the rigidity of the agarose.

## Broad applicability of the bead-based framework to multi-view imaging

{% include thumbnail src='/media/plugins/spim-registration/rotation-chamber.png' title='<b>Figure 7:</b> Multi-view imaging with spinning disc confocal microscopy'%} Having the bead-based registration framework for multi-view reconstruction established, we sought to expand its application beyond SPIM, to other microscopy techniques capable of multi-view acquisition.[^54] We designed a sample-mounting set-up that allows imaging of a sample embedded in a horizontally positioned agarose column with fluorescent beads ([Figure 7a](/media/plugins/spim-registration/rotation-chamber.png)). The agarose column was manually rotated mimicking the SPIM multi-view acquisition. We acquired multiple views of fixed Drosophila embryos stained with nuclear dye on a spinning disc confocal microscope and reconstructed the views using the bead-based registration framework. By mosaicking around the sample, we captured the specimen in toto and achieved full lateral resolution in areas that are compromised by the poor axial resolution of a single-view confocal stack ([Figure 7b,c,d](/media/plugins/spim-registration/rotation-chamber.png) and [Video 6](/media/plugins/spim-registration/supplementary-video-6-drosophila-spinning-disc.mov) ). The combination of multi-view acquisition and bead-based registration is applicable to any imaging modality as long as the fluorescent beads can be localized and the views overlap.

### Sample mounting for multi-view imaging on an upright microscope

We constructed a sample chamber for multi-view imaging on an upright microscope that consists of a teflon dish equipped with a hole in the side wall which has the diameter of a standard glass capillary ([Figure 7a](/media/plugins/spim-registration/rotation-chamber.png)). The capillary mounting hole continues on the bottom of the dish as a semi-circular trench of approximately half the thickness of the capillary diameter, extending about 2/3 of the dish radius towards the center of the dish. This trench serves as a bed for the glass capillary inserted through the capillary mounting hole. The trench is extended by a second, shallower trench whose bottom is elevated with respect to the deeper trench by the thickness of the capillary glass wall. The second trench serves as a bed for the agarose column which is pushed out of the capillary by a tightly fitted plunger (not shown). The teflon dish is equipped on one side with a plastic window enabling visual inspection of the sample and the objective lens.

For imaging, the capillary with the sample embedded in agarose containing appropriate amount of fluorescent beads was inserted into the capillary mounting hole until it reached the end of the capillary bed. The teflon dish was filled with water and the agarose was pushed out of the capillary into the agarose bed by the plunger. Water dipping objective was lowered into the dish and focussed on the *Drosophila* embryo specimen in agarose. A confocal stack was acquired using variety of optical sectioning techniques (spinning disc confocal ([Figure 7](/media/plugins/spim-registration/rotation-chamber.png)), single photon confocal (see [Video 3](/media/plugins/spim-registration/supplementary-video-3-global-opt-with-tiling.mov)), two photon confocal, apotome (data not shown). Next, to achieve multi-view acquisition, the agarose column was retracted into the capillary by the plunger and the capillary was manually rotated. The angle of the rotation was only very roughly estimated by the position of a tape piece attached to the capillary. The agarose was again pushed out into the agarose bed and another confocal stack was collected. In this way arbitrary number of views can be collected as long as the sample does not bleach.

## Implementation

{% include thumbnail src='/media/screenshot.png' title='<b>Figure 8:</b> Screenshot of SPIM registration plugin in Fiji'%} The bead-based registration framework is implemented in the Java programming language and provided as a fully open source plugin packaged with the ImageJ distribution [Fiji](/software/fiji) (Fiji Is Just ImageJ, that is actively developed by an international group of developers. The plugin ([Figure 8](/media/screenshot.png)) performs all steps of the registration pipeline: bead segmentation, correspondence analysis of bead-descriptors, outlier removal (RANSAC and global regression), global optimization including optional visualization, several methods for fusion, blending and time-lapse registration.

The tutorial on how to use the plugin in basic and advanced mode is available at [SPIM Registration](/plugins/spim-registration). The test data containing 7-view SPIM acquisitions of *Drosophila* embryo can be downloaded from [^1](http://fly.mpi-cbg.de/preibisch/nm/HisYFP-SPIM.zip).

## Acknowledgments

We want to thank [Carl Zeiss Microimaging](http://www.zeiss.de/micro) for access to the SPIM demonstrator, Radoslav Kamil Ejsmont[^55] for His-YFP flies, Dan White,[^56], Jonathan Rodenfels,[^57] Ivana Viktorinova,[^58] Mihail Sarov,[^59] Steffen Jänsch,[^60] Jeremy Pulvers,[^61] and Pedro Campinho[^62] for providing various biological samples for imaging with SPIM shown in [Figure 6](/media/showcase.jpg).

<references group="A" />

## References

[^1]: {% include citation content='journal' author='J. Huisken and J. Swoger and F. D. Bene and J. Wittbrodt and E. H. K. Stelzer' title='Optical Sectioning Deep Inside Live Embryos by Selective Plane Illumination Microscopy' journal='Science' volume='305' number='5686' pages='1007 1010' year='2004' %}

[^2]: {% include citation content='journal' author='P. Shaw, D. Agard, Y. Hiraoka, and J. Sedat' title='Tilted view reconstruction in optical microscopy. Three-dimensional reconstruction of Drosophila melanogaster embryo nuclei' journal='Biophysical Journal' volume='55' number='1' year='1989' pages='101–110' %}

[^3]: 

[^4]: {% include citation doi='10.1038/nmeth1036' %}

[^5]: 

[^6]: {% include citation content='journal' author='J. Swoger, P. Verveer, K. Greger, J. Huisken, and E. H. K. Stelzer' journal='Opt. Express' number='13' pages='8029–8042' title='Multi-view image fusion improves resolution in three-dimensional microscopy' volume='15' year='2007' %}

[^7]: {% include citation title='Three Dimensional X-Ray Opaque Foreign Body Marker Device' author='E. H. Gullekson' year='Patent number: 3836776, Filing date: 1 Mar 1973, Issue date: Sep 1974' %}

[^8]: {% include citation content='journal' author='B.J. Erickson and Jack C.R. Jr.' title='Correlation of single photon emission CT with MR image data using fiduciary markers' journal='American Journal of Neuroradiology' volume='14' number='3' pages='713 720' year='1993' %}

[^9]: {% include citation content='journal' author='J. M. Fitzpatrick and J. B. West' title='The distribution of target registration error in rigid-body point-based registration' journal='IEEE Transactions on Medical Imaging' volume='20' number='9' year='2001' month='September' pages='917 927' %}

[^10]: {% include citation content='journal' author='A. D. Wiles, A. Likholyot, D. D. Frantz, and T.M.Peters' title='A Statistical Model for Point-Based Target Registration Error With Anisotropic Fiducial Localizer Error' journal='IEEE Transactions on Medical Imaging' volume='27' number='3' year='2008' month='March' pages='378 390' %}

[^11]: {% include citation doi='10.1007/978-3-540-85990-1_124'  %}

[^12]: {% include citation content='journal' title='Towards automatic electron tomography' journal='Ultramicroscopy' volume='40' number='1' pages='71 87' year='1992' author='K. Dierksen, D. Typke, R. Hegerl, A. J. Koster, and W. Baumeister' %}

[^13]: {% include citation content='journal' author='A. J. Koster, R. Grimm, D. Typke, R. Hegerl, A. Stoschek, J. Walz, and W. Baumeister' title='Perspectives of molecular and cellular electron tomography' journal='J Struct Biol' year='1997' volume='120' number='3' pages='276 308' %}

[^14]: 

[^15]: {% include citation content='conference' title='A Combined Corner and Edge Detector' author='C. Harris and M. Stephens' locaiton='Plessey Research Roke Manor, UK' booktitle='Proceedings of The Fourth Alvey Vision Conference' location='Manchester' year='1988' pages='147 151' %}

[^16]: {% include citation content='journal' title='A computational approach to edge detection' author='J. Canny' journal='IEEE Transactions on Pattern Analysis Machine Intelligence' volume='8' number='6' year='1986' pages='679 698' %}

[^17]: {% include citation content='conference' author='Y. Lamdan, J. Schwartz, and H. Wolfson' title='On recognition of 3D objects from 2D images' booktitle='Proceedings of the IEEE International Conference on Robotics and Automation' year='1988' pages='1407 1413' publisher='IEEE Computer Society' location='Los Alamitos, CA' %}

[^18]: {% include citation content='conference' author='D. W. Hogg, M. Blanton, D. Lang, K. Mierle, and S. Roweis' title='Automated Astrometry' booktitle='Astronomical Data Analysis Software and Systems XVII' year='2008' series='Astronomical Society of the Pacific Conference Series' volume='394' editor='R. W. Argyle, P. S. Bunclark, and J. R. Lewis' month='August' pages='27-+' %}

[^19]: {% include citation content='journal' author='R. Nussinov and H. J. Wolfson' journal='Proc Natl Acad Sci U S A' number='23' pages='10495 10499' title='Efficient detection of three-dimensional structural motifs in biological macromolecules by computer vision techniques.' volume='88' year='1991' %}

[^20]: {% include citation content='journal' author='D. Fischer, H. Wolfson, S. L. Lin, and R. Nussinov' title='Three-dimensional, sequence order-independent structural comparison of a serine protease against the crystallographic database reveals active site similarities: Potential implications to evolution and to protein folding' journal='Protein Science' volume='3' number='5' pages='769 778' year='1994' %}

[^21]: {% include citation content='journal' author='A.C. Wallace, N. Borkakoti, and J. M. Thornton' title='Tess: A geometric hashing algorithm for deriving 3D coordinate templates for searching structural databases. Application to enzyme active sites' journal='Protein science' year='1997' volume='6' number='11' pages='2308' %}

[^22]: {% include citation content='journal' title='A Model for Statistical Significance of Local Similarities in Structure' journal='Journal of Molecular Biology' volume='326' number='5' pages='1307 1316' year='2003' author='A. Stark, S. Sunyaev, and R. B. Russell' %}

[^23]: {% include citation content='conference' author='A. Stanski and O. Hellwich' title='Spiders as Robust Point Descriptors' booktitle='DAGM-Symposium' year='2005' pages='262 268' %}

[^24]: {% include citation content='journal' author='D. G. Lowe' title='Distinctive image features from scale-invariant keypoints' journal='Int J Comput Vis' volume='60' number='2' pages='91 110' year='2004' %}

[^25]: {% include citation content='journal' author='B. Kuipers and Yung-tai Byun' title='A Robot Exploration and Mapping Strategy Based on a Semantic Hierarchy of Spatial Representations' journal='Journal of Robotics and Autonomous Systems' year='1991' volume='8' pages='47 63' %}

[^26]: {% include citation content='conference' author='D. Bradley, D. Silver, and S. Thayer' title='A regional point descriptor for global localization in subterranean environments' booktitle='IEEE conference on Robotics Automation and Mechatronics (RAM 2005)' pages='440 445' month='December' year='2004' volume='1' %}

[^27]: {% include citation content='conference' author='A. Frome, D. Huber, R. Kolluri, T. Bulow, and J. Malik' title='Recognizing objects in range data using regional point descriptors' booktitle='Proceedings of the European Conference on Computer Vision (ECCV)' month='May' year='2004' %}

[^28]: {% include citation content='journal' author='P. Matula, M. Kozubek, F. Staier, and M. Hausmann' journal='Journal of microscopy' pages='126 42' title='Precise 3D image alignment in micro-axial tomography.' volume='209' year='2003' %}

[^29]: {% include citation content='journal' author='T. Lindeberg' title='Scale-space theory: A basic tool for analysing structures at different scales' journal='Journal of Applied Statistics' volume='21' number='2' pages='224 270' year='1994' %}

[^30]: 

[^31]: {% include citation content='conference' author='M. Brown and D. Lowe' title='Invariant Features from Interest Point Groups' booktitle='In British Machine Vision Conference' year='2002' pages='656 665' %}

[^32]: {% include citation content='book' author='I. H. Witten and E. Frank' title='Data Mining: Practical machine learning tools and techniques' publisher='Morgen Kaufmann' location='San Francisco' isbn='0-12-088407-0' edition='second' year='2005' %}

[^33]: {% include citation content='journal' author='B. K. P. Horn' title='Closed-form solution of absolute orientation using unit quaternions' journal='Journal of the Optical Society of America A' volume='4' number='4' pages='629 642' year='1987' %}

[^34]: {% include citation doi='10.1117/12.812612' %}

[^35]: 

[^36]: 

[^37]: 

[^38]: {% include citation content='journal' author='M. A. Fischler and R. C. Bolles' title='Random sample consensus: a paradigm for model fitting with applications to image analysis and automated cartography' journal='Communications of the ACM' volume='24' number='6' pages='381 395' year='1981' %}

[^39]: {% include citation content='journal' title='Aberrations in Confocal Fluorescence Microscopy Induced by Mismatches in Refractive Index' journal='Journal of Microscopy' volume='169' number='3' pages='341 405' year='1993' author='S. Hell and G. Reiner and C. Cremer and E.H.K. Stelzer' %}

[^40]: 

[^41]: {% include citation doi='10.1093/bioinformatics/btp184' %}

[^42]: 

[^43]: {% include citation doi='10.1117/12.770893' %}

[^44]: 

[^45]: 

[^46]: {% include citation content='journal' author='C. J. Engelbrecht and E. H. K. Stelzer' title='Resolution enhancement in a light-sheet-based microscope (SPIM)' journal='Optics Letters' volume='31' number='10' pages='1477 1479' year='2006' %}

[^47]: {% include citation content='journal' title='High-resolution three-dimensional imaging of large specimens with light sheet-based microscopy' journal='Nature Methods' volume='4' number='4' pages='311 313' year='2007' author='P.J. Verveer and J. Swoger and F. Pampaloni and K. Greger and M. Marcello and E.H.K.Stelzer' %}

[^48]: {% include citation content='conference' author='B. Schmidt' title='Hardware-accelerated 3D visualization for ImageJ' booktitle='ImageJ User and Developer Conference' year='2008' volume='2' editor='Pierre Plumer' %}

[^49]: {% include citation content='journal' author='Jim Swoger and Peter Verveer and Klaus Greger and Jan Huisken and Ernst H. K. Stelzer' journal='Opt. Express' number='13' pages='8029 8042' publisher='OSA' title='Multi-view image fusion improves resolution in three-dimensional microscopy' volume='15' year='2007' %}

[^50]: {% include citation doi='10.1117/12.770893' %}

[^51]: {% include citation content='conference' author='S. Preibisch and R. Ejsmont and T. Rohlfing and P. Tomancak' title='Towards Digital Representation of Drosophila Embryogenesis' booktitle='Proceedings of 5th IEEE International Symposium on Biomedical Imaging' pages='324 327' year='2008' %}

[^52]: {% include citation content='journal' author='P. J. Keller and A. D. Schmidt and J. Wittbrodt and E. H. Stelzer' title='Reconstruction of zebrafish early embryonic development by scanned light sheet microscopy' journal='Science' volume='322' number='5904' pages='1065-9' year='2008' %}

[^53]: {% include citation content='conference' author='S. Preibisch and R. Ejsmont and T. Rohlfing and P. Tomancak' title='Towards Digital Representation of Drosophila Embryogenesis' booktitle='Proceedings of 5th IEEE International Symposium on Biomedical Imaging' pages='324 327' year='2008' %}

[^54]: {% include citation content='journal' author='J. Bradl and M. Hausmann and V. Ehemann and D. Komitowski and C. Cremer' title='A tilting device for three-dimensional microscopy: application to in situ imaging of interphase cell nuclei.' journal='Journal of Microscopy' year='1992' volume='168' pages='47 57' %}

[^55]: Max Planck Institute of Molecular Cell Biology and Genetics, Dresden, Germany

[^56]: 

[^57]: 

[^58]: 

[^59]: 

[^60]: 

[^61]: 

[^62]: 
