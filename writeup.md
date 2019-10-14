## Writeup for the Advanced Lane Lines Project

---

**Advanced Lane Finding Project**

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

[distortion_image_test]: ./output_images/distortion_correction.png "Distortion Correction Test"
[distortion_image_road]: ./output_images/distortion_correction2.png "Road Transformed"
[binary_raw]: ./output_images/binary_output.png "Binary Example"
[perspective_source]: ./output_images/perspective_source.png "Perspective Source Points"
[perspective_topdown]: ./output_images/perspective_topdown.png "Perspective Test Result"
[binary_topdown]: ./output_images/binary_topdown.png "Binary Transformed"
[lanes_topdown]: ./output_images/topdown_lanes.png "Lane finding result"
[lanes_topdown_fit]: ./output_images/topdown_lanes_marked.png "Lanes with fit and markup"
[lanes_reprojected]: ./output_images/reprojected_lanes_test.png "Lanes reprojected onto image"
[video1]: ./output_images/project_video_with_lanes.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The OpenCV camera calibration routine expects as input a set of points in both image space 
and world space. The image space coordinates are those from the source images that will get
mapped to 3D world space coordinates by the correction routine.

To find those points requires a couple of steps. First, the object world space destination
coordinates are easy. Since the goal is an undistorted image, facing the camera, we can 
use regularly spaced coordinates, matching the number and orientation of the expected source
coordinates. So we just create those like:

    objp = np.zeros((6*9,3), np.float32)
    objp[:,:2] = np.mgrid[0:9,0:6].T.reshape(-1,2)

These will be re-used throughout the procedure, since they don't change. Note that these are
in 3D space, but we've set Z=0 to be facing the camera, in line with the image plane.

Getting the source coordinates is a little harder, but OpenCV provides help for that. Assuming
our calibration images are chessboard patterns, we use `cv2.findChessboardCorners()` to extract
the matching coordinates in source image (2D) space from each image.

We then run through the complete set of calibration images, extracting the chessboard
corners for each. Finally we pass that to `cv2.calibrateCamera()` to create the distortion
matrix and parameters to be able to correct camera images with `cv2.undistort()`

The result of that with a test image is shown below and you can see that the barrel and
fisheye distortion is removed.

The complete procedure is in the `In [1]` cell of the Jupyter notebook.
![alt text][distortion_image_test]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Now that we have the calibration parameters, we can try applying it to some real images
to make sure it works. Below are the provided test images with their undistorted counterparts
on the right. The effect is more subtle on real images, but you can clearly see that the
hood geometry changes a bit and the tree shapes are altered near the edges. In practice,
since most of the detection action occurs in the center of the images where the distortion
is less pronounced, the effect is not huge.

![alt text][distortion_image_road]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I created a function called `extract_features()` (see cell `In [3]`) to create the 
binary images of lane edges for input to the detection algorithm. I used a combination
of three factors to decide whether a pixel belonged to a candidate lane marker or not.

First I converted the image to HLS color-space so I could work with saturation and 
luminance channels.

Next I computed the Sobel transform of the image in both the x and y directions.

Finally I used the Sobel x and y scalar fields to compute the gradient direction at
each pixel position using `arctan2()`.

With this data I thresholded the saturation channel, sobel x, and the direction field
to narrow down lane marker like features.

    binary[((scaled_sobel >= sx_thresh[0]) & (scaled_sobel <= sx_thresh[1])) |
          ((s_channel >= s_thresh[0]) & (s_channel <= s_thresh[1])) &
          ((dir_grad >= dir_thresh[0]) & (dir_grad <= dir_thresh[1]))] = 1

The thresholds are adjustable inputs to `extract_features()`
    
Here is the raw output of `extract_features()` applied to some test images. It does
a pretty good job of isolating strong lane marker-like features.

![alt text][binary_raw]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The next step of the pipeline is perspective transformation, to get a top-down view of the
road. The function that handles that is called `perspective_correction()` and takes the
source image as an argument and returns a transformed image. The function uses
a transformation matrix (the output of `cv2.getPerspectiveTransform()`) as input
to `cv2.warpPerspective()` to produce a birds-eye view of the road. 

`perspective_correction()` is defined in `In [7]`.

