# Vehicle Detection
[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)


The goal of this project is to write a software pipeline to detect vehicles in a video stream obtained from a camera mounted on a car while driving.

---

The goals / steps of this project are the following:

* Perform a Histogram of Oriented Gradients (HOG) feature extraction on a labeled training set of images and train a classifier Linear SVM classifier
* Optionally, you can also apply a color transform and append binned color features, as well as histograms of color, to your HOG feature vector. 
* Note: for those first two steps don't forget to normalize your features and randomize a selection for training and testing.
* Implement a sliding-window technique and use your trained classifier to search for vehicles in images.
* Run your pipeline on a video stream (start with the test_video.mp4 and later implement on full project_video.mp4) and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

[//]: # (Image References)
[image1]: ./output_images/sample_images.png
[image2]: ./output_images/hog_outputs.png
[image3]: ./output_images/sliding_window_sizes_image.png
[image4]: ./output_images/sliding_window_test_images.png
[image5]: ./output_images/thresholded_heatmap.png
[image6]: ./output_images/labeled_test_images.png
[video1]: project_video_processed.mp4

## [Rubric](https://review.udacity.com/#!/rubrics/513/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  

You're reading it! 

`Note: ` All the code for this project is contained in the Jupyter Notebook - `Vehicle_Detection.ipynb`

###Histogram of Oriented Gradients (HOG)

####1. Explain how (and identify where in your code) you extracted HOG features from the training images.

I started by reading in all the `vehicle` and `non-vehicle` images utilizing the `glob` library in python. I randomly selected 3 images from each category which are shown below. 
The code for this step is contained in the `second and third code cell` of the Jupyter notebook. 

![alt text][image1]

In the `fourth code cell` of the notebook, I define the helper functions to assist with extracting features using HoG, Spatial Binning, and Color Histogram. The 
`extract_features` function iterates over all images passed into the function and extracts features by calling the `color_hist`, `bin_spatial`, and the `get_hog_features` functions.
The `extract_features` function is called for the list of images, which contain a car and the list which doesn't, in the `sixth code cell`. The results of these extracted features
for each set of images are then stacked and normalized (using sklearn's `StandardScaler` method) and then split into training and testing datasets (using sklearn's `train_test_split` method).
The code for this is contained in the `seventh code cell` of the notebook. 




####2. Explain how you settled on your final choice of HOG parameters.

I iterated over several colorspaces without much changes to the different parameters required for `skimage's hog method`. I based my selection eventually on the test set accuracy I achieved
after training my classifier. This isn't a very robust metric to identify what combination would work the best, but coupled with observing the results of searching and classifying vehicles in the test images, the following parameters yielded
good results. These parameters are defined in the `fifth code cell` in the notebook.

* cspace = 'HSV'
* spatial_size = (16, 16)
* hist_bins = 32
* hist_range = (0, 256)
* orient = 8
* pix_per_cell = 8
* cell_per_block = 2
* hog_channel = "ALL"

Here is an example using the `HSV` color space, all channels, and HOG features extracted based on the above parameters.


![alt text][image2]

####3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).

Once I obtained my training and test set from extracted features, as explained above, I utilized `sklearn's Linear Support Vector Classification (LinearSVC)` to train a classifier.
This was implemented in the `8th code cell` of the notebook, and I obtained a test accuracy of 99.07%

###Sliding Window Search

####1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

All of the helper functions for this section are available in the `9th code cell` of the notebook.

This part of the algorithm took up quite a bit of trial and error, and after a while it became obvious that there could be numerous configurations and the window size and the overlap
were important for the entire pipeline to perform well, perhaps even more so than fine-tuning the classifier after a point.

I went for a big tradeoff here. I decided to go for relatively big window sizes, with a 0.75 overlap for all of them. The tradeoff here was in terms of processing speed.
I noticed that I could perhaps get better results with smaller window sizes, and higher overlap than this, but that resulted in my video processing times to be quite high (even hours). 

The current configuration allowed me for quick testing on the final video as well (mostly 1.5 to 2 fps) while not sacrificing too much on the results. Although there are small spots
where one of the cars wasn't tracked too well. 

| Window Sizes	| X-range	    | Y-range	    | Overlap	    |
|:-------------:|:-------------:|:-------------:|:-------------:|
| 64 x 64		| (640, 1280)   | (360, 450)   	| (0.5, 0.5)	|
| 120 x 120		| (640, 1280)	| (360, 450)   	| (0.75, 0.75)	|
| 140 x 140		| (640, 1280)	| (400, 514)   	| (0.75, 0.75)	|
| 160 x 160		| (640, 1280)	| (450, 600)   	| (0.75, 0.75)	|

I further restrained the windows to just the right half of the image/frame. This does limit my model performing well for only this specific project video, but it was immensely helpful 
in reducing false positives.

The following displays the sliding windows overlayed on a test image.

![alt text][image3]

The next section covers the remaining pipeline for individual images.

####2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to try to minimize false positives and reliably detect cars?

In `code cell 10` of the notebook, I run my pipeline for individual test images. I initially defined the sliding windows I discuss above. I then call the `search_windows` function which
was defined in the `9th cell` of the notebook. This function (based off of the functions we covered in the course) identifies windows in the image(s) where the classifier detects a vehicle.

Following are the results of the above.
![alt text][image4]

I then created a heatmap that takes in the bounding boxes per image, and for those regions in a new blank image, adds a constant value for every box. This results in identifying where
the classifier predicts the highest probability of a car being present in the image. However, there are still some false positives with this heatmap. As a result, a thresholding operation
is carried out to reduce those. Following is the result of this process -

![alt text][image5]

The result of `scipy.ndimage.measurements.label()` labels the regions that are connected together in the thresholded heatmap. Using this result, a bounding box is drawn around the detected
vehicle using the `draw_labeled_bboxes` function defined in `code cell 9`. Here are the results of this pipeline on the test images.

![alt text][image6]

---

### Video Implementation

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (somewhat wobbly or unstable bounding boxes are ok as long as you are identifying the vehicles most of the time with minimal false positives.)
Here's a [link to my video result](./project_video_result.mp4) named `project_video_result.mp4` [also attached].


####2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.

The code for the video pipeline in the `process_video` function in the `11th code cell` of the notebook. 

Most of the pipeline that I implemented for the individual images above was utilized for the video as well. 

There were couple prominent additions that I made. 

The results of the above pipeline led to some missing frames and a lot of jittery bounding boxes.

To reduce the jitter I added a weighted moving average, defined by - 
`heatmap_image = prev_heatmap * heatmap_factor + heatmap_image * (1 - heatmap_factor)` where the heatmap_factor was set to `0.9` after a bit of experimentation. 

The `prev_heatmap` was stored in a global variable. The new heatmap image was then thresholded again. 

This resulted in  some good results, but I was still missing detection at some frames. Now, the best solution would have been to go tinker around with the sliding window sizes.
That would have made a lot of good changes. But fine-tuning that is more time-consuming as per me.

Instead I decided to keep track of the last 15 frames, for which I added up the heatmaps. I then thresholded this heatmap and again fed it to the weighted moving average equation above.

The results I obtained were better than before as you can see from the video.


---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Most of the problems or issues with the implementation was tuning the parameters and finding the right combinations. Furthermore, a more robust solution would not work in real-time.

The results I get aren't perfect, of course.
* Using global variables is bad programming practice as per me. I next plan to implement a proper class to handle the detection parts.
* My implementation doesn't generalize well on any other video. Primary improvement area as per me would be the sliding window sizes, or using a DL based approach in it's entirety. 
* Current implementation can't distinguish between the cars when one eclipses the other. 

I strongly believe a DL approach would be better for this project. Or at the very least, more relevant dataset to help generalize the classification process.
