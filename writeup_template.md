# Advanced Lane Finding Project

## Introduction
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

[image1]: ./output_images/step0.JPG "Undistorted"

[image2]: ./output_images/step1.JPG "Threshold"
[image3]: ./output_images/step2.JPG "Image ROI"
[image4]: ./output_images/step3.JPG "Unwraped ROI lane1"
[image5]: ./output_images/step4.JPG "Unwraped ROI lane2"
[image6]: ./output_images/step5.JPG "left and right lanes"
[image7]: ./output_images/step6.JPG "detect lane "

[image10]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

### My whole implemented algorigm pipline is decribed in:
* Advanced Lane Finding.ipynb
* Advanced_Lane_Finding.html

## Camera Calibration

* For extracting lane lines that bend it is crucial to work with images that are distortion corrected. 
* Non image-degrading abberations such as pincussion/barrel distortion can easily be corrected using test targets. 
* Samples of chessboard patterns recorded with the same camera that was also used for recording the video are provided in the `camera_cal` folder. 
* I start by preparing "object points", which are (x, y, z) coordinates of the chessboard corners in the world (assuming coordinates such that z=0). 
* Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image. 
* `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.
* `objpoints` and `imgpoints` are then used to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function. 
* I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

## Pipeline (single images)

* For color thresholding I worked in HLS space. 
* Only the L and S channel were used. I used the s channel for a gradient filter along x and saturation threshold, as well as the l channel for a luminosity threshold filter. 
* A combination of these filters is used in the function `binarize` 

![alt text][image2]

* A perspective transform to and from "bird's eye" perspective is done in a function `called warp()`. 
* The `warp()` function takes as input an color image (img), as well as the tobird boolean paramter. 
* This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 190, 720      | 340, 720      | 
| 589, 457      | 340, 0        |
| 698, 457      | 995, 0        |
| 1145, 720     | 995, 720      |

* I verified that my perspective transform was working as expected by drawing the src and dst points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image. 
* See the following image: 
![alt text][image3]

* Additionally, I included a region of interest that acts on the warped image to reduce artefacts at the bottom of the image.
* This region is defined through the function `region_of_interest()` and was tested using the wrappers `warp_pipeline(img)` and `warp_binarize_pipeline(img)`.
* An examples are shown below: 
![alt text][image4]
![alt text][image5]

* The function `find_peaks(img,thresh)` takes the bottom half of a binarized and warped lane image to compute a histogram of detected pixel values. 
* The result is smoothened using a gaussia filter and peaks are subsequently detected using. 
* The function returns the x values of the peaks larger than `thresh` as well as the smoothened curve.
* Then I wrote a function `get_next_window(img,center_point,width)` which takes an binary (3 channel) image `img` and computes the average x value `center` of all detected pixels in a window centered at `center_point` of width `width`. 
* It returns a masked copy of img a well as `center`.
* The function `lane_from_window(binary,center_point,width)` slices a binary image horizontally in 6 zones and applies `get_next_window` to each of the zones. 
* The `center_point` of each zone is chosen to be the center value of the previous zone. 
* Thereby sub-sequent windows follow the lane line pixels if the road bends. 
* The function returns a masked image of a single lane line seeded at `center_point`. 
* Given a binary image `left_binary` of a lane line candidate all properties of the line are determined within an instance of a `Line` class.

* The `Line.update(img)` method takes a binary input image `img` of a lane line candidate, fits a second order polynomial to the provided data and computes other metrics. 
* Sanity checks are performed and successful detections are pushed into a FIFO que of max length `n`. 
* Each time a new line is detected all metrics are updated. 
* If no line is detected the oldest result is dropped until the queue is empty and peaks need to be searched for from scratch.
* A fit to the current lane candidate is saved in the `Line.current_fit_xvals` attribute, together with the corresponding coefficients. * The result of a fit for two lines is shown below:
![alt text][image6]
![alt text][image7]

### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines # through # in `another_file.py`).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()`, which appears in lines 1 through 8 in the file `example.py` (output_images/examples/example.py) (or, for example, in the 3rd code cell of the IPython notebook).  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
