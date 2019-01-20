## Advanced Lane Detection.
---
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

[image_dist]: ./test_images/test1.jpg "Distorted"
[image_undist]: ./undistorted_images/undist_image0.jpg "Undistorted"
[image_processed]: ./output_images/processed_image0.jpg "Processed"
[image_trans]: ./output_images/transformed_image0.jpg
[image_conv]: ./output_images/conv_line_image0.jpg
[image_conv1]: ./output_images/poly_fit_image0.jpg
[image_lines]: ./output_images/road_lanes_image0.jpg
[image_lines1]: ./output_images/road_lanes_with_txt0.jpg


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

1) The coordinates of chessboard corners are found with `findChessboardCorners` in the image (x,y) with the knowledge that it's 9x6. The calculation to find the object points isn't required by making an assumption that the corners are stationary on a fixed (x,y) plane at z=0 in the real world. Hence, the same object point coordinates are added to the objpoints list for every image, while different image points are added to the imgpoints list.  
2) Used `objpoints` and `imgpoints` with OpenCV function `calibrateCamera()` to calculate the coffecients required to undistort the image. These coefficients are then used with `undistort()` to get an undistorted image. The result is shown in Pipeline (1.)  

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
The images below provide the comparison between the distorted image (provided in the /test_images directory), and an undistorted image (images that were saved in the /undistorted_images directory). 

_Original Image:_            |  _Undistorted Image:_
:-------------------------:|:-------------------------:
![alt text][image_dist]  |  ![alt text][image_undist]


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
Location: Advanced Lane Finding.ipynb, in Cell under Helper functions.  
Function name: color_thresholding (line 3-19) and abs_sobel_thresh (line 22-34)  

1) Gradient thresholding was used in x and y direction to extract features / edges from the image. Coordinates that fit within the thresholds, are indicated as '1' in a binary matrix. Results from gradient thresholding from x and y are then combined using AND operation to eliminate some of the noise.  
2) Image was converted to HSV and HSL, thresholding was used for Value channel and Saturation channel. Coordinates that fit within the thresholds, are indicated as '1' in a binary matrix. Results from colour thresholding from V channel and S channel are then combined using AND operation to eliminate some of the noise.  
3) The results from 1) and 2) are combined using OR operation. The output becomes the result.  

![alt text][image_processed]


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.
Location: Advanced Lane Finding.ipynb, in Cell under Helper functions.  
Function name: lane_transform (line 36-65)  
  
1) Defined 4 points for the trapezoid, which became the `src`. Tried to isolate the lanes as much as I could because the part of the clip between 20 and 25 seconds caused a lot of problems. Used constants for easy testing.
```python
    trap_top_width = 0.06 # To get top x-coordinates for the trapezoid
    trap_bot_width = 0.5 # To get bot x-coordinates for the trapezoid
    trap_top_height = 0.62 # To get top y-coordinates
    trap_bot_height = 0.935 # To get bot y-coordinates
    
    # Source points for transform
    top_left_corner =  [image_size[0] * (0.5 - trap_top_width / 2), image_size[1] * trap_top_height]
    top_right_corner = [image_size[0] * (0.5 + trap_top_width / 2), image_size[1] * trap_top_height]
    bot_left_corner =  [image_size[0] * (0.5 - trap_bot_width / 2), image_size[1] * trap_bot_height]
    bot_right_corner = [image_size[0] * (0.5 + trap_bot_width / 2), image_size[1] * trap_bot_height]
    
    src = np.float32([top_left_corner, top_right_corner, bot_right_corner, bot_left_corner])
```

2) Defined `dst` as,
```python
    offset = image_size[0] * 0.25
    
    # Dest points for transform
    top_left_trans  = [offset, 0]
    top_right_trans = [image_size[0] - offset, 0]
    bot_left_trans  = [offset, image_size[1]]
    bot_right_trans = [image_size[0] - offset, image_size[1]]
    
    dst = np.float32([top_left_trans, top_right_trans, bot_right_trans, bot_left_trans])
```
3) Used `cv2.getPerspectiveTransform` to get the M and Minv to perform the transform (with `warpPerspective`) and to undo the undo the transform to draw the lane lines on the road in the future.  

The following is the result.
![alt text][image_trans]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?
Location: Advanced Lane Finding.ipynb, line_tracker class.
Function name: `find_window_centroids` line 13-46.  

Once the image is transformed, the `find_window_centroids` function is used from the line_tracker class.
1) The image is split into 4 quarters to find the left and right starting points. The left lower quarter is summed vertically and then convolution is employed with a window array. To find the index of peaks in x axis, the `argmax` is used. The same process is applied to the right lower quarter.  
2) With the known starting point, convolution is then performed for each layer. However, since it is implied that the road line pixels are around the general area of the last layer, the convolution isnt needed for an entire layer. Only the local area is convolved (to improve the performance). The x values of peaks are stored in a list. The following image is the result from convolution.  
![alt text][image_conv]
3) The x values that were found in the last step (and y values that are already known by knowing the height of the window) are then fed into the `polyfit` function with a degee of 2. The coefficienct are then used to produce lane lines. This produced the following image.  
![alt text][image_conv1]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.
Location: Advanced Lane Finding.ipynb  
Function name: lane_finding_pipeline, lines 66-67 (radius), lines 70-71 (offset)

1) The values of x and y of the left line were first scaled to 'real world' measurements (from px to m). These values were then used to find a new polynomial with `polyfit`.  
2) Then a mathematical formula was used with the polynomial coefficients to deremine the curve.  
  
1) To determine the offset, the mid x point of the image was subtracted from the mid point between the left and right lane (last x values of left and right). This offset is then scaled from px to m.  

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The result image of step 4, using the Minv found in step 3 was used to `warpPerspective` back into the original image. The result is shown below,  

![alt text][image_lines]  

With step 5 and added green region between the lanes, the result is the followng,  

![alt text][image_lines1]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./lane_finding_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

_Problems:_  
1) In the video provided, the biggest problem was between 20 - 25s. The road changes in colour, and the road becomes more exposed to the sun. As a result, the gradient thresholding becomes less accurate during that change period. I tried to reduce the transform size of the trapezoid so that the edge between the shadow of the rail and road isn't picked up. By doing that it caused the lines to be less smooth. It eliminated the problem where the left line was generated in the wrong place, but it made the lines jitter.  
2) There is still a problem where the top of the left line curving wrongly between 20 - 25s of the video, due to the problem discussed in 1).  

_Improvements:_  
1) More froms of thresholding can be used to eliminate more noise. To reduce the problem in 1), the trapezoid corners can be more precisely selected to ensure that the correct line is detected. With the changing road conditions, an algorithm could potentially be developed to adjust the thresholds dynamically based on the changed conditions.  
2) Curve limits can be implemented to prevent the lines from curving suddenly.  