The source points for the transformation were obtained empirically using a straight section
of road. The below image is the image used with the source points outlines with a red polygon.

![alt text][perspective_source]

To verify that this produces the correct output, we can check the post-transform image.

![alt text][perspective_topdown]

This looks good and produces straight lane lines as expected. Applying it to the test images
and their binary feature outputs results in:

![alt text][binary_topdown]

Those are fairly clean and free of extraneous detail, suitable for lane finding.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Lane detection is handled by a class called `LaneDetector` which is first defined in cell
`In[10]`. It is later refined and improved after testing.

First on init, the lane detector is provided with a seed images for size hinting purposes,
a set of hyperparameters to drive the search, and an array of `Line` classes to track
found lane-line stats over time.

The lane detector class has two methods for finding lane pixels in images. First is the
window method, whereby a histogram is taken to determine a likely starting point for the
lane markers near the bottom of the image. Then it identifies nonzero pixels within 
windows around those peaks. The process is repeated with sliding windows moving "up"
the image in y. If there are enough pixels (above a configurable margin) in a window,
it is allowed to be recentered on the new peak, to follow the lane marker. At the end 
of the procedure a complete set of non-zero pixels corresponding to a lane marker is
produced. This is all handled in the method `search_via_windows`.

The next method, defined in `search_via_prior_fit` relies on a previous successful
polynomial fit of either the result of `search_via_windows` or itself. This is an 
optimization that eliminates repetitive sliding window searches by simply finding
all non-zero pixels near the polynomial fit line. This can fail if the image quality
is poor or the lane direction changes suddenly.

Detetion as a whole is driven by the method `detect()` which goes through each `Line`
instance provided and tries to find the lane markers via whatever method is appropriate
at the time. If a Line has not been ever detected before, or it's been lost, `detect`
will use `search_via_windows`. If a Line has been detected before, it will attempt to 
use `search_via_prior_fit`. After searching the data is cached in the `Line` instance,
a fit is produced via the method `fit_poly()` and we move on.

Below is the result of this on top down images.

![alt text][lanes_topdown]

These classes were all expanded upon and improved for use on the video and I will describe
those changes in sections 5 and 6.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I reimplemented the lane detector class and it's supporting classes in cells `In [45]`
through `In [48]`. Particularly important changes were the inclusion of a confidence
return value from the search functions, allowing `detect` to more accurately detect lane
quality problems and fall back to `search_via_windows`. 

Additionally, the `Line` class and the `LaneDetector` class were extended to keep an 
average of the last `N` detected lanes. This smoothes out noisy image to image variation.

A couple of convenience functions were added to handle some bookkeeping and compute lane
data. `update_data()` in particular handles data averaging as well as computing the current
lane position and curvature in meters. `notate_lane_data()` in cell `In [48]` computes the
final car position, combined lane curvature and then writes those values back on top of the
image.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The final step in the pipeline is to reproject the lane lines back on the source image
and overlay the statistics. The overlay of the markers is done in `overlay_lane_marker`
(cell `In [49]`). The outer loop that drives detection is defined in cells `In [52]` and
`In [53]`. Below is an example of the pipeline applied to single test images in top-down
view.

![alt text][lanes_topdown_fit]

And reprojected onto the source images:

![alt text][lanes_reprojected]
---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result][video1]

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

There's a few areas I wanted to tackle but ran out of time. First, my pipeline doesn't
work well (or may crash) if no lane pixels are detected at all. This is why it fell 
down on the challenge videos. Given more time, I'd include that in the confidence value handling. If one lane exists and the other doesn't, we can synthesize the missing one
from the curvature and known probably x-offset of standard lanes.

Second I think further tweaking of the thresholds would handle shadowed areas better.
I think some more experimentation there could dial in a more capable pipeline. I think
using some other features (luminance thresholding, Sobel Mag) could also possibly help.

Finally, being adaptive with some of the parameters and hyperparameters might be an interesting
path to explore. Changing thresholds based on some macro image factors (overall brightness, noise, etc.) might be better than relying on static values.

Theoretically, it would be pretty doable to extend the way I've handled lane lines and 
detection to more than two lanes to get a complete view of the highway.