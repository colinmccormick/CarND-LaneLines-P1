# Finding Lane Lines on the Road

This project is part of the [Udacity Self-Driving Car Nanodegree program](https://github.com/udacity/CarND-LaneLines-P1).

The goal of this project is to write a data pipeline to find lane lines on the road in individual images and video frames. The recommended approach is to use fairly standard computer vision techniques, particularly the [OpenCV](http://opencv.org) library. My pipeline works well on the test images and the two primary video examples. I have not tried to improve it to work on the challenge video.

[image1]: ./test_videos_output/solidWhiteRight.mp4 "Video"
![alt text][image1]

### Reflection

My pipeline uses the following steps:

1. Grayscale the image using OpenCV (cvtColor). 

2. Apply a Gaussian blur using OpenCV (GaussianBlur). This smoothes out any spurious pixels from noise in the camera or other sources. These could lead to false edge detections in the next step if we don't deal with them now. This step requires one parameter, the size of the Gaussian kernel in pixels. My pipeline works well with the value 5.

3. Detect edges in the image using the Canny transform in OpenCV (Canny). This requires two paramters, a high and low threshold, which I determined experimentally. My pipeline works well with the values 150,50.

4. Create a masked image, with all pixels outside a region of interest (ROI) set to zero (black). This uses the approach in the provided helper function, leveraging two OpenCV functions (fillPoly, bitwise_and) and multiple steps. First, a new image is created with all pixels set to zero. Second, the polygon region defined by a set of vertices (the ROI) is set to white (255). Third, this new image and the processed image from the previous step are combined with a bit-wise and, retaining the original value within the ROI and setting all other pixels to zero. 

	To create the ROI, I specified three parameters: the height, and the upper-left and upper-right positions, all as fractions of the image height and width. After playing around a little, I found the pipeline worked well with a height of 0.6 (remember that y increases downward, so this is below the vertical mid-point) and left/right positions of 0.47,0.53.

5. Detect lines. For the first part of the project the pipeline simply detected line segments, using OpenCV's Hough transform (HoughLinesP). This requires several parameters, some of which are default in the provided helper function. The others arethe threshold, minimum length, and maximum gap for detected segments. I set these empirically after some trial and error; the pipeline works well with 32,16,16.

	For the second/main part of the project, the pipeline needs to combine these segments into a full lane line. To do this, the pipeline uses the function hough_lines_extended(). After detecting line segments, this function calculates the slope and y-intercept of each one, using the simple formula for a line, y = m*x + b. Next, it sorts segments into left or right lanes, depending on the slope: positive values are the right lane, and negative values are the left lane. It then finds the average (mean) value of the slope and y-intercept values for left-lane and right-lane segments, and uses this as the definition of the true/complete lane line. To draw this on the image, it calculates the (x,y) points of this line at the bottom and top of the ROI.

	This method worked well on the first video, but had some trouble with the second video, where spurious non-lane lines appear on the road (particularly at 0:11). These impact the calculation of the mean, and lead to incorrect averages. To handle them, I defined a range of expected values for the slope of the true lane lines (slope_high and slope_low), based on examining the test images. For each line segment detected by the Hough transform, if the (absolute value of) the slope is outside of this range, that segment is discarded and not included in the calculation of the average lane lines. This seems to work well, and improves the accuracy of the pipeline for the second video, with values 1.0,0.5.

### Potential shortcomings

**Faded/obscured lane markings**. In real road conditions, we might not have such well-painted lane markings with good constrast, and the edge detection step might fail.

**Grayscaling**. By using only the grayscaled image, I am throwing away potentially useful color information that could help identify the lane lines. I found this was fine for these examples, but in more difficult conditions (e.g. shadows, curving lanes, etc) it might be necessary to use color information as well.

**Rejecting out-of-range slopes**. This hard-coded method of rejecting spurious lines by testing their slope might fail if the lane is curving sharply, and the true slope of a lane marker falls outside the specified range. It might also fail if there are spurious lines on the road from tire marks, etc. I initially tried an approach of calculating the median of the distribution of all slopes and rejecting any outliers (i.e. more than 2 standard deviations away) but this fails if there is a small numberof line segments, so I instead used this out-of-range method.

**Averaging slopes and intercepts**. This method is most accurate when dealing with straight roads that have lane markers lying along the same true line. That's the case for our current examples. However, on curved roads the lane markers would not be in a straight line, and averaging their slopes and intercepts would produce only a straight line, not the true curve. In that case we would have to use a more sophisticated approach.

### Possible improvements to the pipeline

**Lane marker memory**. In the current pipeline we process each frame independently, with no input/memory from the previous frame. That's discarding potentially useful information, since the real lane markers can't be too far from where they were in the previous frame. We could potentially keep a running list of the lane marker positions in previous frames and use this to accept/reject estimates of lane marker segments in the current frame.

**Color processing**. As noted above, we discard color information by grayscaling. We could instead implement a simple color-based method by applying the entire pipeline independently to each of the RGB color channels, and then average the resulting lane lines to get the final result. This might protect against spurious line markings, particularly if they are mostly one color. The drawback is increased processing complexity, and possibly reducing our accuracy with yellow line markers.

**Improved/adaptive edge detection**. The pipeline relies on hard-coded thresholds for the Canny edge detection. There are many improvements that can be made to the default implementation of the Canny method (Wikipedia has a [good summary](https://en.wikipedia.org/wiki/Otsu%27s_method)). These seem fairly challenging to implement, and with the current example images, the edge detection step doesn't seem to be a problem, so this would be low-priority.

**Modified image masking**. The approach taken in the provided helper function works, but it seems needlessly complex. It creates a second (zero-valued) image, fills the polygon defining the lane with a white value, and then uses a bitwise-and function to retain pixel values only in that region. I suspect the course designers chose this approach becaues it makes defining the vertices easy: it's just four points that define the shape of the lane ahead (a trapezoid). But I think it would be conceptually easier overall to instead specify a slightly more complex set of vertices (six) that define the non-interesting region - i.e. everything but the lane ahead. Then we can simply set pixels in that region to zero, and not use a bitwise and. I tested this approach (region_of_interest_modified()) and it seems to work well.
