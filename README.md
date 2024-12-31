# Intro

This repo serves to document a submission for the first automated segmentation price of the vesuvius challenge 2024.
It serves as a documentation on how to run the full automated segmentation pipeline and a decription of the algorithms and tools used.

# Links & Repos
- this documentation: https://github.com/hendrikschilling/FASP
- surface tracing and inspection: https://github.com/hendrikschilling/volume-cartographer (branch dev-next)
- ink detection: https://github.com/hendrikschilling/Vesuvius-Grandprize-Winner

# Submission Data

The final data is available at:
https://dl.ash2txt.org/community-uploads/waldkauz/fasp/

Overview:
- /fasp_fill_hr_20241230145834485 the final submission surface in tiffxyz format
~~~- /ink.jpg - the ink detection of the submitted surface, aligns with the surface xyz pixels by scaling the xyz coords by 1.25x (ink is sampled at 1/16 and the surface at 1/20)~~~ not yet uploaded as per submission form hint
- /layers - layer 0 - 21 where 10 is the central layer not offset against the surface, they are at half of full voxel resolution so 10:1 against the surface xyz tiffs and 8:1 against the ink detection
- /masks - additional information on the surface quality, a value of 0 in the state.tif means no surface, 100 - high quality, 80 - infilled

# Tools & Contributions

- volume surface prediciton: *Segmenting_Scroll_Surfaces.pdf*
- vc_grow_seg_from_seed: Generate patches in a volume prediction, previously released and documented: https://discord.com/channels/1079907749569237093/1162822163171119194/threads/1312490723001499808
- vc_grow_seg_from_segments: trace larger surfaces by searching for consensus points on collections of surface patches, this can trace very large continuous surfaces and represents the core part of this submission
- vc_tifxyz_winding: estimate consistent relative winding numbers for a trace
- vc_fill_quadmesh: inpainting and flattening of large traces
- vc_render_tifxyz: fast rendering of huge traces (tested with sizes surpassing the jpg img size limit!)

# FASP Criteria

## Inputs

The submission was created using approximately TODO of compute and 4 hours of human supervision.

### Compute Time

Computer times where split approximately like this (time in minutes):
- inference TODO
- packing TODO
- patch seed	53
- patch expansion	599
- repeated traces	720
- winding estimation	1
- Inpaint & flattening	40
- render	31
- ink	30
On an AMD 5950x, Nvidia RTX3090 and 64GB of RAM.
Execution times are documented per task in the relevant sections below.

### Human Input

Human input was (appart from starting prepared commands and file operations) limited to 4 hours.
The time was used to create 243 annotations, the detailed process is described later in this document (compare step 4.4 Annotation).
The time split is appoximately (minutes):
- 237 coarse masks: 119
- 6 fine masks: 60
- inspection: 60

## Outputs

### Geometry
The result is submitted as a quadmesh stored as "tiffxyz" - every quad corner is a single pixel with the coordinates stored in x.tif/y.tif/z.tif and some metadata information in meta.json. It is a single continuous manifold mesh that covers most of the GP banner (and a bit more on the inside). High quality areas (compare state.tif) should be free of self-intersections.

### Segmentation Quality
High quality areas (compare state.tif) surpass the GP quality and quite closely follow the surface.

### Flattening
Most areas show low distortion, some distortion occurs around inpainted areas, especially at the left.

### Ink Detection
The ink detection is based on the GP ink detection, with the code adapted to produce smaller files much faster and with lower memory requirements

## Tradeoffs

To achieve the submission in the allowed time several tradeoffs were made. Depending on the goal it is possible choose a different quality/compute/human-time tradeoff:

