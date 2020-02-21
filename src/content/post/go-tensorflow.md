+++
date = "2017-07-03T15:45:58+01:00"
draft = false
title = "Using your tensorflow model with go"
image = "tensorflow-logo.jpg"
tags        = [ "go", "tensorflow", "inference"]
+++


This post will serve as a simple end-to-end example of how to 
use your own tensorflow-model to do inference in your go-application.
 You will need to train your own model with tensorflow in order to make it work properly.

If you are doing inference in java (or any other language)
the blogpost will still be useful since the principles are the 
same for languages with bindings to tensorflow. 

## TLDR;
Name your tensors and operations in the tensorflow graph before exporting the model. 
Use the same names to address input and inference-operations when you want to use the model in go(or any other language).
Be careful to match the dimensions of your input and output when using the model.

## Motivation

The tensorflow library is written in c++, but python is the most used language to use it. But since I am a 
go-developer I want to be able to use my tensorflow models in-process
to do inference. This way I can use inference as a part of my program or expose it as a 
rest-service if needed and hopefully save some milliseconds over python when doing inference. 


## Pre reqs

To execute the code in this example you will need to

* have tensorflow installed
* have some version of go installed(I currently use 1.8)
* some familiarity with tensorflows compute-graph if you run into problems


## Install go-bindings

To get started with go, you need to install the go-bindings: 

    TF_TYPE="cpu" # Change to "gpu" for GPU support
    TARGET_DIRECTORY='/usr/local'
    curl -L \
       "https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-${TF_TYPE}-$(go env GOOS)-x86_64-1.1.0.tar.gz" |
    sudo tar -C $TARGET_DIRECTORY -xz

    sudo ldconfig

    go get github.com/tensorflow/tensorflow/tensorflow/go
    go test github.com/tensorflow/tensorflow/tensorflow/go


(if any of the above fails, find out why on https://www.tensorflow.org/install/install_go )


## Name your tensors and operations before training

To be able to address specific parts of your tensorflow-graph during inference, 
you need to **give names to the tensors and operations** that you are interested in. 
In this case, it is the input-tensor and the inference-step I want to address.
 
Here I have named my input-tensor "imageinput" and the inference-step for "infer"

    # the input-tensor
    x = tf.placeholder(tf.float32, [None, 784], name="imageinput")

    # the infer operation
    infer = tf.argmax(y,1, name="infer")

If you are interested in the result of other tensors or operations, be sure to name them as well. For instance, you will probably be interested in 
the accuracy of each inference, so go ahead and give the accuracy-operation a proper name.

## Save your model with a tag

You need to build **a save-builder with a tag** before saving your trained model. This is to fetch the correct graph when reading the graph from go.

    builder = tf.saved_model.builder.SavedModelBuilder("export2")
    
    builder.add_meta_graph_and_variables(sess,["serve"])
    
    builder.save()


For the complete python-code for this example, have a look at [the code on github](https://github.com/nilsmagnus/tensorflow-with-go/blob/master/mnist.py).

## Using the model in go

Import the tensorflow bindings as tf
    
    import (
    	"fmt"
    	tf "github.com/tensorflow/tensorflow/tensorflow/go"
    )

Now , load the model with the tensorflow binding function *tf.LoadSavedModel(...)*. Use the same tag-name as you specified when saving the model:    
    
    func main() {
    
    	model, err := tf.LoadSavedModel("mnistmodel", []string{"serve"}, nil)
    
    	if err != nil {
    		fmt.Printf("Error loading saved model: %s\n", err.Error())
    		return
    	}
    
    	defer model.Session.Close()
    
The  input in the example code is just dummy-input with a blank matrix. Be sure to match the dimensions of the input you defined in your trained model. In this case I will define a 2-d matrix of type float32 with dimensions 1x784 since the mnist-images are 784 pixels. 
    
    	tensor, terr := dummyInputTensor(28 * 28) // replace this with your own data
    	if terr != nil {
    		fmt.Printf("Error creating input tensor: %s\n", terr.Error())
    		return
    	}

Now, this is the most important part: executing the tensorflow graph and getting the results. 
In order to use the graph, we need to **address the input-tensor and the operation** to instruct tensorflow where to put our input and what operations in the graph we want to execute. 
In  this case, i have named my input **inputimage** and my inference-operation **infer**. If everything works fine, the result(s) is returned
to the **result** variable.  
    
    	result, runErr := model.Session.Run(
    		map[tf.Output]*tf.Tensor{
    			model.Graph.Operation("imageinput").Output(0): tensor,
    		},
    		[]tf.Output{
    			model.Graph.Operation("infer").Output(0),
    		},
    		nil,
    	)
    
    	if runErr != nil {
    		fmt.Printf("Error running the session with input, err: %s\n", runErr.Error())
    		return
    	}

Again, the dimension of the result is the same as defined in your tensorflow model, so you might want to tweak this part. 

The result in the **infer-operation** is a matrix with 1 value:
    
    	fmt.Printf("Most likely number in input is %v \n", result[0].Value())
    
    }

## Example code

The code for both training and inference can be found in its completeness on github at
https://github.com/nilsmagnus/tensorflow-with-go

## Similar blogposts

I have not found any documentation or end-to-end example of what I have described here, but there is a nice post about working with golang and tensorflow in general here:
https://pgaleone.eu/tensorflow/go/2017/05/29/understanding-tensorflow-using-go/?utm_source=golangweekly&utm_medium=email
