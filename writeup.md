#**Finding Lane Lines on the Road** 

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report

[//]: # (Image References)

[original]: ./writeup/original.jpg "Original"
[colorMask]: ./writeup/color-mask.jpg "Color mask"
[canny]: ./writeup/canny.jpg "Canny edge detection"
[hough]: ./writeup/hough.jpg "Hough transform"
[regionOfInterest]: ./writeup/region-of-interest.jpg "Region of interest"
[processed]: ./writeup/processed.jpg "Processed image"

---

### Reflection

###1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

The image processing pipeline consists of the usual steps explained in the lecture materials, as well as some additional steps we found useful for solving the problem, particularly the `challenge` video problem.

Throughout the writeup we will show examples of the results of different image processing steps on an image. The original image is here:

![alt text][original]

Before any manipulation is done to an image, we produce a [hue-saturation-value](https://en.wikipedia.org/wiki/HSL_and_HSV) (HSL) mask of the image, showing which parts of the image matched specified HSL ranges we specified. While we were able to get acceptable results for the main images in the `test_images` folder without this step, we found this extra step was an important part of my solution for the `challenge` video, to remove extraneous detected lines that did not correspond to lane lines. We used HSL instead of RGB as a filter because using hue and lightness to define ranges of colors corresponding to orange and white lane lines, respectively, was easier than using RGB ranges. An example of the color mask produced by this technique can be seen below. After producing the mask, we apply a Gaussian kernel to make the acceptable region a bit larger surrounding the passed-through values.

![alt text][colorMask]

Similar to the hard-coded (in the lectures and quizzes) inclusion region, my HSL mask allows edge-finding and ultimately lane line finding only in specific regions of the image. The regions correspond to orange or white lane lines, or colors similar to these lane lines. HSL specifically (as opposed to the more common RGB representation of an image's colors) was used because specifying a hue or brightness ("value") range was more effective at locating lane lines than specifying RGB ranges would have been. See discussion [here](https://en.wikipedia.org/wiki/HSL_and_HSV).

After constructing an HSL mask for later use, we defined a hard-coded "region of interest" to limit what region of the image was used to find edges for lane line detection. This region was roughly a trapezoid with the angled sides roughly parallel to lane lines, the bottom side near the bottom of the image, and the parallel smaller top side about halfway up the image around the level of the horizon. Like the HSL mask, this region would define an area of the image from which edges would be detected that ultimately informed the parameters of the detected lane lines. The original image with the region drawn on top is shown below.

![alt text][regionOfInterest]

Next, we perform [Canny edge detection](https://en.wikipedia.org/wiki/Canny_edge_detector) over the whole image. This process finds the edges in an image. The parameters of the Canny edge detector were adjusted so that the lane lines apparent in the image were easily visible, while attempting to minimize extraneous edges in the region of interest / road area. An example image of the result of performing Canny edge detection is shown below.

![alt text][canny]

After we have an image showing edges from Canny edge detection, we perform a [probabilistic Hough transform](https://en.wikipedia.org/wiki/Randomized_Hough_transform) on the image to detect the lines revealed by the Canny edge detection process. Like Canny edge detection, we adjusted the many parameters of the Hough transform to achieve desirable and effective results. The goal was to detect all the relevant lane lines, while avoiding extraneous and irrelevant lines, if possible. An example image showing the detected Hough lines is below.

![alt text][hough]

The output of the Hough transform is a collection of definitions of lines, as defined by pairs of points (`(x1, y1, x2, y2)`). The pipeline uses some helper methods to perform analysis on these points. First, slopes of the lines were extracted using the simple [definition of a line](https://en.wikipedia.org/wiki/Linear_equation), discarding vertical lines with infinite slopes for simplicity. Intercepts for the lines were similarly extracted.

In order to detect the lane lines, we first separate the Hough lines into two separate groups. To achieve this separation, lines with positive slopes and lines with negative slopes are separated. Additionally, lines that match the slope criteria but fall in the opposite region of the image than expected are discarded (e.g, a line with a slope matching the left lane line but appearing  in the right side of the image). Both of these steps helped reduce extraneous lines not associated with an actual lane line.

Once these groups are obtained, lines whose location (calculated as the average of the line's two points) fall outside the hard-coded trapezoidal region, or outside the HSL color mask, are discarded. This helps further ensure that we are only examining Hough lines that correspond to actual lane lines in the image. The HSL mask in particular was especially useful in more challenging images with a lot of noise (i.e. extraneous detected edges in the image).

Using this filtered set of lines (with slopes and intercepts collected separately), a linear polynomial fit using `NumPy`'s `polyfit` function is performed to determine the best fit line that falls through the locations of the remaining Hough lines, one fit for each lane line / side of the image. This best-fit line is the final result of a single-image analysis, and the line is plotted on top of the image. 

For video analysis, there are additional challenges. Our best-fit line produces a reasonable but noisy result for video analysis. I.e., the slope and intercept of the best-fit line varies frame-by-frame by small amounts (a "jittery" effect). Additionally, in the analysis of a video (which are really just many images in rapid succession), there are occasionally single frames/images with poor results. To address both of these issues, we compute a running median using the last five computed slopes and intercepts and use that as our detected lane line instead of the single-image result. Using this rolling median improves the analysis of a video dramatically, both in smooth frame-by-frame appearance of the line, and, more important, the reduction of highly deviant results from single, difficult-to-fit frames.

A final version of a processed image can be seen below, with the lane lines and region of interest drawn. Additionally, the mean locations of Hough lines are drawn under the lane line best-fit.

![alt text][processed]

###2. Identify potential shortcomings with your current pipeline

There are several shortcomings using this method. Most of the shortcomings are related to hard-coding or "overfitting" of our pipeline to our example data. In tuning the model, parameters of the Canny edge detection, the Hough transform, and, more dramatically, color filtering and hard-coded "region of interest", are all specific to our "training" data. Ideally we would have a more rigorous approach when it comes to training, validation, testing, and real-world use.

For example, unexpected changes in the pointing of the camera could have dramatic effects. "Hard-coded" assumptions exist about: lanes being on one half of the image or the other, how orange or white lane lines are, and the general (trapezoidal) region of interest that line lines exist in the image. Very different lighting conditions, or different-colored lane lines (e.g. international or regional lane lines) could affect which regions of the image are allowed for analysis due to our HSV filter described above. If the camera happens to be pointed in a substantially different orientation than expected, our hard-coded regions of interest and assumptions about lane line locations based on left- and right-sides of the image would be violated.

For a self-driving car, we would want to make a much more generalized and robust solution for detecting lane lines.

###3. Suggest possible improvements to your pipeline

One big improvement would be to analyze a much larger dataset, and formally split the data into training and validation sets to evaluate quality of fit. In fact, any quantitative analysis of goodness-of-fit would be an improvement over the current situation, where goodness of fit is only evaluated qualitatively by eye. A labeled data set could be created by humans drawing their own lane lines on images or videos, and testing the algorithm's correctness based on those values would be an improvement.

Dynamically determining a region of interest somehow would also be an improvement, as the hard-coded trapezoidal region is fragile to changes in camera orientation.

Exploring parameter space, either by brute force or a more efficient optimization algorithm, for Canny edge detection and Hough transform parameters, using a quantitative analysis of correctness as described above, would also be an improvement over hand-tuning parameters.
