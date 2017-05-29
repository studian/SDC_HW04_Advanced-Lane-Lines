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

[image8]: ./output_images/test_results/detected_lane_straight_lines1.jpg "result1"
[image9]: ./output_images/test_results/detected_lane_straight_lines2.jpg "result2"
[image10]: ./output_images/test_results/detected_lane_test1.jpg "result3"
[image11]: ./output_images/test_results/detected_lane_test2.jpg "result4"
[image12]: ./output_images/test_results/detected_lane_test3.jpg "result5"
[image13]: ./output_images/test_results/detected_lane_test4.jpg "result6"
[image14]: ./output_images/test_results/detected_lane_test5.jpg "result7"
[image15]: ./output_images/test_results/detected_lane_test6.jpg "result8"

[video1]: ./output_images/test_results/detected_lane_project_video.mp4 "Video1"
[video2]: ./output_images/test_results/detected_lane_challenge_video.mp4 "Video2"
[video3]: ./output_images/test_results/detected_lane_harder_challenge_video.mp4 "Video3"

### My whole implemented algorigm pipline is decribed in:
* Advanced Lane Finding.ipynb
* Advanced_Lane_Finding.html

---

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

* The radius of curvature is computed upon calling the `Line.update()` method of a line. 
* The method that does the computation is called `Line.get_radius_of_curvature()`.
* For a second order polynomial f(y)=A y^2 +B y + C the radius of curvature is given by R = [(1+(2 Ay +B)^2 )^3/2]/|2A|.
* The distance from the center of the lane is computed in the `Line.set_line_base_pos()` method, which essentially measures the distance to each lane and computes the position assuming the lane has a given fixed width of 3.7m.
* This last function `process_image(img)` handles all lost lane logic. 
* Here is an example of the result on a test image:
![alt text][image8]
![alt text][image9]
![alt text][image10]
![alt text][image11]
![alt text][image12]
![alt text][image13]
![alt text][image14]
![alt text][image15]

---

## Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).


Here's a [link to my video (project_video) result #1](./output_images/test_results/detected_lane_project_video.mp4)
Here's a [link to my video (challenge_video) result #1](./output_images/test_results/detected_lane_challenge_video.mp4)
Here's a [link to my video (harder_challenge_video) result #2](./output_images/test_results/detected_lane_harder_challenge_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
