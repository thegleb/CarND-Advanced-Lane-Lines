## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image0]: ./writeup_images/original_checkerboard.jpg "Original checkerboard pattern"
[image1]: ./writeup_images/undistorted_checkerboard.jpg "Undistorted checkerboard"
[image2]: ./writeup_images/original.jpg "Original road image"
[image3]: ./writeup_images/undistorted.png "Undistorted road image"
[image4]: ./writeup_images/white_thresholds.png "Thresholds for finding white lane markers"
[image5]: ./writeup_images/yellow_thresholds.png "Thresholds for finding yellow lane markers/lines"
[image6]: ./writeup_images/magnitude_gradient.png "Magnitude gradient via Sobel operator"
[image7]: ./writeup_images/directional_gradient.png "Directional gradient via Sobel operator"
[image8]: ./writeup_images/all_image_filters.png "All together: yellow OR white OR (magnitude AND gradient)"
[image9]: ./writeup_images/lane_image_before_warp.png "Lane image before warping"
[image10]: ./writeup_images/lane_image_after_warp.png "Lane image transformed into a top-down perspective"
[image11]: ./writeup_images/sliding_window.png "Sliding window search"
[image12]: ./writeup_images/final_output.png "Final output"

[video1]: ./project_video_with_lane_lines.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this is in the 2nd code cell of the notebook at `P2.ipynb`.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![Original][image0]
![Undistorted][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

`cv2.undistort(img, mtx, dist, None)` makes it easy to get a distortion-corrected image. `mtx` and `dist` values come from the `cv2.calibrateCamera()` call described earlier. Here is a road image before and after distortion correction:

![Original road][image2]
![Undistorted road][image3]


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I broke down image processing into several steps:
* Transform image to the HSV color space for color processing
* Use custom thresholds on the HSV image to extract white and yellow lane markers
* Transform image to grayscale to apply Sobel operator
* Perform Sobel operator for both magnitude and directional gradients
* Binary AND for magnitude and directional gradients
* Binary OR the gradients above with the white and yellow color thresholds

This is done in the `find_edges(img)` method in `P2.ipynb`, with some of the sub-items such as the sobel filtering done separately in the `sobel(...)` method.

![Thresholds for finding white lane markers][image4]
![Thresholds for finding yellow lane markers/lines][image5]
![Magnitude gradient via Sobel operator][image6]
![Directional gradient via Sobel operator][image7]
![All together: yellow OR white OR (magnitude AND gradient)][image8]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in lines 1 through 8 in the file `example.py` (output_images/examples/example.py) (or, for example, in the 3rd code cell of the IPython notebook).  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

The code for my perspective transform is done globally (not super clean, I know), near the bottom of the `P2.ipynb` notebook in the last code cell. I created a trapezoid that sort of follows parallel lines to match perspective in the main video:

```python

img_transform_min_y = 450
img_transform_top_center_offset = 49
img_transform_bottom_side_offset = 150

src = np.float32([
    [img_transform_bottom_side_offset, img_height],
    [img_width / 2 - img_transform_top_center_offset, img_transform_min_y],
    [img_width / 2 + img_transform_top_center_offset, img_transform_min_y],
    [img_width - img_transform_bottom_side_offset, img_height]
])
dst = np.float32([
    [300, img_height],
    [300, 0],
    [img_width - 300, 0],
    [img_width - 300, img_height]
])

```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 591, 450      | 330, 0        | 
| 150, 720      | 330, 720      |
| 1130, 720     | 950, 720      |
| 689, 450      | 950, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![Filter image before warping][image9]
![Filter image transformed into a top-down perspective][image10]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I took a pretty naive lane-line pixel finding approach in the `find_and_update_lane_lines()` method. Essentially on the first run I do a sliding window search (minimum threshold of 100 pixels to move the next window over), and on subsequent runs I iterate over the previously found line and find pixels within a certain margin of the existing line.

In the rare case that we were not able to find something like a polynomial, we reset the search back to a sliding window search. In my experience we don't really trigger the fallback (we are able to find the lane lines just fine after the first frame) so the test video did not require additional tuning here.

Here is an example picture demonstrating debugger output of the sliding window search (I did not implement debug output for the second type of search):

![Sliding window search for pixels][image11]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

First we have to calculate the lane line fits in "real world" scale. I used a couple assumptions to create a pixel-to-meter conversoin factor in both the X and Y directions. I used the average width of a lane (12 feet) and dashed lane line indicator dimensions to estimate a real-world distance in both dimensions.

For curvature I average the radius of curvature for the 2 lines representing the sides of the lane since they're not "parallel" or have the same curvature down to the individual pixel.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I wrote a method called `draw_lane_on_image()` which takes a blank image and draws the lane line on it in the form of polylines as well as a polynomial.

Since this is unwarped immediately after, to also plot the curvature and car's offset on the final image we have to do this after the unwarp step. This is done in `annotate_image()`; this method is also where we calculate the final curvature and offset values because it is one place in the code where we have both lines together, so it is a decent place to summarize some values based on both.

![Final output][image12]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_with_lane_lines.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The main problem for me developing a solution is the speed of the pipeline. At peak it was processing ~4-5 frames per second, at its slowest it processes 1 frame every ~3 seconds, so I had to test various parameters on short video clips, and re-render the entire video any time I thought I had a working solution.

As a result I didn't spend a lot of time looking at the challenge video results (which start off fairly promising); it was taking far too long to test so I decided to optimize the main solution instead.

The other limitations of my current pipeline, in no particular order:
- My definition of "line not found" is very simplistic - a line is considered as not found if we couldn't fit a polynomial to it. The pipeline doesn't check curvature or approximate where the left or right lane line is "supposed" to be. This would be one point of improvement.
- The warp transform is very dependent on one angle of the camera. For example, the challenge video looks to be filmed from a slightly different perspective, so we have to re-calculate the transform for that specific case. A more automated case that can calibrate itself from a straight-line view could be helpful.
- I am not super happy with my sobel filter skills, as that part of the filtering gets pretty useless in parts of the (easier) challenge video. The parts of that video where the car is in the tunnel basically become pretty useless with the filter setup I have currently. Color filters on an otherwise unmodified image have not been super useful in my testing, so further work would be needed to get that part of the challenge to work.