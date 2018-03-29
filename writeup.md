# **Finding Lane Lines on the Road** 

[//]: # (Image References)

[white_select_image]: ./test_images_output/white_select_solidYellowCurve2.jpg "WhiteGraySelect"
[yellow_select_image]: ./test_images_output/yellow_select_solidYellowCurve2.jpg "YellowGraySelect"
[color_select_image]: ./test_images_output/color_select_solidYellowCurve2.jpg "ColorSelect"
[edge_image]: ./test_images_output/edge_solidYellowCurve2.jpg "Edge"
[region_select_image]: ./test_images_output/region_select_solidYellowCurve2.jpg "RegionSelect"
[line_segment_image]: ./test_images_output/line_segment_solidYellowCurve2.jpg "LineSegment"
[line_image]: ./test_images_output/line_solidYellowCurve2.jpg "LaneLine"
[annotated_image]: ./test_images_output/solidYellowCurve2.jpg "LaneLine"

## The Overview

In this project, I designed a pipeline to process the videos taken from car on different road condition, identified the left and right lanes the car is in, and draw an annotation lines to mark the lanes found. The left lane and right lane are identified almost all the time except in two frames of the challenge video, the right lane is not identified.

---


## The Pipeline

My pipeline for finding the lane consists of 5 major steps:

---

### 1. Color Filter and Gray Scale

Because the lanes are all in yellow or white color in the videos, I use color filtering to first filter out the information we're not interested. This step is not very critical for solidWhiteRight.mp4 and solidYellowLeft.mp4 but is necessary for the challenge.mp4. The reason is because the challenge.mp4 has tree shadow on the road, and the edges along the left side dividor. Those information can't be filter out by the pipeline below and will eventually be identified as line segments as the lane. 

In this step, I did the follow:
### *Filter the white color*
First convert the image from RGB format to HSV format. Then select the white color with specific HSV range. Because in challenge video I noticed some times the lane segment under the tree shadow were not found, I converted the image to grayscale and then use cv2.THRESH_BINARY to convert the intensity from 200 above to 255, and intensity below 200 to 0. I hope this threshold conversion can make the white lane more stand out for the pipeline afterwards. It does seem help a little bit. 

The picture below showed the image with white color selected in grayscale:

![alt text][white_select_image]

---

### *Filter the yellow color*
Do the similar color selection for yellow color. The picture below showed the image with yellow color selected in grayscale:

![alt text][yellow_select_image]

---

### *Combine two images*
Then I combined the white selected image and yellow selected image together with cv2.bitwise_or function.

![alt text][color_select_image]

---

### 2. Edge Detection

### *Gaussian Blurring*
First use gaussian_blur() function with kernal size 7 to smooth out some noise. I didn't find the size of kernel matter too much. 

### *Canny Edge*
Then use canny edge detection algorithm to convert the image to edge image as shown below.

![alt text][edge_image]

---

### 3. Region Select

After the edge detection and conversion to edge image, I use region selection to select the interested area of the lanes. This is useful to filter out the nearby lanes and the nearby white or yellow cars. I defined a trapezoid region with four corners:
1. $(0, height)$ -- left lower corner
2. $(length, height)$ -- right lower corner
3. $(length\times0.5 + 50, height\times0.55)$ -- the right upper corner
4. $(length\times0.5 - 50, height\times0.55)$ -- the left upper corner

The is the image after region select
![alt text][region_select_image]

---

### 4. Line Segment Detection

With a region select image, the information in the image are mostly the lane segments. Use the hough line detection to detect the line segments of the lanes. 

![alt text][line_segment_image]


---

## Improvements to draw_lines() 
After the line segments were detected successfully, I modified the draw_lines() function to improve the final result. Now the image with the improved line drawing looks like the following image:

![alt text][line_image]

The improvement I did are:
---
### Filter out the nearly horizontal lines
I noticed in the video, somethings there are horizonal lines with the same color and shape as the lane lines. However, they're not lane lines, they're just some markers from the lane. So I calculate the slope of all the line segments, if the absolute value of the line's slope is less than 0.2, I assume that line is too horizontal to be a lane segment. I just drop it.

---
### Partition the line segments into left lane and right lane
I group the line segment points into left lane and right lane by the slope of the line segment. If the slope is negative, it belongs to the left lane, otherwise, it belongs to the right now. We already filter out the small slope line segment in previous step, so slope can't be zero in this step. In fact, the left lane segments have slope less than -0.2, and the right lane segments have slope greater than 0.2.

In addition, I also partition the line segments by their position. If a line segment's slope showing it belonging to the right lane, but its position shows it's on the left side of the image, I assume it is noise instead of lane line segment. It will be dropped. Similar policy also applies to left lane segment filtering.

---
### Extrapolate the lane segments
After the line segments filtered and grouped the points on line segments into left lane and right lane groups, I then use the np.polyfit function to extrapolate the linear line fit to all the points on left lane and right lane. After the linear function is found for left lane and right lane, we find the slope and the intercept of the left lane and right lane. Then I choose the two end points for each lane. The points are selected by the Y value, one point has $Y=length$, and the other point has $Y=length\times0.65$. The X value is calculated from Y value with the slope and intercept of the lines.


## The Annotated Image

With the pipeline and the drawline() improvement, the lane lines are found. Add the lane lines to the original image, we can get the image with the lane lines annotated.

![alt text][annotated_image]

---

## The Limitations

There are a few limitations of this solution:

1. The extrapolation algorithm assumes straight line for the lanes. But in a very curved road, it won't be a straight line for the lanes, this algorithm can show some weird result. In fact, for challenge video, it already shows not working well with curves.

2. The parameters for each stage of the pipeline are hard coded. I had spent lots of time to try different parameters, sometimes it works well with the test images, but then it doesn't work well on some frames from videos. So I think the hard coded parameters won't work well in all scenarios. When the light condition changes, or the road changes, the hard coded parameters can't adapt to all the cases.

3. The region select algorithm defines a trapezoid zone in front of the car. But quite often, there are markers on the road, for example EXIT or STOP words are written on the group, they're often in white/yellow color and they have clear edge. It is hard to filter them out by the current pipeline.


## Future Improvements

### Line Extrapolation 

The line extrapolation algorithm can be improved to consider the curved road. So instead of drawing a single straight line, maybe it can draw multiple shorter straigh lines to form a curved lane line.

### Guess the Lane Lines
In the challenge video, my algorithm has some trouble to detect the lane lines in a few frames. It might be able to be improved by better tuned parameters, however, I tried to tune different parameters but it doesn't help much. In real life, sometimes the lane lines can be worn out and not be visible. But human can guess the lane lines based on previous lane lines location. I think self-driving car should also be able to guess the lane lines when we can't detect any lane line. The guess algorithm can guess the position of the lanes based on previous frames assuming the road should not have a sharp change in a second. It can also identify other objects along the road like the curb, or the dividor, or even other cars. Based on the changes of the other objects, it can guess the lanes' changes from previous frames.

### Adaptive Parameters
The parameters in the pipeline can't be hard coded for all the road and light conditions. The color filtering algorithm definitely needs to change the threshold based on the actual color of the lanes and the light condition. The light can give elusion of the colors.

### Region Select
The region select algorithm can be improve to select a better defined region so we can filter out the markers on the group. For example, we can select the region for left lane only, and right lane only. The region in the center in front of us should not be lane, and it should not be selected. 