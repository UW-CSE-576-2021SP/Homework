![panorama of field](../../figs/field_panorama.jpg)

# CSE 576 Homework 3 #

Welcome, it's time for homework 3! Similar to last homework, perform

```
git pull
make clean
make
```

and then `./main test hw3` and/or `python tryhw3.py`.

This homework covers finding keypoints in an image, describing those key points, matching them to the corresponding points in another image, computing the transform from one image to the other, and stitching them together into a panorama.

The high-level algorithm is already done for you! You can find it near the bottom of `src/hw3/panorama_image.c`, it looks approximately like:

    image panorama_image(image a, image b, float sigma, float thresh, int nms, float inlier_thresh, int iters, int cutoff)
    {
        // Calculate corners and descriptors
        descriptor *ad = harris_corner_detector(a, sigma, thresh, nms, &an);
        descriptor *bd = harris_corner_detector(b, sigma, thresh, nms, &bn);
    
        // Find matches
        match *m = match_descriptors(ad, an, bd, bn, &mn);
    
        // Run RANSAC to find the homography
        matrix H = RANSAC(m, mn, inlier_thresh, iters, cutoff);
    
        // Stitch the images together with the homography
        image comb = combine_images(a, b, H);
        return comb;
    }

So we'll find the corner points in an image using a Harris corner detector. Then we'll match together the descriptors of those corners. We'll use RANSAC to estimate a projection from one image coordinate system to the other. Finally, we'll stitch together the images using this projection.

This homework may use your previous code from HW1 and HW2. You can use `./main test hw1` and `./main test hw2` to test your previous code as well. Also `./main test hw3` is provided as a sample driver program and does some basic tests and runs most parts of your algorithm. If  `./main test hw3` runs without errors and produces some panoramic images as well as good matches then there's a good chance that you are about 75% done. 

First we need to find those corners!

## 1. Harris corner detection ##

We'll be implementing Harris corner detection as discussed in class. Implement this part in `harris_image.c`. The basic algorithm is:

    Calculate image derivatives Ix and Iy.
    Calculate measures Ix^2, Iy^2, and Ix * Iy.
    Calculate structure matrix S as weighted sum of nearby measures.
    Calculate Harris "cornerness" as estimate of 2nd eigenvalue: det(S) - α trace(S)^2, α = .06
    Run non-max suppression on response map

## 1.1 Compute the structure matrix ##

#### TO DO ####
Fill in `image structure_matrix(image im, float sigma)`. This will perform the first 3 steps of the algorithm: calculating derivatives, the corresponding measures, and the weighted sum of nearby derivative information. You can use Sobel filter and associated functions from HW2 to calculate the derivatives. The measures are element-wise multiplications. The weighted sum can be easily computed with a Gaussian blur as discussed in class. Use the parameter `sigma` to create the Gaussian kernel and convolve the result with it.

With a correct implementation, you should pass `test_structure()`.

### 1.1.1 Make a fast smoother ###

You want a fast corner detector! You can decompose the Gaussian blur from one large 2D convolution to 2 1D convolutions. Instead of using an N x N filter you should convolve with a 1 x N filter followed by the same filter flipped to be N x 1. The size of the 2D Gaussian filter should be the same as described in HW2. The formula is given in the slides.

#### TO DO ####
Fill in `image make_1d_gaussian(float sigma)` and `image smooth_image(image im, float sigma)` to use this decomposed Gaussian smoothing.

## 1.2 Computer cornerness from structure matrix ##

#### TO DO ####
Fill in `image cornerness_response(image S)` to calculate cornerness using the formula mentioned.

With a correct implementation, you should pass `test_cornerness()`.

## 1.3 Non-maximum suppression ##

We only want local maximum responses to our corner detector so that the matching is easier. For every pixel in `im`, check every neighbor within `w` pixels (Chebyshev distance). Equivalently, check the `2w+1` window centered at each pixel. If any responses are stronger, suppress that pixel's response (set it to a very low negative number).

#### TO DO ####
Fill in `image nms_image(image im, int w)`.

## 1.4 Complete the Harris detector ##

