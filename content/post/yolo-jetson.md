+++
date = "2020-01-21T07:44:01+01:00"
draft = false
title = "Using YOLO models on nvidia jetson"
tags  = ["jetson-nano", "yolo", "opencv", "cuda"]
+++


[YOLO](https://pjreddie.com/darknet/yolo/) is a highly optimized machine-learning model to recognize objects in videos and images. Running these on your jetson nano is a great test of your board and a bit of fun. To use these awesome models you need to install darknet, the program that runs interference on a video stream from your camera. 

What you need:

* Jetson Nano Developer Kit with Jetson Nano Developer Kit SD Card image installed
* An USB web-camera

Since we have a CUDA gpu on the jetson we should modify the Makefile to enable CUDA and make sure that nvcc is on `$PATH`. We also want to use `OpenCV` to make use of our webcam later on. 

	# cuda/bin is not on PATH by default, add it to .bashrc
	echo "export PATH=$PATH:/usr/local/cuda/bin" >> ~/.bashrc
	source ~/.bashrc
	


Now get the source for darknet

    git clone https://github.com/AlexeyAB/darknet


Now edit the `Makefile`, changing `GPU=0` to `GPU=1`, `CUDNN=0` to `CUDNN=1` and `OPENCV=0` to `OPENCV=1` and build

    cd darknet		
    make
    
Download the YoloV3 and run the model

	# Download the pretrained yolo3 model
	wget https://pjreddie.com/media/files/yolov3.weights
	
	# run the model 
	./darknet detector demo cfg/coco.data cfg/yolov3.cfg yolov3.weights
	

Voila

<iframe src="https://player.vimeo.com/video/392921272" width="640" height="497" frameborder="0" allow="autoplay; fullscreen" allowfullscreen></iframe>
	
