# CSE 576 Homework 2 #

Welcome, it's time for homework 2! This one may be a little harder than the last one so remember to start early! In order to make grading easier, please only edit the files we mention to submit. You will submit the `src/hw2/modify_image.c` file on Canvas.

Run the following commands from inside your `Homework` folder:
```
git pull
make clean
make
```
and then run `./main test hw2`. For python tests use `python tryhw2.py`. During testing, you can call all the commands in the same line as:
```
make clean; make; ./main test hw2; python tryhw2.py
```

NOTE: You may (ideally should) directly use functions you wrote in Homework 1 when needed.

## 1. Image resizing ##

We've been talking a lot about resizing and interpolation in class, now's your time to do it! To resize we'll need some interpolation methods and a function to create a new image and fill it in with our interpolation methods.

### 1.1 Nearest Neighbor Interpolation ###

#### TO DO ####
Fill in `float nn_interpolate(image im, float x, float y, int c);`. It should perform nearest neighbor interpolation. Remember to use the closest `int`, not just type-cast because in C that will truncate towards zero. NOTE: your rounding of `x` and `y` should account for the fact that `x` and/or `y` may be integers.

### 1.2 Nearest Neighbor Resizing ###

#### TO DO ####
Fill in `image nn_resize(image im, int w, int h)`. It should:
- Create a new image that is `w x h` and the same number of channels as `im`
- Loop over the pixels and map back to the old coordinates (remember to use 0.5 offset appropriately)
- Use nn_interpolate() to fill in the image

Now you should be able to run the following python command in `tryhw2.py`:

    from uwimg import *
    im = load_image("data/dogsmall.jpg")
    a = nn_resize(im, im.w*4, im.h*4)
    save_image(a, "dog4x-nn")

Your image should look something like:

![blocky dog](../../figs/dog4x-nn.png)

### 1.3 Bilinear Interpolation ###

#### TO DO ####
Fill in the function `float bilinear_interpolate(image im, float x, float y, int c)` for bilinear interpolation.

### 1.4 Bilinear Resizing ###

#### TO DO ####
Fill in `image bilinear_resize(image im, int w, int h)` to perform resizing using bilinear interpolation. Try it out again in python:

    im = load_image("data/dogsmall.jpg")
    a = bilinear_resize(im, im.w*4, im.h*4)
    save_image(a, "dog4x-bl")

![smooth dog](../../figs/dog4x-bl.png)

These functions will work fine for small changes in size, but when we try to make our image smaller, say a thumbnail, we get very noisy results:

    im = load_image("data/dog.jpg")
    a = nn_resize(im, im.w//7, im.h//7)
    save_image(a, "dog7th-bl")

![jagged dog thumbnail](../../figs/dog7th-nn.png)

As we discussed, we need to filter before we do this extreme resize operation!


## 2. Image filtering ##

For this section, make sure to free all the local variables inside each function at the end of the function to reduce memory overhead. We'll start out by filtering the image with a box filter.

### 2.1 Create your box filter ###

We want to create a box filter, which as discussed in class looks like this:

![box filter](../../figs/boxfilter.png)

One way to do this is to make an image, fill it in with all 1s, and then normalize it. That's what we'll do because the normalization function may be useful in the future!

#### TO DO ####
Fill in `void l1_normalize(image im)`. This should divide each value in image `im` by the sum of all the values in the image.

#### TO DO ####
Fill in `image make_box_filter(int w)`. We will only use square box filters, so just create a square image of width = height = w and number of channels = 1, with all entries equal to 1. Then use `l1_normalize` to normalize your filter.

### 2.2 Write a convolution function ###

#### TO DO ####
Fill in `image convolve_image(image im, image filter, int preserve)`. The parameter `preserve` takes a value of either 0 or 1. We will use "clamp" padding for the image borders (get_pixel() already handles it). For this function we have a few scenarios. With normal convolutions we do a weighted sum over an area of the image. With multiple channels in the input image there are a few possible cases we want to handle:

- If `filter` and `im` have the same number of channels then it's just a normal convolution. We sum over spatial and channel dimensions and produce a 1 channel image. UNLESS:
- If `preserve` is set to 1 we should produce an image with the same number of channels as the input. This is useful if, for example, we want to run a box filter over an RGB image and get out an RGB image. This means each channel in the image will be filtered by the corresponding channel in the filter. UNLESS:
- If the `filter` only has one channel but `im` has multiple channels we want to apply the filter to each of those channels. Then we either sum between channels or not depending on if `preserve` is set.

Also, `filter` should have the same number of channels as `im` or have 1 channel. This MUST be checked with an `assert()`.
Hint: You can reduce number of lines of code by using the conditional operator in C.

We are calling this a convolution but you don't need to flip the filter or anything (we're actually doing a cross-correlation). Just apply it to the image as we discussed in class:

![convolution](../../figs/convolution.png)

Once you are done, test out your convolution by filtering our image (we need to use preserve because we want to produce an image that is still RGB).

```
from uwimg import *
im = load_image("data/dog.jpg")
f = make_box_filter(7)
blur = convolve_image(im, f, 1)
save_image(blur, "dog-box7")
```

