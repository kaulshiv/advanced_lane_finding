# Advanced Lane Finding Project

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

[image1]: ./output_images/undistorted_checkerboard.png "Undistorted"
[image2]: ./output_images/undistorted_test6.png "Road Transformed"
[image3]: ./output_images/binary_test6.png "Binary Example"
[image4]: ./output_images/perspective_straight_lines1.png "Warp Example Straight Lines"
[image5]: ./output_images/perspective_test6.png "Warp Example Curved Lines"
[image6]: ./output_images/fit_test6.png "Fit Visual"
[image7]: ./curvature_formula.png "Curvature Formula"
[image8]: ./output_images/test6.jpg "Lane Region"
[video1]: ./project_video.mp4 "Video"

---

## Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for calculating the camera matrix and distortion coefficients is located in `./advanced_lane_finding.ipynb` under the `Find Calibration Matrix` section.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

The camera matrix is store in the `mtx` variable while the distortion coefficients are stored in the `dist` variable. These two are referenced periodically in the rest of the notebook.

---
## Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The image below shows an original distorted road image `test_image/test6.jpg` and its corresponding undistorted image. The Code for this section is under the  `Example of an Undistorted Road Images` section of the notebook. I use `cv2.undistort()` with the camera matrix and distortion coefficients calculated in the previous cell.

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I performed thresholding on the R and G channels in the RGB colorspace in order to better separate out the yellow and white-colored lanes. I also converted the input image into the HLS colorspace. I then took the Sobel derivate in the x direction on the L channel. I applied thresholding to the resulting tensor to get a binary image. I then separately applied thresholding on the S channel to get a separate binary image. I combined all the above thresholding methods together with logical operators to produced the final binary image. The below image shows the result on the same `test_image/test6.jpg`.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is located in the two cells underneath the `Perspective Transform` title in the notebook. The first cell contains hardcoded values for the source and destination points of the perpective transform. These are also shown in the table below. It then goes on to calculate the tranform and inverse transform matrices. Finally, I define a function to calculate the perspective transform on an input image.

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 580, 460      | 300, 0        | 
| 710, 460      | 1000, 0       |
| 1115, 720     | 1000, 720     |
| 210, 720      | 900, 720      |

The second cell applies the transform to one of the straight line images provided as well as the same `test6.jpg` image we have been using. The results are shown below. In both cases, we can see the lanes are approximately parallel in the perspective transformed images. The source and destination points are plotted as quadrilaterals as well for visualization purposes.


![alt text][image4]

![alt text][image5]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for these tasks appears in the `Identify Lane Pixels and Fit Polynomials` section of the python notebook.

In order to identify lane pixels, I used a sliding window algorithm with 9 non-overlapping equal sized windows spanning the y-dimension of the image. The inital windows were centered on the two x-coordinates that had the most non-zero pixels in the bottom half of the image. One of these peaks would be less than the midpoint of the x-dimension and would therefore correspond to the left lane. Similar logic could be applied to the right lane. 

Nonzero pixels were identified within each of the sliding windows. In each iteration, the windows were recentered if there were a sufficient number of pixels found (`50` pixels in my case). These windows were arbitrarily set to be `100*2=200` pixels wide in the x-dimension and `720/9=80` pixels wide in the y-dimension.

Once these lane pixels were identified, I fit a second degree polynomial to the left and right pixels using `np.polyfit()`. I plotted the left pixels in red and the right pixels in blue. The fitted lines are plotted in yellow. Results for `test6.jpg` are shown below after a perspective transform.

![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calculated the radius of curvature and the vehicle position in the section titled `Calculate Curvature and Vehicle Position` in the notebook. To calculate the vehicle curvature, I used the following equation

![alt text][image7]


In this formula, `A` and `B` represent the coefficients of the second and first degree term in the polynomial of best fit, respecitvely. The variable `y` represents the y dimesion of the point on the line where you would like to calculate the curvature. In my case, I wanted to calculate the curvature corresponding to where the line intersects the bottom of the image at `y=720`. 

Note that the above equation is in units of pixels. Therefore I had to make some adjustments to convert it to units of meters. In particular, I used the conversion factors `ym=30/720` meters per pixel for the y-dimension and `xm=3.7/720` meters per pixel for the x-dimension. I scaled `A` by and `xm_per_pix/(ym_per_pix**2)` and `B` by `xm_per_pix/ym_per_pix`

In order to calculate the vehicle position, I first calculated the midpoint of the line segment connecting the two lane lines at the bottom of the image. I then took this number and subtracted it from the "expected center" of `1280/2=640`. I then used the conversion factor discussed previously to convert this position to meters.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in `Draw Lane Region on Non-warped Image` section of the notebook. Here is an example of my result on a test image:

![alt text][image8]

The rest of the images can be viewed in the output folder and are viewable within the notebook as well, as the output of a cell.

---

## Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_out.mp4)

---

## Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The lane line estimates were a bit unstable in areas where the road suddenly changed to lighter pavement and in the shadows as well. In order to make it more robust, we can add a post-processing step that averages the polynomial coefficients from several frames so the lane boundaries don't change too rapidly.