- patch expansion was run only with 16 instead of 32 threads to have capacity for other tasks at that time, also as long as fast shared network storage is available this could be processed in a distributed manner
- ink detecton accuracy: Quality was set to "1", the ink detection code has higher quality modes available, but time requirements rise quadratic.
- rendering resoution: to limit rendering times and RAM requirements half resolution source volumes and render resolution was used and then upscaled on-the-fly in the ink detection. This reduces ink detection quality slightly.
- Annotation time: If more human input is acceptable most of the inpainted areas could be raised to the high quality standard of the trace. If the annotation is confined to regions of interest after an initial ink detection run annotations times should not explode too much.

# Overview

The overall process consists of the following steps:
1. surface prediction volume generation
2. patch collection seeding
3. patch collection expansion
4. iterative surface tracing & annotation
5. fusion & flattening of large traces
6. rendering
7. ink prediction



# Input Data

## Surface Prediction

## Tracing to Ink Detection
The tracer and the rendering require ome-zarr volumes with a meta.json file according to the VC requirements:
For example:
```
{"height":7888,"max":65535.0,"min":0.0,"name":"thename","slices":14376,"type":"vol","uuid":"theuuid","voxelsize":7.91,"width":8096, "format":"zarr"}
```
For the submission this volume was used:
https://dl.ash2txt.org/community-uploads/james/Scroll1/Scroll1_8um.zarr/

# Guide
This guide serves to document the steps taken to achieve the FASP submission without diverting too much into all the additional options the tools provide.

## 1. Inference

The volume prediction is documented in *Segmenting_Scroll_Surfaces.pdf*.

