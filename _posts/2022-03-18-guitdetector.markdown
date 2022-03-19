---
layout: post
title: "GuitDetector - Computer Vision Guitar 2D Pose Tracking"
---
# GuitDetector - Computer Vision Guitar 2D Pose Tracking
This project was completed by Kirit Narain in the span of approximately two weeks. The code for the IPython notebook can be found here: <a href="https://github.com/kiritnarain/GuitDetector">https://github.com/kiritnarain/GuitDetector</a>

## Abstract
GuitDetector is a Computer Vision project to track the 2D pose of a physical guitar using a single camera. The pipeline uses a transfer learning approach by modifying a pretrained Faster RCNN object detection model to predict the position of guitar inlays (indicator circles on a Guitar fretboard). The predicted guitar inlay positions are then passed through a RANSAC-like algorithm to compute the holography transform from a static reference frame to image coordinates. Finally, the computed homography is used to visualize markers along a guitar fretboard.

<div class="d-flex justify-content-center" style="margin-bottom: 50px">
<iframe width="560" height="315" src="https://www.youtube.com/embed/gEBkWAxGnD4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Problem and Goals
Beginner and Intermediate Guitar players often struggle with learning new songs. Guitarists need to juggle looking at music sheets and then translate notes into finger positions over a guitar fretboard, which usually requires hours of practice and memorization for every new melody.

This project seeks to place finger markers on the guitar fretboard itself, unlocking the possibility of simply picking up a guitar and playing a new song with zero practice or memorization required.

It is a follow-up to [GuitXR](https://uwrealitylab.github.io/xrcapstone21sp-team4/), a project I worked on last year using Augmented Reality for similar Guitar note visualization. The primary challenge with GuitXR was the need for external tracking equipment to map the physical guitar pose.

This project instead is targeted at using Computer Vision to track the Guitar in space without any external equipment or markers, resulting in a far more accessible and convenient solution using a single camera.

## Dataset
I annotated a small custom dataset for this task. I captured videos of my guitar held in different positions and orientations (including videos of me playing songs) and split them into image frames (.jpg files with 1280x720p resolution). I then used the [Hasty.AI](https://hasty.ai/) object detection labeling pipeline to annotate bounding boxes around the guitar inlays for 44 images manually. Hasty.AI included annotation assistance tools which enabled me to quickly label an additional 440 images with minor tweaks, giving a total dataset size of 484 annotated images (most images had 6 boxes). 

I then used [this tool](https://github.com/akarazniewicz/cocosplit) to create an 80-20 train/test split.

## Techniques

#### Object Detection Model
1.	Data Loading: I wrote a dataset loader class from scratch to load the images and annotations metadata from .json files. This involved converting the image color to RGB from BGR (using the cv2 library), and to reduce training time, I additionally scaled the images to be 512x512 and applied the same scaling to the bounding boxes. The train data additionally had random flips, rotations, and blurring applied for introducing variations in the data.

2.	Model and parameters – A Faster R-CNN model with a ResNet-50-FPN backbone was loaded from torchvision which was pretrained on the MS COCO 2017 dataset. The last box predictor layer was swapped out with a 2 class predictor to predict ‘background’ or ‘inlay’ classes.
The model was trained for 20 epochs with a 0.001 base learning rate and using Stochastic Gradient Descent as the optimization function. Additionally, a learning rate scheduler was implemented to reduce the LR by a factor of 10 every 5 epochs. A momentum of 0.9 was also used with a decay of 0.001 for regularization. I filtered the output boxes to have a score >50%.
      
This resulted in a final training loss of  0.2163
      
![Training Loss](/assets/img/guitdetector-loss-graph.jpg)



I chose not to use standard object detection metrics (like IOU or mAP) for this project because only the bounding box center was needed for 2D pose estimation (and the annotation process was not very accurate in terms of area as the objects are circular). Instead for each image, the predicted center of each inlay was compared with the true inlay center with a 5px threshold. This resulted in the following accuracy measurements:

Train Accuracy: 96.90%

Test Accuracy: 96.91%

#### 2D Pose and Homography
A key challenge of this project was estimating the 2D pose of the guitar from the predicted 6 inlay positions. Since the real-world distance between inlays was known, I positioned them on a static reference frame, and then implemented an algorithm similar to [RANSAC](https://en.wikipedia.org/wiki/Random_sample_consensus) to try to randomly match the predicted boxes with the positions in the static frame. For each match, the homography transformation was computed and the number of inliers calculated, and the homography with the maximum inliers (or inliers >= threshold) selected.
For optimization and because the state space is small (720 possible permutations of the 6 inlays), a brute force approach could be used instead of random matchings.
To draw visualizations over the guitar fretboard, I created a set of PNG images with alpha channels aligned according to the static reference frame defined earlier. The pixels could then be transformed with the computed homography (ignoring pixels with alpha set to 0) and pasted on the guitar fretboard image.

#### Discussion and Future Work
An immediate next step for this project is to annotate a larger dataset containing a variety of different guitars. Additionally more guitar features could also be annotated and detected, such as the silver fret markers to enable robustness in case some markers are occluded. The current implementation only works well when all 6 inlays are detected and matched (otherwise the homography transform collapses space onto a single line).

Another future task is to apply the same process to real-time video instead of images, so guitarists can visualize the notes live as they play. Some groundwork optimization has already been done which accomplished reducing the processing time from 10s to approximately 2s per image (by performing a single homography matrix multiplication for all reference frame pixels together, and other vectorizations), however to support a real-time capable framerate this will need to be further improved. An area partially explored was attempting to use pooling/multithreading to trial the different matchings which showed promise, but I ran out of time to complete that.

## References and Attribution
<i>The model scaling, transformation and training steps were based on a combination of the following sources with some modifications. </i>
1.	Rath, Sovit Ranjan. “Custom Object Detection using PyTorch Faster RCNN.”  Debugger Café, October 25, 2021, [https://debuggercafe.com/custom-object-detection-using-pytorch-faster-rcnn/](https://debuggercafe.com/custom-object-detection-using-pytorch-faster-rcnn/)
2.	Rajaa, Shangeth et al. “Faster R-CNN Object Detection with PyTorch.” LearnOpenCV, June 18, 2019, [https://learnopencv.com/faster-r-cnn-object-detection-with-pytorch/](https://learnopencv.com/faster-r-cnn-object-detection-with-pytorch/)
3.	Thakur, Abhishek. “training fast rcnn using torchvision.” Kaggle, 2019, [https://www.kaggle.com/code/abhishek/training-fast-rcnn-using-torchvision/notebook](https://www.kaggle.com/code/abhishek/training-fast-rcnn-using-torchvision/notebook)
4.	Redmon, Joseph. “CNNs in Pytorch.” University of Washington Computer Science and Engineering, 2022, [https://colab.research.google.com/drive/1k4SpEurwjaG3a4AAIQraDFg_7Qa1Cat5](https://colab.research.google.com/drive/1k4SpEurwjaG3a4AAIQraDFg_7Qa1Cat5)

<i> Tools used:</i>

1. [Coco Split](https://github.com/akarazniewicz/cocosplit)

2. [Hasty.AI](https://hasty.ai/)

<i>This project was completed as part of the University of Washington Computer Vision course in Winter 2022, and is based on material covered in the class. [https://courses.cs.washington.edu/courses/cse455/22wi/](https://courses.cs.washington.edu/courses/cse455/22wi/) </i>

