---
layout: default
title: Boosting classifier accuracy by grouping categories
permalink: /tutorials/Boosting-classifier-accuracy-by-grouping-categories/
---
# Boosting classifier accuracy by grouping categories

In this tutorial, we will split the 1000 image-categories, which the model was trained to classify, into three disjoint sets: *dogs*, *cats*, and *other* (anything that isn't a dog or a cat). We will demonstrate how a classifier with low accuracy on the original 1000-class problem can have a sufficiently high accuracy on the simpler 3-class problem. We will write a Python script that reads images from the camera, barks when it sees a dog, and meows when it sees a cat.

---

[![screenshot](/ELL/tutorials/Boosting-classifier-accuracy-by-grouping-categories/thumbnail.png)](https://youtu.be/SOmV8tzg_DU)

#### Materials

* Laptop or desktop computer
* Raspberry Pi
* Headphones or speakers for your Raspberry Pi
* Raspberry Pi camera or USB webcam
* *optional* - Active cooling attachment (see our [tutorial on cooling your Pi](/ELL/tutorials/Active-cooling-your-Raspberry-Pi-3/))

#### Prerequisites

* Install ELL on your computer ([Windows](https://github.com/Microsoft/ELL/blob/master/INSTALL-Windows.md), [Ubuntu Linux](https://github.com/Microsoft/ELL/blob/master/INSTALL-Ubuntu.md), [Mac](https://github.com/Microsoft/ELL/blob/master/INSTALL-Mac.md)). Specifically, this tutorial requires ELL, CMake, SWIG, and Python 3.6.
* Follow the instructions for [setting up your Raspberry Pi](/ELL/tutorials/Setting-up-your-Raspberry-Pi).
* Complete the basic tutorial, [Getting started with image classification on Raspberry Pi](/ELL/tutorials/Getting-started-with-image-classification-on-the-Raspberry-Pi/), to learn how to produce a Python wrapper for an ELL model.

## Overview

The pre-trained models in the [ELL gallery](/ELL/gallery/) are trained to identify 1000 different image categories (see the category names [here](https://github.com/Microsoft/ELL-models/raw/master/models/ILSVRC2012/categories.txt)). Often times, we are only interested in a subset of these categories and we don't require the fine-grained categorization that the model was trained to provide. For example, we may want to classify images of dogs versus images of cats, whereas the model is actually trained to distinguish between 11 different varieties of cat and 106 different varieties of dog.

The dogs versus cats classification problem is easier than the original 1000 class problem, so a model that isn't very accurate on the original problem may be perfectly adequate on the simpler problem. Specifically, we will use a model that has an error rate of 64% on the 1000-class problem, but only 5.7% on the 3-class problem. We will build an application that grabs a frame from a camera, plays a barking sound when it recognizes one of the dog varieties, and plays a meow sound when it recognizes one of the cat varieties.

## Step 1: Deploy a pre-trained model on a Raspberry Pi

Start by repeating the steps of the basic tutorial, [Getting Started with Image Classification on Raspberry Pi](/ELL/tutorials/Getting-started-with-image-classification-on-the-Raspberry-Pi/), but replace the model suggested in that tutorial with [this faster and less accurate model](https://github.com/Microsoft/ELL-models/raw/master/models/ILSVRC2012/d_I160x160x3NCMNCMNBMNBMNBMNBMNC1A/d_I160x160x3NCMNCMNBMNBMNBMNBMNC1A.ell.zip). Namely, download the model, use the `wrap` tool to compile it for the Raspberry Pi, copy the CMake project to the Pi, and build it. After completing these steps, you should have a Python module on your Pi named `model`.

Copy the following files to your Pi.
- [dogs.txt](/ELL/tutorials/Boosting-classifier-accuracy-by-grouping-categories/dogs.txt)
- [cats.txt](/ELL/tutorials/Boosting-classifier-accuracy-by-grouping-categories/cats.txt)
- [tutorialHelpers.py](/ELL/tutorials/shared/tutorialHelpers.py)

Additionally, download sound files (.wav) of a dog bark and a cat meow (for example, try this [bark](http://freesound.org/people/davidmenke/sounds/231762/) and this [meow](http://freesound.org/people/blimp66/sounds/397661/) ). Alternatively, record yourself barking and meowing. Rename the bark sound file `woof.wav` and the meow sound file `meow.wav`.

## Step 2: Write a script 

We will write a Python script that invokes the model on a Raspberry Pi, groups the categories as described above, and takes action if a dog or cat is recognized. If you just want the code, copy the complete script from [here](/ELL/tutorials/Boosting-classifier-accuracy-by-grouping-categories/pets.py). Otherwise, create an empty text file named `pets.py` and copy in the code snippets below. 

First, import the modules we need:

```python
import sys
import os
import numpy as np
import cv2
import time
import subprocess
if (os.name == "nt"):
    import winsound
import tutorialHelpers as helpers
```

Also, import the Python module for the compiled ELL model.

```python
import model
```

As in previous tutorials, define a helper functions that reads images from the camera.

```python
def get_image_from_camera(camera):
    if camera is not None:
        ret, frame = camera.read()
        if (not ret):
            raise Exception('your capture device is not returning images')
        return frame
    return None
```

Next, define helper functions that check whether a category is contained in a category list. Since categories can sometimes have more than one text description, each category label may be several strings, separated by commas. Checking whether a category label matches means checking whether any one of those elements is contained in the label, and whether any match occurs in the set.

```python
def labels_match(a, b):
    x = [s.strip().lower() for s in a.split(',')]
    y = [s.strip().lower() for s in b.split(',')]
    for w in x:
        if (w in y):
            return True
    return False

def label_in_set(label, label_set):
    for x in label_set:
        if labels_match(label, x):
            return True
    return False
```

When a prediction belonging to the dog group or the cat group is detected, we want to play the appropriate sound file. Define helper functions that play a bark or a meow.

```python
# Declare variables that define where to find the sounds files we will play
script_path = os.path.dirname(os.path.abspath(__file__))
woofSound = os.path.join(script_path, "woof.wav")
meowSound = os.path.join(script_path, "meow.wav")

def play(filename):
    if (os.name == "nt"):
        winsound.PlaySound(filename, winsound.SND_FILENAME | winsound.SND_ASYNC)
    else:
        command = ["aplay", filename]
        proc = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE, bufsize=0, universal_newlines = True)
        proc.wait()

def take_action(group):
    if group == "Dog":
        # A prediction in the dog category group was detected, play a `woof` sound
        play(woofSound)
    elif group == "Cat":
        # A prediction in the cat category group was detected, play a `meow` sound
        play(meowSound)
```

Define the main entry point and start the camera.

```python
 def main():

    # Open the video camera. To use a different camera, change the camera index.
    camera = cv2.VideoCapture(0)
```

Read the category names from `categories.txt`, the list of dog-breed categories from `dogs.txt`, and the list of cat breed categories from `cats.txt`.

```python
    categories = open('categories.txt', 'r').readlines()
    dogs = open('dogs.txt', 'r').readlines()
    cats = open('cats.txt', 'r').readlines()
```

Get the model input and output shapes and allocate an array to hold the model output. 

```python
    inputShape = model1.get_default_input_shape()

    outputShape = model1.get_default_output_shape()
    predictions = model1.FloatVector(outputShape.Size())
```

For this tutorial, we'll keep some state to ensure we don't keep taking the same action over and over for the same image. Initialize the state as follows.

```python
    lastHist = None
    significantDiff = 5000
    lastPredictionTime = 0
    headerText = ""
```

Declare a loop where we get an image from the camera and prepare it to be used as input to the model.

```python
   while (cv2.waitKey(1) == 0xFF):
        # Get an image from the camera. If you'd like to use a different image, load the image from some other source.
        image = get_image_from_camera(camera)

        # Prepare the image to pass to the model. This helper:
        # - crops and resizes the image maintaining proper aspect ratio
        # - reorders the image channels if needed
        # - returns the data as a ravelled numpy array of floats so it can be handed to the model
        input = helpers.prepare_image_for_model(image, inputShape.columns, inputShape.rows)
```

We'll use OpenCV to get a histogram using OpenCV as a quick way to detect whether the image has changed significantly. This is to create a better experience than having the same action be taken on the same prediction over and over. We'll also ensure that enough time has passed for the sound file to have fully played out.

```python
        hist = np.histogram(input,16,[0,256])[0]
        diff = 1
        if lastHist is None:
            lastHist = hist           
        else:
            diff = max(lastHist - hist)

        # Check whether the image has changed significantly and that enough time has passed
        # since our last prediction to decide whether to predict again
        now = time.time()
        if diff >= significantDiff and now - lastPredictionTime > 2:
```

It's time to call the model to get predictions.

```python
            model1.predict(input, predictions)
```

Use the helpers to get the top predictions, which is returned a list of tuples.

```python
            topN = helpers.get_top_n_predictions(predictions, 1)
```

Check whether the prediction is part of a group.

```python
            group = ""
            label = ""
            if len(topN) > 0:
                top = topN[0]
                label = categories[top[0]]
                if label_in_set(label, dogs):
                    group = "Dog"
                elif label_in_set(label, cats):
                    group = "Cat"
```

If the prediction is in one of the define category groups, take the appropriate action.

```python
            if not group == "":
                # A group was detected, so take action
                top = topN[0]
                take_action(group)
                headerText = "(" + str(int(top[1]*100)) + "%) " + group
                lastPredictionTime = now
                lastHist = hist
            else:
                # No group was detected
                headerText = ""
```

Finally, update the state if enough time has passed and display the image and header text.

```python
        if now - lastPredictionTime > 2:
            # Reset the header text
            headerText = ""

        helpers.draw_header(image, headerText)
        # Display the image using opencv
        cv2.imshow('Grouping', image)

if __name__ == "__main__":
    main()
```

## Step 3: Classify live video on the Raspberry Pi

If you followed the [Raspberry Pi Setup Instructions](/ELL/tutorials/Setting-up-your-Raspberry-Pi), you should have an anaconda environment named `py34`. Activate the environment and run the script.   

```
source activate py34
python pets.py
```

Point your camera at different objects and see how the model classifies them. Look at `dogs.txt` and `cats.txt` to see which categories the model is trained to recognize and try to show those objects to the model. For quick experimentation, point the camera to your computer screen, have your computer display images of different animals, and see when it barks or meows. If you copied the full source for [pets.py](/ELL/tutorials/Boosting-classifier-accuracy-by-grouping-categories/pets.py), you will also see the average time it takes for the model to process a single frame.

## Troubleshooting
If you run into trouble, you can find some troubleshooting instructions at the bottom of the [Raspberry Pi Setup Instructions](/ELL/tutorials/Setting-up-your-Raspberry-Pi).