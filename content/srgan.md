Title: SRGAN for video
Date: 2019-12-05 09:01
Modified: 2019-12-05 09:01
Category: Technology
Tags: technology, ml, gan, srgan
Slug: srgan
Authors: Ondřej Naňka
Summary: Use SRGAN for video superresolution

# Introduction

The highly challenging task of estimating a high-resolution (HR) image from its
low-resolution (LR)counterpart is referred to as super-resolution (SR). SR
received substantial attention from within the computer vision research
community and has a wide range of applications. This post will guide you trough the process of using SRGAN for this task.

# Preprocessing

Data data data

## Cropping

First we need to crop the images to 200*200 pixels, because bigger images caused our hardware to run out of memory.
```python
from PIL import Image

from resizeimage import resizeimage

import os
for file in os.listdir("drive/My Drive/COCO_DATASET"):
    if file.endswith(".jpg"):
        with open('drive/My Drive/COCO_DATASET/'+file, 'r+b') as f:
            try:
              with Image.open(f) as image:
                  cover = resizeimage.resize_crop(image, [200, 200])
                  cover.save("drive/My Drive/COCO_DATASET_CROP2/"+file, image.format)
            except:
              pass
```

## Downscaling 
To obtain low resolution images for training we will use bicubic downscaling, see the code below.


```python
from numpy import array
from scipy.misc import imresize
# Takes list of images and provide HR images in form of numpy array
def hr_images(images):
    images_hr = array(images)
    return images_hr

# Takes list of images and provide LR images in form of numpy array
def lr_images(images_real , downscale):   
    images = []
    for img in  range(len(images_real)):
        images.append(imresize(images_real[img], [images_real[img].shape[0]//downscale, images_real[img].shape[1]//downscale], interp='bicubic', mode=None))
    images_lr = array(images)
return images_lr
```