We'll get some output that looks like this:

![covolution](../../figs/dog-box7.png)

Now we can use this to perform our thumbnail operation:

![covolution](../../figs/dogthumb.png)

Look at how much better our new resized thumbnail is!

Resize                     |  Blur and Resize
:-------------------------:|:-------------------------:
![](../../figs/dog7th-nn.png)    | ![](../../figs/dogthumb.png)

### 2.3 Make some more filters and try them out! ###

#### TO DO ####
Fill in the functions `image make_highpass_filter()`, `image make_sharpen_filter()`, and `image make_emboss_filter()` to return the example kernels we covered in class. You can try them out on some images! Now, answer Questions 2.3.1 and 2.3.2 in the source file (put your answers just right there).

Highpass                   |  Sharpen                  | Emboss
:-------------------------:|:-------------------------:|:--------------------|
![](../../figs/highpass.png)     | ![](../../figs/sharpen.png)     | ![](../../figs/emboss.png)


### 2.4 Implement a Gaussian kernel ###

#### TO DO ####
Fill in `image make_gaussian_filter(float sigma)` which will take a standard deviation value `sigma` and return a filter that smooths using a gaussian with that sigma. How big should the filter be? 99% of the probability mass for a gaussian is within +/- 3 standard deviations, so make the kernel be 6 times the size of sigma. But also we want an odd number, so make it be the next highest odd integer from 6 x sigma. We need to fill in our kernel with some values (take care of the 0.5 offset for the pixel co-ordinates). Use the probability density function for a 2D gaussian:

![2d gaussian](../../figs/2dgauss.png)

Technically this isn't perfect, what we would really want to do is integrate over the area covered by each cell in the filter. But that's much more complicated and this is a decent estimate. Remember though, this is a blurring filter so we want all the weights to sum to 1 (i.e. normalize the filter). Now you should be able to try out your new blurring function:

```
im = load_image("data/dog.jpg")
f = make_gaussian_filter(2)
blur = convolve_image(im, f, 1)
save_image(blur, "dog-gauss2")
```

It should have much less noise than the box filter:

![blurred dog](../../figs/dog-gauss2.png)

## 2.4.1. Hybrid images ##