#### TO DO ####
Fill in the missing sections of `descriptor *harris_corner_detector(image im, float sigma, float thresh, int nms, int *n)`. The function should return an array of descriptors for corners in the image. Code for calculating the descriptors is provided. Also, set the integer `*n` to be the number of corners found in the image.

After you complete this function you should be able to calculate corners and descriptors for an image! Try running the python code in `tryhw3.py`:

    im = load_image("data/Rainier1.png")
    detect_and_draw_corners(im, 2, 50, 3)
    save_image(im, "output/corners")

This will detect corners using a Gaussian window of sigma = 2, a "cornerness" threshold of 100, and an nms distance of 3 (or window of 7x7). It should give you something like this:

![rainier corners](../../figs/corners.jpg)

Corners are marked with the crosses. They seem pretty sensible! Lots of corners near where snow meets rock and such. Try playing with the different values to see how they affect our corner detector.

## 2. Patch matching ##

To get a panorama we have to match up the corner detections with their appropriate counterpart in the other image. The descriptor code is already written for you. It consists of nearby pixels except with the center pixel value subtracted. This gives us some small amount of invariance to lighting conditions.

The rest of the homework takes place in `src/hw3/panorama_image.c`.

## 2.1 Distance metric ##
For comparing patches we'll use L1 distance. Squared error (L2 distance) can be problematic with outliers as we saw in class. We don't want a few rogue pixels to throw off our matching function. L1 distance (sum absolute difference) is better behaved with some outliers.

#### TO DO ####
Implement float `l1_distance(float *a, float *b, int n)` between two vectors of floats. The vectors and how many values they contain are passed in as arguments.

## 2.2 Matching descriptors ##

### 2.2.1 Find the best matches from a to b ###

First we'll look through descriptors for `image a` and find their best match with descriptors from `image b`. 
#### TO DO ####
Fill in the first `TODO` in `match *match_descriptors(descriptor *a, int an, descriptor *b, int bn, int *mn)`.

### 2.2.2 Eliminate multiple matches to the same descriptor in b

Each descriptor in `a` will only appear in one match. But several of them may match with the same descriptor in `b`. This can be problematic. Namely, if a bunch of matches go to the same point there is an easy homography to estimate that just shrinks the whole image down to one point to project from `a` to `b`. But we know that's wrong. So let's just get rid of these duplicate matches and make our matches be one-to-one.

To do this, sort the matches based on distance so shortest distance is first. A comparator `match_compare` is already implemented for you to use, just look up how to properly apply `qsort` if you don't know already. Next, loop through the matches in order and keep track of which elements in `b` we've seen. If we see one for a second (or third, etc.) time, throw it out! To throw it out, just shift the other elements forward in the list and then remember how many one-to-one matches we have at the end. This can be done with a single pass through the data.

#### TO DO ####
Fill in the second `TODO` in `match *match_descriptors(descriptor *a, int an, descriptor *b, int bn, int *mn)`. Once this is done we can show the matches we discover between the images:

    a = load_image("data/Rainier1.png")
    b = load_image("data/Rainier2.png")
    m = find_and_draw_matches(a, b, 2, 50, 3)
    save_image(m, "output/matches")

Which gives you:

![matches](../../figs/matches.jpg)


## 2.3 Fitting our projection to the data ##

Now that we have some matches we need to predict the projection between these two sets of points! However, this can be hard because we have a lot of noisy matches. Many of them are correct but we also have some outliers hiding in the data.

### 2.3.1 Projecting points with a homography ###

#### TO DO ####
Implement `point project_point(matrix H, point p)` to project a point using matrix `H`. You can do this with the provided matrix library by calling `matrix_mult_matrix` (see `src/matrix.c`). Or you could pull out elements of the matrix and do the math yourself. Whatever you want! Just remember to do the proper normalization for converting from homogeneous coordinates back to image coordinates. (We talked about this in class, something with that w squiggly thing).

And now you should pass `test_projection()`.

### 2.3.2  Calculate distances between points ###

#### TO DO ####
Fill in `float point_distance(point p, point q)` to compute L2 distance between two points.

