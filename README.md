# Intro

This repo serves to document a submission for the first automated segmentation price of the vesuvius challenge 2024.
It serves as a documentation on how to run the full automated segmentation pipeline and a decription of the algorithms used.

# Links & Repos

# Data

# Tools

# Overview

The overall process consists of the following steps:
1. surface prediction volume generation
2. patch gen seeding
3. patch gen expansion
4. iterative surface tracing
    4.1 large surface tracing
    4.2 annotation
5. fusion & flattening of large traces
    5.1 winding number assignment
    5.2 joint fusion and inpainting 
6. rendering
7. ink prediction

# Short Guide
This short guide serves to document the steps taken to achieve the FASP submission without diverting too much into all the options the tools provide.

## 1. Inference
## 1.5 data prep
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
- pick a seed patch
- generate a trace
- check for errors
- annotated errors
### 4.1 large surface tracing
### 4.2. annotations
## 5. fusion & flattening of large traces
## 5.1. winding number assignment
## 5.2. joint fusion and inpainting
## 6. rendering
## 7. ink prediction

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


13653.19user 118.87system 39:56.61elapsed 574%CPU (0avgtext+0avgdata 1945672maxresident)k
147992inputs+2594872outputs (9major+8252759minor)pagefaults 0swaps