---
layout: post
title: Use LibSVM in OpenCV
description: "Using libsvm in OpenCV 101."
modified: 2016-11-18
tags: [C++, OpenCV, SVM]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

This post will elaborate how to use libsvm in OpenCV for image classification.

>Image analysis always comes down to classification, and classification always comes down to svm.

## Why use libsvm instead of CvSVM?

When people try to use svm with OpenCV, they always first come to OpenCV's built-in svm class names **CvSVM**. However, CvSVM is developed based on a very old version of [libsvm](http://www.csie.ntu.edu.tw/~cjlin/libsvm/). I am not sure how much difference it makes, but it definitely lacks a very important feature: when making prediction, it only returns the class with highest probability, not the probability of all classes.

My above statement maybe not precise enough. CvSVM does provide a probability-alike prediction with a flag -- but only for 2-class classification ([CvSVM::predict](http://docs.opencv.org/2.4/modules/ml/doc/support_vector_machines.html#cvsvm-predict)). But mostly, people need multi-class classification, and we want probabilities for each class because we need to do some post-prediction analysis as the original prediction may not be accurate enough. After some research about tuning CvSVM, it comes to conclusion that CvSVM is incapable of that. The only solution comes to using libSVM.

## How to use libsvm offline with OpenCV

I will only explain the very basic use case here.
*Prerequisite*: 
>download libsvm source code, compile it (`$ make`), to obtain executables (`svm-predict.exec`, `svm-scale.exec`, `svm-predict.exec`).

**LIBSVM** is mostly for command line use. To use it, we need two files: one is the training set, and one is the test data.
First step is to construct the training file. A training data file is a txt file, with N lines of data. Each line represents a training vector. The structure is like this:

```
${classLabel} ${attributeIndex}:${attribute} (repeat)   
```

For example, if vector `[0.3, 0.5, 0.7]` belongs to class 2, it's represented in the file as

```
2 1:0.3 2:0.5 3:0.7
```

For some reasone I forgot, the attributeIndex starts from 1 instead of 0.
libsvm is designed to accommodate sparse matrix, so value zero can be skipped. It means a vector of `[0, 0, 0.3, 0, 0, 0.5]` in class 1 is represented as 

```
1 3:0.3 6:0.5
```

After the training matrix is saved as `trainingData`, we need to train a model. Using the following command

```
$ ./svm-train -b 1 trainingData
```

There are some parameters we can set, but here we will use default for all except `-b`, which is `probability_estimates: whether to train a SVC or SVR model for probability estimates, 0 or 1 (default 0)`. This is exactly what we want from libsvm.

Then we will get the svm model `trainingData.model`. You can use `svm-scale` to scale it, but that is not our focus now.

Then create the vector to be predicted. It has the same format as in training vectors. The first value ${classLabel} is not important. But we can use it for verification. If the class label is the same as the result of prediction, it means a successful prediction. So basically we can built a test set, to verify how well the trained model works. Furthermore, we can use this test set to iterate training process to obtain the best svm parameter.

Suppose the test vector file name is `testData`. To predict, run

```
$ svm-predict -b 1 testData trainingData.model testResult
```

This means to use `trainingData.model` to predict `testData` and save the result in `testResult`.
For a 5 classes classification, the `testResult` may look like this

```
3 0.1 0.1 0.9 0.3 0.4
```

This is quite self-explanatory. So the work of using libsvm with OpenCV offline is straghtforward.
1. write training data matrix into a file using libsvm format
2. run command to train the model
3. write target vector to file using libsvm format
4. run command to predict the class
5. read the result file and process

This is ok if you don't do prediction that frequently. However, if you use it a lot, as in video processing, it is still far away from feasible. We need some C++ code to do the prediction.

## Run svm-predict in your code

If you google `libsvm` and `opencv`, you will probably end up with [this blog](http://kuantinglai.blogspot.ca/2013/07/using-libsvm-with-opencv-mat.html).
This is a very good one. It inspired me how to make it work, though its code is not exactly what I want. This code essentially loads the model and target vector from files, then processes them. We just need to make some modification so we don't need to load target vector, but create the targe vector in code directly. Kuanting's code illustrated how the target vector in code should look like.

So in my code, I need to predict class by a histogram vector. My prediction function looks like this:

```
#include "svm.h"
...
void predict(string modelPath, Mat& hist) {

    const char *MODEL_FILE = modelPath.c_str();
    if ((this->SVMModel = svm_load_model(MODEL_FILE)) == 0) {
        this->modelLoaded = false;
        fprintf(stderr, "Can't load SVM model %s", MODEL_FILE);
        return;
    }

    struct svm_node *svmVec;
    svmVec = (struct svm_node *)malloc((hist.cols+1)*sizeof(struct svm_node));
    int j;
    for (j = 0; j < hist.cols; j++) {
        svmVec[j].index = j+1;
        svmVec[j].value = hist.at<float>(0, j);
    }
    svmVec[j].index = -1; // this is quite essential. No documentation.

    double scores[8]; // suppose there are eight classes
    if(svm_check_probability_model(SVMModel)) {
        svm_predict_probability(SVMModel, svmVec, scores);
    }
}
```
Then `scores` will save the prediction probability of all 8 classes.

## What can be improved

1. since we already know how to construct the target vector, it should be easy to build the training vectors, which means we can do the training directly in code instead of save to file -> run svm-train.
2. libsvm's default parameter of course doesn't guarantee good result. Grid-search can be done to find the best parameters, but I haven't tried yet.