### 2.3.3 Calculate model inliers ###

Figure out how many matches are inliers to a model.

#### TO DO ####
Fill in `int model_inliers(matrix H, match *m, int n, float thresh)` to loop over the points, project using the homography, and check if the projected point is within some `thresh` distance of the actual matching point in the other image.

Also, we want to bring inliers to the front of the list. This helps us later on with the fitting functions. Again, this should all be doable in one pass through the data.

### 2.3.4 Fitting the homography ###

We will solve for the homography using the matrix operations discussed in class to solve equations like `M a = b`. Most of this is already implemented, you just have to fill in the matrices `M` and `b` with our match information.

You also have to read out the final results and populate our homography matrix. Consult the slides for details about what should go where, or derive it for yourself using the projection equations!

#### TO DO ####
Fill in `matrix compute_homography(match *matches, int n)`.

## 2.4 Randomize the matches

One of the steps in RANSAC is drawing random matches to estimate a new model. One easy way to do this is randomly shuffle the array of matches and then take the first `n` elements to fit a model.

#### TO DO ####
Implement the [Fisher-Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle#The_modern_algorithm) in `void randomize_matches(match *m, int n)`.

After this step, you should pass the rest `test_compute_homography`.

## 2.5 Implement RANSAC ##

Implement the RANSAC algorithm discussed in class.
#### TO DO ####
Fill in `matrix RANSAC(match *m, int n, float thresh, int k, int cutoff)`. Pseudocode is provided in the code file.

## 2.6 Combine the images with a homography ##

Now we have to stitch the images together with our homography!

Some of this is already filled in. The first step is to figure out the bounds to the image. To do this we'll project the corners of `b` back onto the coordinate system for `a` using `Hinv`. Then we can figure out our "new" coordinates and where to put `image a` on the canvas. Paste `a` in the appropriate spot in the canvas.

Next we need to loop over pixels that might map to `image b`, perform the mapping to see if they do, and if so fill in the pixels with the appropriate color from `b`. Our mapping will likely land between pixel values in `b` so use bilinear interpolation to compute the pixel value (use your implementation from HW2).

#### TO DO ####
Fill in `image combine_images(image a, image b, matrix H)`. With all this working you should be able to create some basic panoramas:

    im1 = load_image("data/Rainier1.png")
    im2 = load_image("data/Rainier2.png")
    pan = panorama_image(im1, im2, thresh=50, draw=1)
    save_image(pan, "output/easy_panorama")

![panorama](../../figs/easy_panorama.jpg)

Try running the python code `rainier_panorama()` in `tryhw3.py`. You will get a Rainier panorama named `rainier_panorama_5.jpg`. Attach it in your report.

You may also try out some of the other panorama images in `./data` (sun\*.jpg, helens\*.jpg). If you stitch together multiple images you should not set `draw=1`  in `panorama_image` that marks corners and draws inliers. See `rainier_panorama()`in `tryhw3.py`.

## 3. Projections ##

Mapping all the images back to the same coordinates is bad for large field-of-view panoramas, as discussed in class. Our pipeline is going to change by reprojecting out images to different coordinates before inputting them to `panorama_image`.

### 3.1 Cylindrical 

#### TO DO  ####
Implement `image cylindrical_project(image im, float f)` to project an image to cylindrical coordinates and then unroll it. Consult the lecture slides for details about how to implement it. With this implementation you should be able to generate some very big panoramas (the `field*.jpg` panorama) using `field_panorama` in `tryhw3.py`. Attach the generated `field_panorama_5.jpg` in your report. You may also try your own images if you want!

## Turn it in ##

Turn in your `harris_image.c`, `panorama_image.c`, and `report.pdf` on canvas under Homework 3. Your report should contain:

* Page 1: `inliers.jpg`, follwed by `panorama.jpg`; created by running `tryhw3.py`.
* Page 2: `rainier_panorama_5.jpg`; created by running `tryhw3.py` using planar projection.
* Page 3: `field_panorama_5.jpg`; created by running `tryhw3.py` using cylindrical projection.

