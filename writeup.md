## Advanced Lane Finding Project

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/undistort_output.png "Undistorted"
[image2]: ./output_images/undistort-original-image.png "Undistorted Original Image"
[image3]: ./output_images/original-xthreshold.png "xSobel Threshold"
[image4]: ./output_images/original-magnitude-threshold.png "Magnitude Threshold"
[image5]: ./output_images/original-direction-threshold.png "Directional Threshold"
[image6]: ./output_images/original-combined-threshold.png "Gradients Combined Threshold"
[image7]: ./output_images/original-color-threshold.png "Color Threshold"
[image8]: ./output_images/combined-color-gradient-threshold.png "Color-Gradient Threshold"
[image9]: ./output_images/H-L-S.png "HLS Color Channels"
[image10]: /output_images/lane-lines-region.png "Source Points Warp"
[image11]: ./output_images/original-warp-prespective.png "Warped Image"
[image12]: ./output_images/histogram.png "Histogram Peak"
[image13]: ./output_images/sliding-window.png "Sliding Window Polyfit"
[image14]: ./output_images/polyfit.png "Second Order Polynomial Fit"
[image15]: ./output_images/Rcurve.png "Radius Curve Equation"
[image16]: ./output_images/final-image.png "Final Output"

[video1]: ./output_videos/project_video.mp4 "Video"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is shown in the code cell of IPython notebook located in "./AdvanceLaneLines.ipynb" under the heading Camera Calibration.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained the following results:

![alt text][image1]

### Pipeline

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:

![alt text][image2]

First I read the image in from the test_img by using `cv2.imread()` and then converteing the image from BGR to RGB, `cv2.cvtColor(test_img, cv2.COLOR_BGR2RGB)`. Once the image was ready I applied the following method `cv2.undistort(test_img, mtx, dist, None, mtx)`. This takes the test_img from our test set, and distortion applied with the matrix, mtx and distination points, dist to undistort the image. Mtx and Dist have been found from camera calibration sets take before.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image.  Here are examples of color and gradient thresholds individually with the final one being a combination of both as it provided the best results.

![alt text][image3]
![alt text][image4]
![alt text][image5]
![alt text][image6]
![alt text][image7]
![alt text][image8]

As you can see the last image provides the best results for the lane lines. This involved fine tuning some of the parameters to get this results. The gradient threshold techniques used in this were Sobelx, Sobely, directional and magnitude. For color the image changed the channels from RGB to HLS, the image below shows the results where it shows S-channel clearly detecting the lane lines.

![alt text][image9]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp_perspective()`, which appears in `./AdvanceLaneLines.ipynb` IPython notebook. The `warp_perspective()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
# Bottom Left, Top Left, Top Right, Bottom Right
src = np.float32([(230, 720), (570, 480), (780, 480), (1180, 720)])
dst = np.float32([(230, 720), (230, 0), (1180, 0), (1180, 720)])
```

This resulted in the following source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 230, 720      | 230, 720      |
| 570, 480      | 230, 0        |
| 780, 480      | 1180, 0       |
| 1180, 720     | 1180, 720     |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image10]
![alt text][image11]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

You will find the code used to identigy lane-line pixels in `./AdvanceLaneLines.ipynb` IPython notebook under the header 'Detect Lane Pixels and Fit to Find The Lane Boundary'

Firstly I plotted a histogram which identified the two peaks, corresponding to the left and right lane-lines and this can be a starting point to search for lane-lines.

[!alt text][image12]

A large sum of code was contructed into a function called `find_lanes()` which took an input of `img`, `nwindows=9`, `margin=100`, `minpix=50`. The function of `find_lanes()` is to find the initial lanes using histogram to find two peaks and start to search through the base peaks one by one using the variables `nwindows` sets the number of slding windows to search for on 1 lane-line. The `margin` sets the width of the windows +/- margin and `minpix` to set the minimum number of pixels found to recenter window.

This windows search would successful accumulate left, right lane indexs, which are the pixels where lane-lines have been detected. These points can be concatenated and be broken down to x and y pixel postions for left and right lanes-lines which can be fit a second order polynomial.

Later points are generated for x and y values for plotting a second order polynomial fit as shown below:

![alt text][image13]

![alt text][image14]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in `./AdvanceLaneLines.ipynb` IPython notebook under the function `radius_metres()`. The equation for Radius curve is used, where the first and second derivative are taken of the second order polynomial curve found by locating lane line pixels.

![alt text][image15]

To find the vehicle center it is assumed that the camera is mounted in the middle. Which will mean true center would be half of images width. The second order polynomial curve is used to find the left and right line at the bottom of the image. The the value of between left and right line is substracted from 'true center', where negative value would be mean the vehicle is on the right and vise versa.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I did this in `./AdvanceLaneLines.ipynb` IPython notebook under the function `draw_lane()`. Where it unwarps the binary image that contain the left and right lane on to the original image and use a method `cv2.fillPoly()` to draw onto the warped image. The method used to warp the image back to the orginal was `cv2.warpPerspective()` however this time `Minv` was used which is the inverse of `M` that was used to get the birds-eye point of view. Information was also drawn on such as left and right curvature and vehicle position.

Here is an example of my result on a test image:

![alt text][image16]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_videos/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

One of the issues I saw was that in the change of brightness the lane struggled and use to not follow the lane line, so the change I conducted was normalize the histogram data, this helped alot in the fluctuation of the lane lines. Another aspect I would like to explore is using the L-channel in HLS, to make it more robust.

Also as there are sooo many variables that has been fine-tuned that in extreme condition they might not work, so example the color-graident thresholds.Because of the intensive or repetitive calculations this will effect the performance of the process.

Current the pipeline might fail is there were one lane due to extreme steep curves. In the future I would love to pursue and attempt the challenging videos to future improve my pipeline. 
