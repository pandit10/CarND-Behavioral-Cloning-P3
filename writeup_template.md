
# P3 - Behavioral Cloning

In this project i explored creating a steering agent for a driving simulator using convolutional neural networks. Using only pixel data generated by a virtual camera mounted on the car in the simulator and steering angles as the training signal, i demonstrate that the driving agent learns internal representations for the road in an unsupervised manner.

## Software Requirements
* python 3
* keras (with tf backend)
* tensorflow
* opencv3
* pandas


## Model Training

To train the model i used the training dataset provided by Udacity, which is a collection of images collected from a center camera , left camera and right camera mounted on the virtual car along with the corresponding steering angles, throttle and current speed of the car.
> [Udacity Dataset](https://d17h27t6h515a5.cloudfront.net/topher/2016/December/584f6edd_data/data.zip)

> Simulator [MacOS](https://d17h27t6h515a5.cloudfront.net/topher/2016/November/5831f290_simulator-macos/simulator-macos.zip) [Windows](https://d17h27t6h515a5.cloudfront.net/topher/2016/November/5831f3a4_simulator-windows-64/simulator-windows-64.zip) [Linux](https://d17h27t6h515a5.cloudfront.net/topher/2016/November/5831f0f7_simulator-linux/simulator-linux.zip)

### data exploration and augmentation
The complete process of dataset exploration and data augmentation can be evaluated by running the [Model_Training.ipynb](https://github.com/aditbiswas1/P3-behavioral-cloning/blob/master/Model_Training.ipynb) notebook. I found that the udacity dataset is highly biased towards the center and left turns and that the images contain more information than the road such as scenery.

The steps taken to help solve problems with the dataset were:

1. **Crop images:** I cropped the images 60 pixels from top until 135 pixels. I found that the top 60 pixels of the iamge are scenary information such as skies, trees etc. and the bottom 25 pixels contained the car hud. By removing the top pixels, I could create a simpler model which doesnt learn scenic features of the dataset and by removing the bottom pixels I could reuse the left / right camera images as fake center camera images to simulate recovery scenarios.

2. **Resize image:** I took the cropped images to 224x49 and resized them using the resize function available on opencv, this was done so that i could train a simpler model while preserving as much information.

3. **Reuse Left / Right Camera images:** Since the left and right cameras were always at a constant angle from the car's hud I could reuse it to simulate recovery scenarios such as when the car drives near the edge of the road. For left camera image we want the car to turn right and return to center and vice versa for the right camera, thus i augmented the steering angles for the car by  + 0.22 for left camera images and -0.22 for the right camera images

4. **Brightness Augmentation:** Since the simulation track had a constant sunlight, it was possible that the driving agent only learns color specific features in the specific light conditions, to account for this I added a random variation in brightness of the images by manipulating the V channel after converting the images to HSV color space and converting back to RGB.

5. **Horizontal Flipping** The training track had a larger number of left turns than right turns, this could make the agent biased towards taking better left turns, to account for this I augmented the entire dataset by horizontally flipping the images and inversing the corresponding steering angles.

6. **Normalization and Centering** To make the gradient descent algorithm converge to stable weights I normalized and centered the entire dataset by substracting the Mean of the RGB image stage and dividing by the standard deviation. This was done in the input layer of the model, so that the function could be used in the final model during evaluation.


### model architecture design
I carried out various experiments using starting from the architecture presented on the nvidia paper , a le-net based architecture as well using a pretrained VGG16 model as a feature extractor. After a couple of iterations the final model which i decided to use for the project was as follows:

| Layer (type)               |      Output Shape    |
|-----------------------------|----------------------|
| Convolution2D | (None, 49, 224, 3)|
| MaxPooling2D | (None, 24, 112, 3 |
| Convolution2D | (None, 24, 112, 32) |
| MaxPooling2D | (None, 12, 56, 32) |
| Convolution2D | (None, 12, 56, 64) |
| MaxPooling2D | (None, 6, 28, 64)|
| Convolution2D |  (None, 6, 28, 64) |
| MaxPooling2D | (None, 3, 14, 64) |
| Fully Connected | 512 hidden nodes |
| Fully Connected | 10 hidden nodes |
| Fully connected | 1 node |

### training
The model was trained using an adam optimizer against the loss function of mean square error since i was trying to solve the regression problem of predicting a value for the steering angle the car should take.
To ensure that the network learns to generalize I introduced a dropout layer between every weighted layer including the convolution layers, I found that using a very low dropout in the convolution layers allowed the model to generalize better and train for a longer time without overfitting to the training dataset. I finally used dropout of 0.2 for most of the convolution layers and 0.5 for the large dense layers.

I trained the neural network on the augmented dataset on an aws p2.xlarge instance for 10 epochs after trying out different training periods of 5, 10, 15. I chose to train for a long time because it was found that the validation accuracy continued to increase throughout the training time of 10 epochs and is able to complete the track 1. I could continue training the model till 25 but i found that the model starts overfitting track 1 even though training for more epochs resulted in smoother driving on track 2.

The entire training process can be retrained from scratch by executing the command
    ``` python model.py```
The resulting model was saved to model.json along with the corresponding weights in model.h5

## Evaluation

After training each model, i took the ordered images from the dataset to observe what kind of steering angles are predicted. I found that the models tend to predict a seemingly oscillating steering angles similar to the initial dataset, i found that when the model predicts smoother oscilations on the ordered images it tends to perform a bit better on the final track.

I modified the drive.py file with two things:
1. preprocessing the image to feed into the model, the function that was used for preprocessing in training was also used here.
2. adaptive throttling: initally the drive.py produces a constant throttle of 0.3, I chose to slow down the model by applying negative throttle when the car is turning, this resulted in more stable driving on the road without much swinging.
```
    if (abs(float(speed)) < 22):
        throttle = 0.5
    else:
        if (abs(steering_angle) < 0.1):
            throttle = 0.3
        elif (abs(steering_angle) < 0.5):
            throttle = -0.1
        else:
            throttle = -0.3
```

To use the model in the simulation tracks:
1. start the drive.py server using the command ```python drive.py model.json```
2. start the simulation application and select the desired track in autonomous mode. (for image quality i evaluated the model on the fastest, fast and simple modes.

A demonstration of the models can be found on the links below:

1. [test track 1 demo](https://youtu.be/M0KSysBsEJA?t=18s)
2. [test track 2 demo](https://youtu.be/KLTsNKx9Amo)

## Reflection

This was an interesting project for me because we had to strongly focus on balancing the dataset appropriately to achieve the desired results. Along with this I learnt how to write optimized solutions for loading datasets in small constant batches of memory.