Gaussian filters are cool because they are a true low-pass filter for the image. This means when we run them on an image we only get the low-frequency changes in an image like color. Conversely, we can subtract this low-frequency information from the original image to get the high frequency information! Using this frequency separation we can do some pretty neat stuff. For example, check out [this tutorial on retouching skin](https://petapixel.com/2015/07/08/primer-using-frequency-separation-in-photoshop-for-skin-retouching/) in Photoshop (but only if you want to). We can also make [really trippy images](http://cvcl.mit.edu/hybrid/OlivaTorralb_Hybrid_Siggraph06.pdf) that look different depending on if you are close or far away from them. That's what we'll be doing. They are hybrid images that take low frequency information from one image and high frequency information from another. Here's an example: 

Small                     |  Medium | Large
:-------------------------:|:-------:|:------------------:
![](../../figs/marilyn-einstein-small.png)   | ![](../../figs/marilyn-einstein-medium.png) | ![](../../figs/marilyn-einstein.png)

Check out `figs/marilyn-einstein.png` and view it from far away and up close. Your job is to produce a similar image. But instead of famous dead people we'll be using famous fictional people from the Harry Potter franchise! Like this:

Small                     | Large
:-------------------------:|:------------------:
![](../../figs/ronbledore-small.jpg)   | ![](../../figs/ronbledore.jpg) 

For this task you'll have to extract the high frequency and low frequency from some images. You already know how to get low frequency, using your gaussian filter. To get high frequency you just subtract the low frequency data from the original image. You will probably need different values for each image to get it to look good.

#### TO DO ####
Fill in `image add_image(image a, image b)` to add two images a and b and `image sub_image(image a, image b)` to subtract image b from image a, so that we can perform our transformations of `+` and `-` like this:

```
im = load_image("data/dog.jpg")
f = make_gaussian_filter(2)
lfreq = convolve_image(im, f, 1)
hfreq = im - lfreq
reconstruct = lfreq + hfreq
save_image(lfreq, "low-frequency")
save_image(hfreq, "high-frequency")
save_image(reconstruct, "reconstruct")
```

The functions MUST include some checks that the images are the same size using `assert()`. Now we should be able to get these results:

Low frequency           |  High frequency | Reconstruction
:-------------------------:|:-------:|:------------------:
![](../../figs/low-frequency.png)   | ![](../../figs/high-frequency.png) | ![](../../figs/reconstruct.png)

Note, the high-frequency image overflows when we save it to disk. Is this a problem for us? Why or why not? Think!

## 2.5. Sobel filters ##

The [Sobel filter](https://www.researchgate.net/publication/239398674_An_Isotropic_3x3_Image_Gradient_Operator) is cool because we can estimate the gradients and direction of those gradients in an image. They should be straightforward now that you all are such pros at image filtering.

### 2.5.1 Make the filters ###

#### TO DO ####
Fill in the functions `image make_gx_filter()` and `image make_gy_filter()` to make our sobel filters. They are for estimating the gradient in the x and y direction:

Gx                 |  Gy 
:-----------------:|:------------------:
![](../../figs/gx.png)   |  ![](../../figs/gy.png)


### 2.5.2 Feature normalization ###

To visualize our sobel operator we'll want another normalization strategy, [feature normalization](https://en.wikipedia.org/wiki/Feature_scaling). This strategy is simple, we just want to scale the image so all values lie between [0-1]. In particular we will be [rescaling](https://en.wikipedia.org/wiki/Feature_scaling#Rescaling) the image by subtracting the minimum from all values and dividing by the range (i.e. max - min) of the data. If the range is zero you should just set the whole image to 0 (don't divide by 0). 

#### TO DO ####
Fill in the function `void feature_normalize(image im)`.

### 2.5.3 Calculate gradient magnitude and direction ###

#### TO DO ####
Fill in the function `image *sobel_image(image im)`. It should return two images, the gradient magnitude and direction. The strategy can be found [here](https://en.wikipedia.org/wiki/Sobel_operator#Formulation). We can visualize our magnitude using our normalization function:

```
im = load_image("data/dog.jpg")
res = sobel_image(im)
mag = res[0]
feature_normalize(mag)
save_image(mag, "magnitude")
```

which results in:

![](../../figs/magnitude.png)

### 2.5.4 Make a colorized representation ###

Now using your sobel filter you can make a cool, stylized one. 
#### TO DO ####
Write a function `image colorize_sobel(image im)`. Call `image *sobel_image(image im)`, use the magnitude to specify the saturation and value of an image and the angle (direction) to specify the hue. Then use the `hsv_to_rgb()` function we wrote in Homework 1. Using some smoothing (a Gaussian with sigma 4), the result should look like this:

![](../../figs/lcolorized.png)

## 2.6 EXTRA CREDIT: Median Filter ##

Now, we want to apply a non-linear filter, [Median Filter](https://en.wikipedia.org/wiki/Median_filter), to an image. Median filter is a great tool to solve the salt and pepper noises. We assume a median filter is a square, with the same height and width. The kernel size is always a positive odd number. We use clamp padding for borders. The output image should have the same width, height, and channels as the input image. You should apply median filter to each channel of the input image.

#### TO DO (extra credit) ####
Fill in the function `image apply_median_filter(image im, int kernel_size)`. NOTE: You may need to write appropriate header functions. Feel free to create a test function like the existing ones in test.c and/or tryhw2.py to check the correctness of your implementation. Hint: Use the `qsort` function of C, and define the compare function yourself.

To submit the final image apply your filter to `/data/landscape.jpg` (write appropriate code in tryhw2.py and uwimg.py) and name it `median.jpg`. Good luck!


Input Noisy Image                 |  Output Image 
:-----------------:|:------------------:
![](../../figs/salt_petter_building.jpg)   |  ![](../../figs/building-median.png)


### 2.7 SUPER EXTRA CREDIT: Bilateral Filter ###

Now let's try blurring by not just assigning weights to surrounding pixels based on their spatial location in relation to the center pixel but also by how far away they are in terms of color from the center pixel. The idea of the [bilateral filter](https://cs.jhu.edu/~misha/ReadingSeminar/Papers/Tomasi98.pdf) is to blur everything in the image but the color edges. Once again we will be forming a filter, except now it will be different per pixel. The weights for a pixel's filter can be described as such:

![W](../../figs/bilateral_W.png)

where the individual weights are

![W](../../figs/bilateral_wij.png)

where f(.) returns the image pixel value and the normalization factor is 

![W](../../figs/bilateral_norm.png)

for a kernel of size (2k+1), which you can perform by calling the `l1_normalize` function.

Hint: For the spatial Gaussian, you can use the `make_gaussian_filter()` you implemented above, with `sigma1`. For the color distance Gaussian, you should compute the Gaussian with `sigma2` with the distance between the pixel values for each channel separately.

#### TO DO (super extra credit) ####
Write a function `image apply_bilateral_filter(image im, float sigma1, float sigma2)` where `sigma1` is for the spatial gaussian and `sigma2` is for the color distance Gaussian. Use a kernel size of 6 x `sigma1` for the bilateral filter. Your image should have a similar effect to the image below, so we suggest testing out a few spatial and color sigma parameters before submitting your final image (you can find the before image in `/data/bilateral_raw.png`. Note that it is 40x40 pixels and is being upsampled in this README). Ideally, `sigma1` should be > 1 and `sigma2` should be < 0.5.

To submit the final image apply your filter to `/data/landscape.jpg` (write appropriate code in tryhw2.py and uwimg.py) and name it `bilateral.jpg`. Good luck!

Before                 |  After 
:-----------------:|:------------------:
![](../../figs/bilateral_pre.png)   |  ![](../../figs/bilateral_post.png)

![cat](../../figs/bilateral_cat.png)


## Turn it in ##

Turn in your `modify_image.c` on canvas under Homework 2. If you attempt the extra credits, also submit `median.jpg` and/or `bilateral.jpg`.
