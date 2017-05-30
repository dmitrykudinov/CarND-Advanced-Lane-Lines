# **Advanced Lane Finding Project**

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

[image1]: ./examples/undistort_output.png "Undistorted Chessboard"
[image2]: ./examples/undistorted_road.png "Undistorted Road"
[image3]: ./examples/red_sobel_x.png "RED/SOBEL_X"
[image4]: ./examples/saturation.png "SATURATION"
[image5]: ./examples/light.png "LIGHT"
[image6]: ./examples/red_degree.png "RED/DEGREE"
[image7]: ./examples/red_degree_light.png "RED/DEGREE and LIGHT"
[image8]: ./examples/red_sobel_saturation.png "RED/SOBEL_X or SATURATION"
[image9]: ./examples/color_filter_on_test_pictures.png "Color filter on test pictures"
[image10]: ./examples/pt_src.png "Perspective Transform: Source"
[image11]: ./examples/pt_dst.png "Perspective Transform: Destination"
[image12]: ./examples/color_filter_on_test_pictures_pt.png "Color filter on test pictures and Perspective Transform"
[image13]: ./examples/hist1.png "Sliding histogram - 1"
[image14]: ./examples/hist2.png "Sliding histogram - 2"
[image15]: ./examples/hist3.png "Sliding histogram - 3"
[image16]: ./examples/lane_and_metrics.png "Lane and metrics"
[image17]: ./examples/simplified_lane_search1.png "Simplified lane search - 1"
[image18]: ./examples/simplified_lane_search2.png "Simplified lane search - 2"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

#### solution: ./Advanced_Lane_Detector.ipynb

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README
You're reading it :)

### Camera Calibration

#### 1. Checkerboard as calibration images.

I start, in code cell one, by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

In the second code cell I used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

Note the tree branches in the left-top corner of the image - these change the most after compensating for the camera's optical distortions. Looks like this particular camera has mostly radial distortions caused by imperfections of the lens.

The code, which does undistort of all the test images is located in the 4th code cell. the `tst_images` from that moment on store all the undistorted test images which will be used in further processing.

#### 2. Color space transformations.
During my experimetns, I also tried various Degree and Magnitude filters, Light (HLS) channel, etc.

After a few various experiments combining different parameters, filters, channels, I decided to use SOBEL_X on Red channel, OR-combined with filtered SATURATION channel.

Here are some examples:

- RED channel with SOBEL_X filter produced nice results after experimenting with its Kernel size and Threshold parameters:

![alt text][image3] 

- Good results were shown by filtered SATURATION channel: 

![alt text][image4]

- Light channel is a promising one, but in time allotted, I didn't have a chance to find a good application for it, as it produces a lot of noise which needs to be filtered out through AND-combination with something else:

![alt text][image5] 

- similar story with RED channel after DEGREE filter: 

![alt text][image6] 

- here are the RED/DEGREE and LIGHT AND-combined:

![alt text][image7] 

- This one is more interesting: look at the third picture it's the  RED/SOBEL_X and SATURATION OR-combined:

![alt text][image8] 

- And here are all the test images after the RED/SOBEL_X and SATURATION OR-combined filter:

![alt text][image9] 


The code showing the process of finding good parameters for all the above individual filters can be found in cells 5-16.

#### 3. Perspective transform.
In code cell 17, the perspective transform gets defined.

Taking one of the test images with the most straight section of the road, I manually defined the following `src` and `dst` arrays:

```
src = np.float32([
          [245, 690],[1065, 690],
          [594, 450],[684, 450]])
dst = np.float32([
          [300, 700],[1005,700],
          [300,0],[1005,0]])
```
calling `getPerspectiveTransform(src, dst)` with following after `warpPerspective(...)` allows performing the transformation:

![alt text][image10]

![alt text][image11] 

It took few iterations to make sure that the src and dst points align well, so the resulting lane markings are vertical, as in the picture above.

Code cell 18 puts both the color space filter and perspective transform into a couple concise functions, which are used in cell 19 to process all the test images:

![alt text][image12] 



#### 4. Identifying lane-line pixels and fitting their positions with a polynomial.

For static test images, I used the sliding histogram approach suggested in the course materials. This code, with some minor tweaks is encapsulated in the `calc_lane_indexes()` method, code cell 21.

Here is an example of a test picture, its lane lines pixels, and histogram of the bottom portion:

![alt text][image13] 
![alt text][image14] 

Here is an example of applying this sliding histogram approach to another test image with a nice curve on it:

![alt text][image15] 


Sliding histogram is a computationally expensive process, so in order to optimize when processing video frames, I modified the logic a bit to use the previously known good line polynoms to search within a 50px margin: see the code cell 27 for more details.


#### 5. Radius of curvature of the lane and the position of the vehicle with respect to center.

This was done in code cell 22: radius curvature was calculated using provided formula on top of the previously fit polynoms for both lines. The average of both was considered the curvature of the lane.

Offset from the lane center, assuming the dashcam is located in the center of the vehicle, was calculated as offset of the average between the line polynoms at the bottom of the frame from the center of the frame. 

BTW, the difference between left and right line curvatures is used as a sanity check in code cell 27 when processing video frames.

#### 6. Plotting detected lane back down onto the road such that the lane area is identified clearly.

This was implemented in code cell 23:

![alt text][image16] 


### Pipeline (video)

#### 1. Video output.

Here's a [link to my video result](./project_video_output_aggregator.mp4)

#### 2. Simplified lane search 

Since lane markings between different frames did not change much, it's reasonable to use previously fit polynoms to speed up the process of detecting lines in next frame.

It also makes sense, in case newly detected lines don't pass sanity checks, to adhere to previously known good polynoms for some number of frames.

This all was done in the code cell 27 using `Lane` class and `update_lane` method.

The code cell 28 does some sanity check on whether this aggregation works: the same test image is fed twice to the video processing pipeline as two consecutive frames - as you can see the second frame's turn radius and offset metrics are pretty much identical to the first frame:


![alt text][image17] 

![alt text][image18] 

### Discussion

#### 1. Problems with current implementation:

Testing on harder videos provided along with the repo, I saw that the solution I built doesn't always correctly detects the lanes. Certainly more work is needed in both color-space filtering, and lane-retention sections:

Possible options which I would test first if time would permit:
 - Color space filtering:
    - try using HUE / LIGHT channels with low values to detect black lines and combine with the current filter using NOT-condition. I'm thinking this would help properly ignoring the tar-stripes as in the second highway video.
 - Fitting 3rd order polynomial:
    - on the forest road video, the dashcam catches more than once curve in a frame.
 - more sanity checks:
    - like too big offset of the center of the lane.
    - too small turn radius.

