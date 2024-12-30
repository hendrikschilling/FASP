# Intro

This repo serves to document a submission for the first automated segmentation price of the vesuvius challenge 2024.
It serves as a documentation on how to run the full automated segmentation pipeline and a decription of the algorithms used.

# Links & Repos

# Input Data
- TODO src 16bit ome-zarr

# Submission Data

# Tools

# Overview

The overall process consists of the following steps:
1. surface prediction volume generation
2. patch gen seeding
3. patch gen expansion
4. iterative surface tracing & annotation
5. fusion & flattening of large traces
6. rendering
7. ink prediction

# Short Guide
This short guide serves to document the steps taken to achieve the FASP submission without diverting too much into all the options the tools provide.

## 1. Inference
## 1.5 Data prep
data prep
{"height":7888,"max":65535.0,"min":0.0,"name":"043_044_050_random_ensemble_otsu_ome.zarr/","slices":14376,"type":"vol","uuid":"043_044_050_random_ensemble_otsu_ome.zarr/","voxelsize":7.91,"width":8096, "format":"zarr"}
## 2. patch gen seeding
The patch seeding will trace small (4cm^2 by default) patches from the surface prediction provided as ome-zarr. Detailed documentation is available [in this thread](https://discord.com/channels/1079907749569237093/1312490723001499808).
The command used for the submission is:

~~~
time seq 1 1000 |  xargs -i -P 32 bash -c 'nice ionice vc_grow_seg_from_seed /path/to/gp-prediction-ome-zarr /path/to/patch-collection params_grow_seed.json'
~~~

params_grow_seed.json:
~~~
{
    "cache_root" : "/path/to/cache",
    "generations" : 200,
    "min_area_cm" : 0.3,
    "thread_limit" : 1
}
~~~

runtime:
~~~
real    52m51.294s
user    1487m27.790s
sys     35m10.965s
~~~

This process generated an initial set of 775 patches.

The patches can be inspected with VC3D by placing them in the paths directory of the volpkg. Symlinks can also be used.

## 3. patch gen expansion
The patche expansion step also documented [here](https://discord.com/channels/1079907749569237093/1312490723001499808) generates a set of overlapping patches throughout the whole gp-prediction volume.

The commands used where: 

~~~
time seq 1 10000000 |  xargs -i -P 32 bash -c 'nice ionice vc_grow_seg_from_seed /path/to/gp-prediction-ome-zarr /path/to/patch-collection params_expand_overlap.json'
time seq 1 10000000 |  xargs -i -P 16 bash -c 'nice ionice vc_grow_seg_from_seed /path/to/gp-prediction-ome-zarr /path/to/patch-collection params_expand_overlap.json'
time seq 1 1000000000 |  xargs -i -P 16 bash -c 'nice ionice vc_grow_seg_from_seed /path/to/gp-prediction-ome-zarr /path/to/patch-collection params_expand_overlap.json'
~~~

with params_expand_overlap.json:
~~~
{
    "cache_root" : "/path/to/cache",
    "tgt_overlap_count" : 10,
    "generations" : 200,
    "min_area_cm" : 0.3,
    "thread_limit" : 1,
    "mode" : "expansion"
}
~~~

this resulted in a growing number of patches:
~~~
7817
10444
17980
~~~

And the following runtimes:
~~~
real    259m36.492s
user    7128m20.478s
sys     257m29.490s

real    123m11.015s
user    1753m25.029s
sys     110m42.635s

real    215m33.051s
user    2890m24.039s
sys     346m59.649s
~~~

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
~~~
time OMP_WAIT_POLICY=PASSIVE OMP_NESTED=FALSE nice vc_grow_seg_from_segments /path/to/prediction-volume /path/to/patch-collection /path/to/output params.json /path/to/seed/patch
~~~

Using for the finall passes these settings:
~~~
{
        "flip_x" : false,
        "global_steps_per_window" : 3,
        "step" : 10,
        "consensus_default_th" : 10,
        "consensus_limit_th" : 2
}
~~~
- flip_x determines the direction of the trace (it always grows to the right, but that can go to the inside or outside of the scroll, depending on seed location).
- global steps per window: numer of global optimization steps per moving window. The tracer operates in a moving window fasion, once the global optimization steps were run per window and no new corners were added the window is moved to the right and the process repeated. At the beginning use 0 global steps to get a fast and long trace and see if there are any coarse errors.
- consensus_default_th: lowest number of inliers (patch "supports") required per corner to be considered an inlier. Note that a single well connected patch gives more than a single support to a corner (so this is not the number of surfaces). Maximum of 20 to get only well connected patches, minimum of 6 before there are lot of errors.
For the submission values of 6 and 10 were used.
- consesus_limit_th: if we could otherwise not proceeed go down to this number of required inliers, this will only be used if otherwise the trace would end.

The tracer will generated debug/preview surface every 50 generations (labeled z_dbg)


### 4.3. check for errors
### 4.4. annotation
## 5. fusion & flattening of large traces
## 5.1. winding number assignment
## 5.2. joint fusion and inpainting

PATHD=/home/hendrik/data/ml_datasets/vesuvius/manual_wget/dl.ash2txt.org/full-scrolls/Scroll1/PHercParis4.volpkg/paths ; OMP_WAIT_POLICY=PASSIVE OMP_NESTED=FALSE time nice bin/vc_fill_quadmesh ../docs/reference_settings/params_infill_fast_5x.json /home/hendrik/data/ml_datasets/vesuvius/manual_wget/dl.ash2txt.org/full-scrolls/Scroll1/PHercParis4.volpkg/paths/ref_opt2_1x-2024-12-20_7164/ /home/hendrik/data/ml_datasets/vesuvius/manual_wget/dl.ash2txt.org/full-scrolls/Scroll1/PHercParis4.volpkg/paths/ref_opt2_1x-2024-12-20_7164/winding.tif 1.0 /home/hendrik/data/ml_datasets/vesuvius/manual_wget/dl.ash2txt.org/full-scrolls/Scroll1/PHercParis4.volpkg/paths/ref_someerrors_winddetect_03926/ /home/hendrik/data/ml_datasets/vesuvius/manual_wget/dl.ash2txt.org/full-scrolls/Scroll1/PHercParis4.volpkg/paths/ref_someerrors_winddetect_03926/winding.tif 1.0 /home/hendrik/data/ml_datasets/vesuvius/manual_wget/dl.ash2txt.org/full-scrolls/Scroll1/PHercParis4.volpkg/paths/ref_2024-12-25_opt6_rev_th6_14732/ /home/hendrik/data/ml_datasets/vesuvius/manual_wget/dl.ash2txt.org/full-scrolls/Scroll1/PHercParis4.volpkg/paths/ref_2024-12-25_opt6_rev_th6_14732/winding.tif 1.0 $PATHD/ref_opt10_2024-12-29-rev-outer_th10_step10_03766/ $PATHD/ref_opt10_2024-12-29-rev-outer_th10_step10_03766/winding.tif 1.0 $PATHD/ref_opt12_fw_holes_s10_th10_2024-12-30_gen_02771/ $PATHD/ref_opt12_fw_holes_s10_th10_2024-12-30_gen_02771/winding.tif 1.0

13653.19user 118.87system 39:56.61elapsed 574%CPU (0avgtext+0avgdata 1945672maxresident)k
147992inputs+2594872outputs (9major+8252759minor)pagefaults 0swaps

## 6. rendering
## 7. ink prediction

The ink detection is using the accelerated version documented [here]() at a slightly reduced quality to make processing times bearable, and is using the default settings of:
- half resolution rendering (from half scale ome-zarr subdirectory)
- using 21 layers
~~~
time python fast_inference_timesformer.py --layer_path /path/to/rendered-slices --model_path timesformer_wild15_20230702185753_0_fr_i3depoch=12.ckpt --out_path fasp.jpg --quality 1 --compile --reverse
~~~

Which yields the final ink detection image at fasp.jpg.
~~~
real    29m30.555s
user    47m1.576s
sys     18m55.119s
~~~

# Installation and Dependenceis
## Surface Prediction
## VC3D

# Detailed Guide

## 1. surface prediction volume generation

## 2. ....


# proessing times
- surface tracer: at step 10 1/3 of GP segment at 3 optimization steps: 55minutes

# tradeoffs
- patch expansion run only with 16 instead of 32 threads to have capacity for other tasks at that time
- ink detecton accuracy (q1)
- rendering resoution (also ink detection accuracy)

# misc

## HW
all results above apart from the surface volume predicitions were achieve using:
- AMD 5950x
- Nvidia 3090
- 64GB RAM
- 

13653.19user 118.87system 39:56.61elapsed 574%CPU (0avgtext+0avgdata 1945672maxresident)k
147992inputs+2594872outputs (9major+8252759minor)pagefaults 0swaps