## 2. patch collection seeding
The patch seeding will trace small (4cm^2 by default) patches from the surface prediction provided as ome-zarr. Detailed documentation is available [in this thread](https://discord.com/channels/1079907749569237093/1312490723001499808).
The command used for the submission is:

```
time seq 1 1000 |  xargs -i -P 32 bash -c 'nice ionice vc_grow_seg_from_seed /path/to/gp-prediction-ome-zarr /path/to/patch-collection params_grow_seed.json'
```

params_grow_seed.json:
```
{
    "cache_root" : "/path/to/cache",
    "generations" : 200,
    "min_area_cm" : 0.3,
    "thread_limit" : 1
}
```

runtime:
```
real    52m51.294s
user    1487m27.790s
sys     35m10.965s
```

This process generated an initial set of 775 patches.

The patches can be inspected with VC3D by placing them in the paths directory of the volpkg. Symlinks can also be used.

## 3. patch collection expansion
The patche expansion step also documented [here](https://discord.com/channels/1079907749569237093/1312490723001499808) generates a set of overlapping patches throughout the whole gp-prediction volume.

The commands used where: 

```
time seq 1 10000000 |  xargs -i -P 32 bash -c 'nice ionice vc_grow_seg_from_seed /path/to/gp-prediction-ome-zarr /path/to/patch-collection params_expand_overlap.json || true'
time seq 1 10000000 |  xargs -i -P 16 bash -c 'nice ionice vc_grow_seg_from_seed /path/to/gp-prediction-ome-zarr /path/to/patch-collection params_expand_overlap.json || true'
time seq 1 1000000000 |  xargs -i -P 16 bash -c 'nice ionice vc_grow_seg_from_seed /path/to/gp-prediction-ome-zarr /path/to/patch-collection params_expand_overlap.json || true'
```

with params_expand_overlap.json:
```
{
    "cache_root" : "/path/to/cache",
    "tgt_overlap_count" : 10,
    "generations" : 200,
    "min_area_cm" : 0.3,
    "thread_limit" : 1,
    "mode" : "expansion"
}
```

this resulted in a growing number of patches:
```
7817
10444
17980
```

And the following runtimes:
```
real    259m36.492s
user    7128m20.478s
sys     257m29.490s

real    123m11.015s
user    1753m25.029s
sys     110m42.635s

real    215m33.051s
user    2890m24.039s
sys     346m59.649s
```

The patches can be inspected with VC3D by placing them in the paths directory of the volpkg. Symlinks can also be used.

## 4. iterative surface tracing
This step will finally create larger connected surfaces. The large surface tracer generates a single surface by tracing a "consensus" surface from the patches generated earlier. This processed can be influenced by annotating patches in several ways which allows to guide the tracer to avoid errors. Hence the process looks like this:
1. pick a seed patch
2. generate a trace
3. check for errors
4. annotation
and repeat step 2-4 or 1-4 several times

Keep previous trace rounds around as the fusion step can later join multple traces and problematic areas of a trace can be masked out. A mask can be generated for example with VC3D, or by using any 8-bit grayscale image with the same aspect ratio as one of the tiffxyz channels.

### 4.1 Pick a seed patch

By placing patches generated in the last step into the paths directory of the volpkg VC3D allows to inspect them. VC3D can also display the prediction ome-zarrs, which, together with fiber continuity allows to quickly scan for obvious errors. Useful tools for navigation:
- ctrl-click in any slice view to focus and slice on the clicked point
- shift-click to place a red POI
- shift-ctrl-click to place a blur POI
- filter by focus point & filter by POIs to narrow down the choice of patches
For the FASP submission seeds were picked by placing POIs on the outermost end of the outermost GP segement and selecting a large mostly error free patch from there.
Additional runs were done by selecting points from previous runs, both from the innermost area, to trace in reverse (so problem areas result in slightly different gaps) and starting just before a problem area was observed in a previous trace (to see if the annotations did help).
If a patch is selected in VC3D its path is printed on the terminal which allows to just copy that path to generate a trace command.

### 4.2. generate a trace

The traces for the submission were generated using this command:
```
time OMP_WAIT_POLICY=PASSIVE OMP_NESTED=FALSE nice vc_grow_seg_from_segments /path/to/prediction-volume /path/to/patch-collection /path/to/output params.json /path/to/seed/patch
```

Using for the finall passes these settings:
```
{
        "flip_x" : false,
        "global_steps_per_window" : 3,
        "step" : 10,
        "consensus_default_th" : 10,
        "consensus_limit_th" : 2
}
```
- flip_x determines the direction of the trace (it always grows to the right, but that can go to the inside or outside of the scroll, depending on seed location).
- global steps per window: numer of global optimization steps per moving window. The tracer operates in a moving window fasion, once the global optimization steps were run per window and no new corners were added the window is moved to the right and the process repeated. At the beginning use 0 global steps to get a fast and long trace and see if there are any coarse errors.
- consensus_default_th: lowest number of inliers (patch "supports") required per corner to be considered an inlier. Note that a single well connected patch gives more than a single support to a corner (so this is not the number of surfaces). Maximum of 20 to get only well connected patches, minimum of 6 before there are lot of errors.
For the submission values of 6 and 10 were used.
- consesus_limit_th: if we could otherwise not proceeed go down to this number of required inliers, this will only be used if otherwise the trace would end.

The tracer will generated debug/preview surface every 50 generations (labeled z_dbg_gen...) and in the and save a final surface, both in the output dir.

### 4.3. check for errors

Using VC3D (you can directly generate the surfaces into a volpkg paths dir) the traced surfaces can be inspected for errors.
Inspecting with a normal ome-zarr volume (to see fiber continuity) works well as does inspection using the predicted surface volumes.
Also POIs can be used to mark points in one view and inspect it in another (add ctrl-click or ctrl-shift-click).
The errors that need to be fixed are generally sheet jumps, were the surface jumps from one correct surface to another correct surface.
Often these are visible by checking for gaps in the generated trace as a jump will normally not reconnect with the rest of the trace (cause its on another sheet).
Then close to the bottleneck is normally where an error occurs.
It is useful to go back in the generated dbg surfaces to find the first time an error appears so as to annotate the root cause.

### 4.4. Annotation

VC3D allows to annotate patches as approved, defective, and to edit a patch mask which allows masking out areas of a patch that are problematic.
For the submission (where manual input time is quite limited) these were used like this:
**defective**
A patch can be marked as "defective" by checking the checkbox in the VC3D side panel.
This is fast but not very useful as most patches have some good areas and also will have some amount of errors. Errors only matter if they fall at the same spot in multiple patches, so marking a whole patch as defective, which will the tracer ignore it completely is not necessary. Only on patch was marked defective for the FASP submission.

**mask**
most used annotation and it can be performed quite quickly (1-3 patches per minute).
By clicking on the "edit segment mask" button a mask image will be generated and saved as .tif in the segments directory. Then the systems default application for the filetype .tif will be launched. For the submission GIMP was used. The mask file is a multi-layer tif file where the lowest layer is the actual binary mask and the second layer is the rendered surface using the currently selected volume.
To streamline the process for the FASP submission the following approach was used:
Given an error (sheet jump) found in step 4.3.
- place one POI on one side of the jump and a second on the other.
- select "filter by POIs" to get a list of patches who contain this sheet jump, this probably list about 5-20 patches
- press "edit segment mask"
- GIMP will open an import dialog, the default settings are fine so just press import
- click on the layer transparency to make the upper layer around 50% transparent
- your default tool should be the pencil tool with black as color and a size of around 30-100 pixels. Use it to mask out offending areas on the lower layer (the actual mask), refer back to VC3D if you are unsure.
- save the mask by clicking "File->overwrite mask.tif"
With this process a mask can be generated using less than 10 clicks. This is the main annotation method used for the submission (due to the fast iteration time) at a total of 243 masks generated which should take about 2 hours.

**approved**
Checking the approved checkbox will mark a patch as manually "approved" which means the tracer will consider mesh corners coming from such a patch as good without checking for other patches. So it is important that such a patch is error free (which is not necessary when creating a mask to only remove a problem area without checking "approved").
So the whole patch needs to be checked or potentially problematic areas need to be masked out generously, as any error in an approved patch will translate 1:1 to an error in the final trace. For this reason this feature was used sparingly for the submission and only used where otherwise the trace would not continue at all.
If a whole patch needs to be checke the following process was used:
- ctrl-click on a point in the area that shall be "correct".
- follow along the two segment slices and place blue POIs every at an interval whereever you are sure the trace follows the correct sheet.
- place red POIs where errors occur
this process will place a "cross" of two lines which show the good/bad areas of the patch
- ctrl-click to focus on a point on/between blue points to genrate a second line, this way a whole grid of points is genrated
- use this grid as orientation to create a mask in GIMP
This process could take anywhere from 5-30minutes, therefor it was used very sparingly for the submission (6 times) and not to generate a large approved patch but just to bridge a bad spot, areas that weren't necessary to bridge a gap were simply not checked and masked out to save time.
So generate these approved patches took around 1h in total for the submission.

## 5. Fusion & flattening of large traces

Given the above iterative process provides a set of larger (or shorter) traces that can be combined to generate a final fused and flattened output.
For the fusion process the traces need to be error free, you can use the same masking approach from VC3D to quickly and generously mask out error areas.
Traces in both directions (inward and outward) can be combined.

## 5.1. winding number assignment

First step in the fusion process is generating relative winding numbers of each trace by running:
```
OMP_WAIT_POLICY=PASSIVE OMP_NESTED=FALSE \
vc_tifxyz_winding /path/to/trace
```
Which will generate some debug images and the two files "winding.tif" and "winding_vis.tif".
Check the winding vis for errors, it should just be a smooth continous rainbow going from left to right, if it isn't there were some errors in the source trace that weren't masked out. Mask those errors in the source trace and re-run winding estimation until it works, this should not generally be necessary.

Copy the winding.tif and wind_vis.tif to the traces storage directory so its all in one place and ready for the next step.
The winding estimation should take about 10s.

## 5.2. joint fusion and inpainting

The traces (and even the patches) generated in the previous steps could directly be used for ink detection, and for debugging and fast iterations this should definitely be done. However a high qality and filled surface can be achived by running
```
OMP_WAIT_POLICY=PASSIVE OMP_NESTED=FALSE time nice \
vc_fill_quadmesh params.json /path/to/trace1/ /path/to/trace1/winding.tif 1.0 /path/to/trace2/ /path/to/trace2/winding.tif 1.0
```
with an arbitrary number of traces. Note that the first trace will be used as the seed an it will also define the size of the output trace and will generate normal constraints, so it should be the longest and most complete. The number after the trace is the weight of the trace when generating the surface, in all test a weight of 1.0 was used and for the submission this params.json:
```
{
    "trace_mul" : 5,
    "dist_w" : 1.5,
    "straight_w" : 0.005,
    "surf_w" : 0.05,
    "z_loc_loss_w" : 0.002,
    "wind_w" : 10.0,
    "wind_th" : 0.5,
    "inpaint_back_range" : 60,
    "opt_w" : 4
}
```
Takes around 40 minutes:
```
13653.19user 118.87system 39:56.61elapsed 574%CPU (0avgtext+0avgdata 1945672maxresident)k
147992inputs+2594872outputs (9major+8252759minor)pagefaults 0swaps
```
Note that traces that differ in direction _can_ be used, the winding estimator will automatically flip the input winding number and offset it so the different traces align.

## 6. rendering

Using vc_render_tifxyz to render 21 layers from the trace generated in the last step at half scale from the half scale ome-zarr:
```
OMP_WAIT_POLICY=PASSIVE OMP_NESTED=FALSE time nice \
vc_render_tifxyz /path/to/volume/ome-zarr /output/path/%02d.tif /path/to/trace 0.5 1 21
```

Which takes about half an hour:
```
4629.46user 12054.06system 31:08.21elapsed 893%CPU (0avgtext+0avgdata 13987244maxresident)k
47117424inputs+17606296outputs (2major+76947037minor)pagefaults 0swaps
```

## 7. ink prediction

The ink detection is using the accelerated version documented [here]() at a slightly reduced quality to make processing times bearable, and is using the default settings of:
- half resolution rendering (from half scale ome-zarr subdirectory)
- using 21 layers
```
time python fast_inference_timesformer.py --layer_path /path/to/rendered-slices --model_path timesformer_wild15_20230702185753_0_fr_i3depoch=12.ckpt --out_path fasp.jpg --quality 1 --compile --reverse
```

Which yields the final ink detection image at fasp.jpg.
```
real    29m30.555s
user    47m1.576s
sys     18m55.119s
```

If no ink is being detected maybe the layer direction needs to be flipped which can be achived with the --reverse flag.

# Installation and Dependencies

## Surface Volume Predicitions

Please refer to *Segmenting_Scroll_Surfaces.pdf*.

## VC3D & tracing tools
The code is available at: https://github.com/hendrikschilling/volume-cartographer under the dev-next branch.
Please refer to the refer to the "jammy.Dockerfile" to see a list of all dependencies or to build a docker image based on ubuntu 22.04.
In addition it is recommended to use a git version of ceres-solver together with a [Nvidia cudss](https://developer.nvidia.com/cudss) installation to accelerate the large area tracing and the inpainting/flattening.

## Ink detection

Ink detection is based on the original GP ink detection (which could also be used instead).
The fork is found at https://github.com/hendrikschilling/Vesuvius-Grandprize-Winner and installation requirements are unchanged.

# Misc

## HW/SW configuration
All steps above following the volume surface prediction were achieved using:
- AMD 5950x
- Nvidia 3090
- 64GB DDR4 RAM + 256GB swap
- Transcend TS2TMTE220S SSD
- a collection of HDDs in a BTRFS RAID1
SW environment:
- Cuda 12.7
- cudss 12.5.4
- pytorch 2.5.4
- Arch Linux