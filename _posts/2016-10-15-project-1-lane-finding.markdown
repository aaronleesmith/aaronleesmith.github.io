---
layout: post
title: 'Project #1 - Lane Finding'
date: 2016-10-15 19:34:41.000000000 -07:00
image: /assets/images/2016/11/Self-Driving.png
---
*My public fork of the lane finding project, complete with all the code contained below, can be found on [GitHub](https://github.com/thegreenpizza/CarND-LaneLines-P1)*

# An algorithmically driven approach to linear lane finding

This project involved taking a set of images from a vehicle driving straight along a straight road and finding, identifying, and extrapolating lane lines from raw frames.

## Overview
An input / output pair might look like this:

<table>
  <tr>
    <td>
      <img src="https://raw.githubusercontent.com/thegreenpizza/CarND-LaneLines-P1/master/test_images/solidWhiteCurve.jpg" alt="Raw image from the onboard camera"/>
    </td>
    <td><img src="https://raw.githubusercontent.com/thegreenpizza/CarND-LaneLines-P1/master/laneLines_thirdPass.jpg" alt="Image with lanes drawn."></td>
  </tr>
  <tr>
    <td style="text-align: center">
       <i>An unprocessed frame</i>
    </td>
    <td style="text-align: center">
       <i>A frame with lines automatically indicated</i>
    </td>
  </tr>
</table>

## Tools & Assumptions about the problem

To do this, the following libraries are used:

* OpenCV
* Numpy (for matrix math and simple linear regression)
* MoviePy (for inputting and streaming out video frames)

And the following concessions / assumptions are made:

* The algorithm is looking for lines, not curves.
* Images are in good weather and well-lit.
* At least one solid line is visible in the image.
* The camera is stationary and all videos have the camera at the same relative position.

Additionally, the following algorithms are used:

* Convolution with gaussian blur
* Morphological dilation
* Canny edge detection
* Hough transforms
* Linear regression

# Implementation
The algorithm is broken up into a pipeline of steps. They are executed in serial for each frame.

1. Calculate a left region of interest and a right region of interest based on the image dimensions.
2. Conver the image to grayscale and apply a small gaussian blur.
3. Run Canny edge detection to detect edges and give a black and white image.
4. Perform slight dilation on the image to bleed some lines which are close together into one line the hough transform will recognize.
5. Run a Hough transform on the edges in the left region of interest and right region of interest respectively.
6. Remove lines with unacceptable slopes.
7. Run linear regression once on the left line and once on the right.
8. Use some simple geometry (y = mx + b) to calculate extrema, then use the result of the linear regression to extrapolate to those extrema.
9. Draw the lines and return the image.

While this may seem like a lot, it all happens quite quickly and detects and draws lines in the above conditions quite well.

Here is the pipeline in action:

![The entire pipeline in action](/assets/images/2016/11/pipeline.png)

## Regions of interest
While it may be tempting to hardcode the best possible place to look for lines, lines can move around as you drive (well, as you move relative to the lines) and it would be unfortunate if they moved out of the ROI frame. 

For this reason, a simple ratio to setup polygons for left and right ROI is used:

{% highlight python %}
ratio = 5/8

roi_left = np.array([[
    (100, height),
    ((1 - ratio) * width, ratio * height),
    (.5 * width, ratio * height),
    (.5 * width, height)
]], dtype=np.int32)

roi_right = np.array([[
    (.5 * width, height),
    (.5 * width, ratio * height),
    (ratio * width, ratio * height),
    (width, height)
]], dtype=np.int32)
{% endhighlight %}

This is sort of a combination of knowing something around where the lanes will be, but still adjusting as the image changes size.

After applying these two regions on interest, here are the sections of the image the algorithm actually cares about:

![ROI's (left and right)](/assets/images/2016/11/roi.png)

## Image adjustments
To "clean" up the image for line finding, first a gaussian blur is applied, then Canny edge detection, and finally dilation.

{% highlight python %}
img = gaussian_blur(img, gaussian_kernel)
img = canny(img, canny_low_threshold, canny_high_threshold)
img = cv2.dilate(img, cv2.getStructuringElement(cv2.MORPH_DILATE, (5, 5)))
{% endhighlight %}

After applying the above adjustments, the following image is generated:

![Edge detection followed by dilation](/assets/images/2016/11/before_lane_finding.png)

Notice the edges are bled a bit by the dilation process.

After taking the ROI's, the following edges passed into the Hough transform:

![Edges in ROI's](/assets/images/2016/11/canny_roi.png)

## Hough transform
Up next is line finding via Hough transform. During this process the output gets a bit strange, since we are no longer viewing point in the (x, y) space but are now viewing points in the (rho, theta) space that the Hough transform operates in. I am not giving an overview of the Hough algorithm here, but for those interested see the [Into to Computer Vision](https://www.omscs.gatech.edu/cs-4495-computer-vision) from the OMSCS program.

The output of the Hough transform given the lines shown above shows lines on the right as bright spots in the Hough space, where many potential lines converge.

![Line finding Hough space](/assets/images/2016/11/hough_space.png)

There are more lines on the right that the algorithm is confident of and fewer on the left. Still, it will be able to use this output in the next step.

## Linear regression and extrapolation
The final step involves processing and making an educated guess as there where the lanes are, then using that same logic to draw lines spanning the lane edges, not just where the lane markings appear.

In my code I define two helper functions:

{% highlight python %}
# Takes an array of lines and gives back lines which have a valid slope, given a range.
def slope_filter(lines_array, positive, min_slope, max_slope):
slopes = np.apply_along_axis(lambda row: (row[3] - row[1]) / (row[2] - row[0]), 2, lines_array)

if positive:
    slopes[slopes > max_slope] = 0
    slopes[slopes < min_slope] = 0
    lines_array = np.array(lines_array[np.where(slopes > 0)])
else:
    slopes[slopes < -max_slope] = 0
    slopes[slopes > -min_slope] = 0
    lines_array = np.array(lines_array[np.where(slopes < 0)])

return lines_array
{% endhighlight %}

{% highlight python %}
# Performs simple linear regression on a set of lines.
def lines_linreg(lines_array):
x = np.reshape(lines_array[:, [0, 2]], (1, len(lines_array) * 2))[0]
y = np.reshape(lines_array[:, [1, 3]], (1, len(lines_array) * 2))[0]
A = np.vstack([x, np.ones(len(x))]).T
m, c = np.linalg.lstsq(A, y)[0]
x = np.array(x)
y = np.array(x * m + c)
return x, y, m, c
{% endhighlight %}

With these functions in mind, here is the logic to find two lines which represent the edges of lanes which span the road from top to bottom:

{% highlight python %}
# If we have no lines at all we give up.
if len(left) == 0 or len(right) == 0:
    return img

# Run linear regression and get slopes and intercepts.
left_x, left_y, left_m, left_c = lines_linreg(left)
right_x, right_y, right_m, right_c = lines_linreg(right)

# This variable represents the top-most point in the image where we can reasonable draw a line to.
min_y = np.min([np.min(left_y), np.min(right_y)])

# Calculate the top left and top right points using the slopes and intercepts we got from linear regression.
top_right_point = np.array([(min_y - right_c) / right_m, min_y], dtype=int)
top_left_point = np.array([(min_y - left_c) / left_m, min_y], dtype=int)


# Repeat this process to find the bottom left and bottom right points.
max_y = np.max([np.max(right_y), np.max(left_y)])
bottom_left_point = np.array([(max_y - left_c) / left_m, max_y], dtype=int)
bottom_right_point = np.array([(max_y - right_c) / right_m, max_y], dtype=int)

# Draw the lines.
cv2.line(img, (bottom_left_point[0], bottom_left_point[1]), (top_left_point[0], top_left_point[1]), [255, 0, 0], thickness)
cv2.line(img, (bottom_right_point[0], bottom_right_point[1]), (top_right_point[0], top_right_point[1]), [255, 0, 0], thickness)
return img
{% endhighlight %}

Finally, to call the draw_lines function, it is assumed that the lines are already filtered by slope to get just the lines of interest.

{% highlight python %}
lines_left = slope_filter(hough_left, False, .5, .9)
lines_right = slope_filter(hough_right, True, .5, .9)
color_with_lines = np.zeros(image.shape, dtype=np.uint8)
color_with_lines = draw_lines(color_with_lines, lines_left, lines_right, [255, 0, 0], 10)
{% endhighlight %}

# Final video output
The pipeline above works well on non-curved roads with the camera in a stationary position. Much more work will need to be done to make this work on curved roads, in different lighting and weather conditions, and with a camera that might be mounted at different points.

## White lanes
<iframe width="560" height="315" src="https://www.youtube.com/embed/GrVhvzBmKZc" frameborder="0" allowfullscreen></iframe>

## Yellow lanes (with white right lane)
<iframe width="560" height="315" src="https://www.youtube.com/embed/fJlPn7OVfPc" frameborder="0" allowfullscreen></iframe>
