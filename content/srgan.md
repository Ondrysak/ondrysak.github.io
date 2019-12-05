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

# Getting the data and preprocessing

First we need to download and upack the data, here we are using the COCO dataset, but any image dataset with sufficiently large(200x200) can be used.

```bash
wget http://images.cocodataset.org/zips/val2017.zip
unzip val2017.zip
mv val2017/* 'drive/My Drive/COCO_DATASET/'
```
I assume working with a Google Drive storage and Google Colab to actually run train the model, if you want to train the model locally it shouldnt be hard for you to modify the paths in the various parts of the following code.


## Cropping

First we need to crop the images to 200*200 pixels, because bigger images caused our hardware to run out of memory.
```python
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
To obtain low resolution images for training we will use bicubic downscaling, see the code below. First we load the images into numpy array and after that we use imresize from scipy.


```python
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

# Model

Here the main code for defining both generator and discriminator will be present together with the vgg based loss function. If you are not intrested in the implementation feel free to skip this part. Also 


## Architecture

The images below are the best way to see the architecture of the network, if you want to read more about the architecture of this network be sure to read the paper [Photo-Realistic Single Image Super-Resolution Using a GenerativeAdversarial Network] (https://arxiv.org/abs/1609.04802)
### GAN in general

A generative adversarial network (GAN) is a class of machine learning systems invented by Ian Goodfellow and his colleagues in 2014. Two neural networks contest with each other in a game (in the sense of game theory, often but not always in the form of a zero-sum game). Given a training set, this technique learns to generate new data with the same statistics as the training set. For example, a GAN trained on photographs can generate new photographs that look at least superficially authentic to human observers, having many realistic characteristics. Though originally proposed as a form of generative model for unsupervised learning, GANs have also proven useful for semi-supervised learning, fully supervised learning, and reinforcement learning. In a 2016 seminar, Yann LeCun described GANs as "the coolest idea in machine learning in the last twenty years".

The image below show the general architecture of any GAN.

[IMAGE OF GAN ARCHITECTURE]({filename}/images/gan.jpeg)

To read more about GANs i can recommend reading the wikipedia article and also the original paper from Goodfellow. 

[GAN on wikipedia](https://en.wikipedia.org/wiki/Generative_adversarial_network)

[Generative Adversarial Nets](https://arxiv.org/pdf/1406.2661v1.pdf)


### SRGAN

The images linked below show the architecture of SRGAN as shown in the original paper mentioned above.

[IMAGE OF GENERAL SRGAN ARCHITECTURE]({filename}/images/srgan.jpeg)

[IMAGE OF DETAILED SRGAN ARCHITECTURE]({filename}/images/srgan_detail.jpeg)

Some notes explaining a few maybe not so common terms used in the detailed image.

- *Residual blocks*: Since deeper networks are more difficult to train. The residual learning framework eases the training of these networks, and enables them to be substantially deeper, leading to improved performance. More about Residual blocks and Deep Residual learning can be found in paper given below. 16 residual blocks are used in Generator.

- *PixelShuffler x2*: This is feature map upscaling. 2 sub-pixel CNN are used in Generator. Upscaling or Upsampling are same. There are various ways to do that. In code keras inbuilt function has been used.

- *PRelu(Parameterized Relu)*: We are using PRelu in place of Relu or LeakyRelu. It introduces learn-able parameter that makes it possible to adaptively learn the negative part coefficient.

- *k3n64s1* this means kernel 3, channels 64 and strides 1.



## Generator 
The code below implements the generator model in keras.


```python
# Residual block
def res_block_gen(model, kernal_size, filters, strides):    
    gen = model
    
    model = Conv2D(filters = filters, kernel_size = kernal_size, strides = strides, padding = "same")(model)
    model = BatchNormalization(momentum = 0.5)(model)
    # Using Parametric ReLU
    model = PReLU(alpha_initializer='zeros', alpha_regularizer=None, alpha_constraint=None, shared_axes=[1,2])(model)
    model = Conv2D(filters = filters, kernel_size = kernal_size, strides = strides, padding = "same")(model)
    model = BatchNormalization(momentum = 0.5)(model)
        
    model = add([gen, model])
    
    return model
    
    
def up_sampling_block(model, kernal_size, filters, strides):
    
    # In place of Conv2D and UpSampling2D we can also use Conv2DTranspose (Both are used for Deconvolution)
    # Even we can have our own function for deconvolution (i.e one made in Utils.py)
    #model = Conv2DTranspose(filters = filters, kernel_size = kernal_size, strides = strides, padding = "same")(model)
    model = Conv2D(filters = filters, kernel_size = kernal_size, strides = strides, padding = "same")(model)
    model = UpSampling2D(size = 2)(model)
    model = LeakyReLU(alpha = 0.2)(model)
    
    return model
  
class Generator(object):
    def __init__(self, noise_shape):
        
        self.noise_shape = noise_shape
    def generator(self):
        
        gen_input = Input(shape = self.noise_shape)
     
        model = Conv2D(filters = 64, kernel_size = 9, strides = 1, padding = "same")(gen_input)
        model = PReLU(alpha_initializer='zeros', alpha_regularizer=None, alpha_constraint=None, shared_axes=[1,2])(model)
        
        gen_model = model
        
        # Using 16 Residual Blocks
        for index in range(16):
            model = res_block_gen(model, 3, 64, 1)
     
        model = Conv2D(filters = 64, kernel_size = 3, strides = 1, padding = "same")(model)
        model = BatchNormalization(momentum = 0.5)(model)
        model = add([gen_model, model])
     
        # Using 2 UpSampling Blocks
        for index in range(2):
            model = up_sampling_block(model, 3, 256, 1)
     
        model = Conv2D(filters = 3, kernel_size = 9, strides = 1, padding = "same")(model)
        model = Activation('tanh')(model)
    
        generator_model = Model(inputs = gen_input, outputs = model)
		return generator_model
```

## Discriminator
The code below implements the discriminator model in Keras.

```python
def discriminator_block(model, filters, kernel_size, strides):
    
    model = Conv2D(filters = filters, kernel_size = kernel_size, strides = strides, padding = "same")(model)
    model = BatchNormalization(momentum = 0.5)(model)
    model = LeakyReLU(alpha = 0.2)(model)
    
    return model
  
class Discriminator(object):

    def __init__(self, image_shape):
        
        self.image_shape = image_shape
    
    def discriminator(self):
        
        dis_input = Input(shape = self.image_shape)
        
        model = Conv2D(filters = 64, kernel_size = 3, strides = 1, padding = "same")(dis_input)
        model = LeakyReLU(alpha = 0.2)(model)
        
        model = discriminator_block(model, 64, 3, 2)
        model = discriminator_block(model, 128, 3, 1)
        model = discriminator_block(model, 128, 3, 2)
        model = discriminator_block(model, 256, 3, 1)
        model = discriminator_block(model, 256, 3, 2)
        model = discriminator_block(model, 512, 3, 1)
        model = discriminator_block(model, 512, 3, 2)
        
        model = Flatten()(model)
        model = Dense(1024)(model)
        model = LeakyReLU(alpha = 0.2)(model)
       
        model = Dense(1)(model)
        model = Activation('sigmoid')(model) 
        
        discriminator_model = Model(inputs = dis_input, outputs = model)
        
		return discriminator_model
```

## VGG based loss function
This is the key part of the original SRGAN paper mentioned above, rather than using mean squared error on each pixel, we feed both the HR and SR images to a VGG network which generates features using convolution and as can be seen on the last line of the code below, we than use those features to compute the mean squared error.


```python
def vgg_loss(self, y_true, y_pred):
    vgg19 = VGG19(include_top=False, weights='imagenet', input_shape=self.image_shape)
    vgg19.trainable = False
    for l in vgg19.layers:
        l.trainable = False
    model = Model(inputs=vgg19.input, outputs=vgg19.get_layer('block5_conv4').output)
    model.trainable = False
    
	return K.mean(K.square(model(y_true) - model(y_pred)))
```

# Implementation and pretrained weights

The full implementation in form of Jupyter notebook can be found in this repo. Also I am making the pretrained model weights available, which I will try to update once i have a decently sized cluster available to train the network on properly sized dataset. The pretrained model weights available below, are result of training on a small subset of the coco dataset 600 images for training and 300 for testing.

[Git repository](https://gitlab.fit.cvut.cz/nankaond/mvi-sp)
[Pretrained model weight](https://drive.google.com/open?id=1RP-3tbBH4X8qRsdaJkdpx2DlYerMt5fI)

# Results

The results below are output of the network after roughly 1900 epochs of training, which is roughly 3 days of training in the Google Colab environment.

## On a single frame

[Dog]({filename}/images/dog.png)

## Actual SR videos

[Sea dog and ball](https://www.youtube.com/watch?v=lvwLhzCRBEM&feature=youtu.be)

[Sheeps chewing](https://www.youtube.com/watch?v=wRigzcSHP0M&feature=youtu.be)

[Jellyfish](https://www.youtube.com/watch?v=r_LydST1hik&feature=youtu.be)

[Dog](https://www.youtube.com/watch?v=w7kJpZUnkI8&feature=youtu.be)

[Psychedelic face](https://www.youtube.com/watch?v=yEZVtZFvirU&feature=youtu.be)

[Fractal zoom](https://www.youtube.com/watch?v=Jooi5cuuFXo&feature=youtu.be)


# References

[Photo-Realistic Single Image Super-Resolution Using a GenerativeAdversarial Network] (https://arxiv.org/abs/1609.04802)

[Hyeongchan Kim, Awesome-GANs with Tensorflow, (2018), GitHub repository](https://github.com/kozistr/Awesome-GANs)

[Deepak Birla, Photo-Realistic Single Image Super-Resolution Using a Generative Adversarial Network implemented in Keras, (2018), GitHub repository](https://github.com/deepak112/Keras-SRGAN)

[DIVerse 2K resolution high quality images as used for the challenges, Radu Timofte, Eirikur Agustsson, Shuhang Gu, Jiqing Wu, Andrey Ignatov, Luc VanGool](https://data.vision.ee.ethz.ch/cvl/DIV2K/)

[Common objects in Context, COCO Consortium](http://cocodataset.org/#home)


[A Generative Adversarial Networks tutorial applied to Image Deblurring with the Keras library, Raphaël](https://www.sicara.ai/blog/2018-03-20-GAN-with-Keras-application-to-image-deblurring)