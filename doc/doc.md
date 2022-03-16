There is already a lot of information on the Internet about the panoramic view system of the vehicle, but there is almost no code for reference, which is very unfriendly to newcomers. The purpose of this project is to introduce the principle of the panorama system, and to provide a practical Python implementation with complete basic elements for your reference. The knowledge involved in the surround view panoramic system is not complicated, only the reader needs to understand the camera calibration, perspective transformation, and know how to use OpenCV.

This program was originally developed on an unmanned car equipped with an AGX Xavier, and the running effect is as follows (see `img/smallcar.mp4`).

The car is equipped with four USB surround-view fisheye cameras. The resolution of the images returned by the cameras is 640x480. The images are first corrected for distortion, and then converted into a bird's-eye view of the ground under projective transformation. It's all done on the CPU, and the overall operation is smooth.

Later, I refactored the code and transplanted it to a passenger car (the processor is the same type of AGX), and got similar results (see `img/car.mp4`).

This version uses four 960x640 csi cameras, and the output panorama resolution is 1200x1600. The panorama processing thread runs at about 17 fps without brightness equalization, and drops to only 7 fps after brightness equalization is added. I think that if the resolution is appropriately reduced (for example, using a 480x640 output can reduce the number of pixels to 1/6 of the original), smooth visual effects should also be obtained.

> Note: The black part in the picture is the blind spot after the camera is projected. This is because the front camera is installed on the left side of the front of the car in order to avoid the part of the car logo and the angle is inclined, so the field of view is limited. Imagine a person walking with a crooked neck and slanted eyes...

The implementation of this project is relatively rough, and it is only used as a demonstration project to show the basic elements of generating a look-around panorama, and everyone can understand the spirit. I developed this project to display the vehicle's trajectory in real-time during automatic parking, and also as an internship project for the intern I mentored. Since there is no previous experience and reference, most of the algorithms and processes are written in thought, not necessarily clever, please keep that in mind. The code is written in Python, which is definitely not as efficient as C++, so it is only suitable for learning and verifying ideas.

The following will introduce my implementation steps one by one.


# Hardware and software configuration

I would like to emphasize first that the hardware configuration is the least bothersome thing in this project, the hardware used in the first smallcar project is as follows:

1. Four USB fisheye cameras, the supported resolutions are 640x480|800x600|1920x1080. Because I need to run it in real time under Python, the resolution set for efficiency is 640x480.
2. An AGX Xavier. In fact, it runs as fast as running on a normal notebook.

The hardware used for the second passenger car project is as follows:

1. Four csi cameras, the resolution is set to 960x640.
2. An AGX Xavier, the model is the same as the car project above, but an industrial computer is added to receive csi camera images.


I think as long as you only have four cameras with enough field of view to cover the car's surroundings, plus a normal laptop is enough for all offline development.

The software configuration is as follows:

1. OS Ubuntu 16.04/18.04.
2. Python>=3.
3. OpenCV>=3.
4. PyQt5.

Among them, PyQt5 is mainly used to realize multi-threading, which is convenient for porting to the Qt environment in the future.


# Several conventions adopted by the project


For the sake of convenience, in this project, the four surround view cameras are referred to by `front`, `back`, `left`, and `back`, plus: it is assumed that their corresponding device numbers are integers, such as 0, 1, 2, 3. In actual development, please modify it according to the specific situation.


# Preparation: Obtaining the original image and camera internal parameters


First, we need to obtain the internal parameter matrix and distortion coefficient of each camera. I attached a script `run_calibrate_camera.py` to the project, you just need to run this script, tell it the camera device number, whether it is a fisheye camera, and the grid size of the calibration board through command line parameters, and then hold the calibration board by hand in Just take a few poses in front of the camera.

The following are the original pictures taken by the four cameras in the video, the order is front, back, left, and right, and they are named `front.png`, `back.png`, `left.png`, and `right.png` saved in the project's `images/` directory.

<img style="margin:0px auto;display:block" width=400 src="./img/images/front.png"/>

<img style="margin:0px auto;display:block" width=400 src="./img/images/back.png"/>

<img style="margin:0px auto;display:block" width=400 src="./img/images/left.png"/>

<img style="margin:0px auto;display:block" width=400 src="./img/images/right.png"/>

The internal reference files of the four cameras are `front.yaml`, `back.yaml`, `left.yaml`, and `right.yaml`, and these internal reference files are stored in the `yaml/` subdirectory of the project.

You can see that a calibration cloth is laid on the ground in the picture. The size of this cloth is `6m X 10m`, the size of each black and white square is, and `40cm X 40cm` the square where each circular pattern is located is `80cm X 80cm`. We will use this calibrator to manually select the corresponding points to obtain the projection matrix.


# Set the projection range and parameters

Next, we need to obtain the projection matrix of each camera to the ground. This projection matrix will convert the camera-corrected picture into a bird's-eye view of a rectangular area on the ground. The projection matrices of these four cameras are not independent, and they must ensure that the projected areas can be assembled exactly.

This step is achieved through joint calibration, that is, placing calibration objects on the ground around the car, taking images, manually selecting corresponding points, and then obtaining the projection matrix.

Please see the picture below:

<img style="margin:0px auto;display:block" width=400 src="./img/paramsettings.png"/>

First, place four calibration plates at the four corners of the car body. There is no special requirement for the pattern size of the calibration plates, as long as the dimensions are the same and can be clearly seen in the image. Each calibration plate should be located exactly in the overlapping area of ​​the field of view of two adjacent cameras.

In the camera picture taken above, a calibration cloth is laid around the car. It is not important whether this is a calibration plate or a calibration cloth, as long as the feature points can be clearly seen.

Then we need to set several parameters: (all parameters below are in centimeters)

+ `innerShiftWidth`, `innerShiftHeight`：The distance between the inner edge of the calibration plate and the left and right sides of the vehicle, and the distance between the inner edge of the calibration plate and the front and rear of the vehicle.
+ `shiftWidth`, `shiftHeight`：These two parameters determine how far to look outside the calibration plate in the bird's eye view. The larger these two values ​​are, the larger the scope of the bird's-eye view is, and the more serious the deformation of distant objects after being projected, so it should be selected as appropriate.
+ `totalWidth`, `totalHeight`：These two parameters represent the total width and height of the bird's-eye view. In our project, the cloth width is 6m and the height is 10m, so the range of the ground in the bird's-eye view is `(600 + 2 * shiftWidth, 1000 + 2 * shiftHeight)`。For convenience, we let each pixel correspond to 1 cm, so the total width and height of the bird's eye view is
``` 
    totalWidth = 600 + 2 * shiftWidth
    totalHeight = 1000 + 2 * shiftHeight
```

+ The four corners of the rectangular area where the vehicle is located (the red dots marked in the figure), the coordinates of these four corners are `(xl, yt)`, `(xr, yt)`, `(xl, yb)`, `(xr, yb)` (`l` representing `left`, `r` representing `right`，`t` representing `top`，`b` representing `bottom`)。This rectangular area is invisible to the camera, and we will overlay it with a vehicle icon.

Note that the extended lines on the four sides of this vehicle area divide the entire bird's eye view into front left (FL), front center (F), front right (FR), left (L), right (R), rear left (BL), rear center (B), rear right (BR) eight parts, of which FL (area I), FR (area II), BL (area III), BR (area IV) are the overlapping areas of adjacent camera fields of view, which are also our key needs. The part that performs fusion processing. The four areas F, R, L, and R belong to the separate field of view of each camera, and do not need to be fused.

The above parameters are stored in [param_settings.py](https://github.com/neozhaoliang/surround-view-system-introduction/blob/master/surround_view/param_settings.py).

After setting the parameters, the projection area of ​​each camera is determined. For example, the projection area corresponding to the front camera is as follows:

<img style="margin:0px auto;display:block" width=400 src="./img/mask.png"/>

Next, we need to obtain the projection matrix to the ground by manually selecting the marker points.


# Manual calibration to obtain projection matrix

First run the script [run_get_projection_maps.py](https://github.com/neozhaoliang/surround-view-system-introduction/blob/master/run_get_projection_maps.py) in the project. This script requires you to enter the following parameters:

+ `-camera`: Specifies which camera it is.
+ `-scale`: The horizontal and vertical scaling ratios of the corrected screen.
+ `-shift`: The horizontal and vertical shift distance of the center of the screen after correction.

Why do you need `scale` and `shift`? This is because the default OpenCV correction method is to crop out an area that OpenCV "thinks" is appropriate in the image corrected by the fisheye camera and return it, which will inevitably lose some pixels, especially the ones we want to choose. Feature points are cropped. Fortunately, the function [cv2.fisheye.initUndistortRectifyMap](https://docs.opencv.org/master/db/d58/group__calib3d__fisheye.html#ga0d37b45f780b32f63ed19c21aa9fd333) allows us to pass in a new internal parameter matrix to zoom and pan the image after correction but before cropping. You can try to adjust and select the appropriate horizontal and vertical compression ratios and the position of the image center so that the landmarks on the ground appear in a comfortable position in the picture to facilitate calibration.

Run

```bash
python run_get_projection_maps.py -camera front -scale 0.7 0.8 -shift -150 -100
```

The screen after correction by the front camera is displayed as follows:

<img style="margin:0px auto;display:block" width=600 src="./img/original.png"/>

Then click on the four pre-determined sign points in turn (the order can't be wrong!), and the effect is as follows:

<img style="margin:0px auto;display:block" width=600 src="./img/choose_front.png"/>

Note that the setting code for the marker point is [here](https://github.com/neozhaoliang/surround-view-system-introduction/blob/master/surround_view/param_settings.py#L40)。

These four points can be set freely, but you need to manually modify their pixel coordinates in the bird's eye view in the program. When you click on these four points in the calibration map, OpenCV will calculate a projective matrix based on the correspondence between their pixel coordinates in the calibration map and their pixel coordinates in the bird's-eye view. The principle used here is that four points correspond to determine a projective transformation (four points can give eight equations to solve the eight unknowns of the projective matrix. Note that the last component of the projective matrix is ​​always fixed to 1).

If you accidentally point crooked, you can press `d` to delete the last wrong point. After selecting, click Enter, and the projected renderings will be displayed:

<img style="margin:0px auto;display:block" width=600 src="./img/front_proj.png"/>

If you think the effect is good, press Enter, and the projection matrix will be written to `front.yaml` the name of this matrix is `project_matrix`. If you fail, press `q` Exit try again.

Another example is the calibration of the rear camera as shown in the following figure:

<img style="margin:0px auto;display:block" width=600 src="./img/choose_back.png"/>

The corresponding projection image is

<img style="margin:0px auto;display:block" width=600 src="./img/back_proj.png"/>

Using this operation for each of the four cameras, we get the bird's-eye view of the four cameras, and the corresponding four projection matrices. Our next task is to put together these four bird's-eye views.


# Stitching and smoothing of bird's eye view

If everything is working correctly, running [run_get_weight_matrices.py](https://github.com/neozhaoliang/surround-view-system-introduction/blob/master/run_get_weight_matrices.py) should display the following mosaic:

<img style="margin:0px auto;display:block" width=480 src="./img/result.png"/>

Let me walk you through how it does it step by step:

1. Since there are overlapping areas between adjacent cameras, this part of the fusion is the key. If you directly take the weighted average of the two images (with each weight of 1/2), you will get a result similar to the following:

    <img style="margin:0px auto;display:block" width=480 src="./img/bad.png"/>

    You can see that due to the error of correction and projection, the projection results of adjacent cameras in the overlapping area are not completely consistent, resulting in garbled characters and ghosting in the stitched results. The key here is that the weight coefficient should change with the pixel, and change continuously with the pixel.
    
2. Take the upper left area as an example, this area is the overlapping area of ​​the two camera fields of view `front`, `left`. We first take out the overlapping part of the projection map:：

    <img style="margin:0px auto;display:block" width=250 src="./img/overlap.png"/>

    Grayscale and binarize:
    
    <img style="margin:0px auto;display:block" width=250 src="./img/overlap_gray.png"/>

    Note that there is noise in it, which can be removed with morphological operations (it doesn't have to be very fine, it can be roughly removed):
    
    <img style="margin:0px auto;display:block" width=250 src="./img/mask_dilate.png"/>
    
    So far we have got a complete mask of the overlapping area.

3. Detect the boundaries of `front` and `left` the images that are outside the overlapping area. This step is to first call to `cv2.findContours` find the outermost boundary, and then use `cv2.approxPolyDP` to obtain the approximated polygon outline:

    <img style="margin:0px auto;display:block" width=250 src="./img/polyA.png"/>
    
    <img style="margin:0px auto;display:block" width=250 src="./img/polyB.png"/>
    
    We denote the contour after subtracting the overlapping area from the `front` camera `polyA` (the blue border in the upper left image) and `left` the contour after subtracting the overlapping area from the camera as `polyB` (the green border in the upper right image).

4. For each pixel in the overlapping area, use `cv2.pointPolygonTest` to calculate the `polyA` distance $d_A, d_B$ to the two polygons and then, the corresponding weight of the pixel is $w = d_B^2 / (d_A^2 + d_B^2) * polyB$, i.e.: if this pixel falls within the `front` picture, its `polyB` is farther away and thus has a greater weight.

5. For pixels that are not in the overlapping area, if they belong to the range of the `front` camera , the weight is 1, otherwise the weight is 0. So we get a continuously changing matrix $G$ with values ​​ranging from 0 to 1, and its grayscale image is as follows:
  
    <img style="margin:0px auto;display:block" width=250 src="./img/weight_for_FL.png"/>

    Taking $G$ as the weight, the fused image can be obtained as `front * G + (1- G) * left`。

6. Note that since the pixel values ​​in the overlapping area are the weighted average of the two images, the objects appearing in this area will inevitably appear phantom. Therefore, we need to compress the overlapping area as much as possible, and only focus on stitching as much as possible. The pixels around the seam are used to calculate the weights. The pixels above the seam try to use `front` the , and the pixels below the seam use `back` the original pixels from. This step can be obtained by controlling the value of $d_B$.
  
7. We also missed an important step: due to the different exposures of different cameras, there will be brightness differences between light and dark in different areas, which affects the aesthetics. We need to adjust the brightness of each area so that the brightness of the entire stitched image tends to be consistent. This step is not unique, and there is a lot of room for free play. I looked up the methods mentioned online and found that they were either too complicated to be almost impossible to be real-time, or too simple to achieve the desired effect. Especially in the example of the second video above, because the field of view of the front camera is blocked by the car logo, the photosensitive range is insufficient, resulting in a big difference in the brightness of the screen from the other three cameras, and it is very difficult to adjust.
  
    A basic idea is this: each camera returns `BGR` three channels, and four cameras return a total of 12 channels. We need to calculate 12 coefficients, multiply these 12 coefficients to each of the 12 channels, and then combine them to form the adjusted picture. Too bright channels should be dimmed, so the multiplication factor is less than 1, and too dark channels should be brighter, so the multiplication factor is greater than 1. These coefficients can be obtained from the ratio of the brightness of the four images in the four overlapping areas. You can freely design the way to calculate the coefficients, as long as this basic principle is satisfied.

    See my implementation [here](https://github.com/neozhaoliang/surround-view-system-introduction/blob/master/surround_view/birdview.py#L210). It feels like a piece of shader code.

    There is also a lazy way to calculate a *tone mapping* function in advance (such as segment-by-segment linear, or *AES tone mapping* function), and then force all pixels to be converted. This method is the most labor-saving, but the resulting picture tone will be different from the real scene. It seems that some products on the market take this approach.

8. Finally, due to the different intensities of different channels of the camera in some cases, a color balance needs to be performed, as shown in the following figure: (the original picture after splicing, the brightness balance picture, the brightness balance + color balance picture)

    <img style="margin:0px auto;display:block" width=250 src="./img/example1.png"/>

    <img style="margin:0px auto;display:block" width=250 src="./img/example2.png"/>

    <img style="margin:0px auto;display:block" width=250 src="./img/example3.png"/>

    In the second video example, the color of the picture is reddish, and the picture returns to normal after adding color balance.


# Implementation-specific considerations

1. Multithreading and thread synchronization. In the two examples in this article, the four cameras are not synchronized by hardware triggering, and even if they are synchronized by hardware, the processing threads of the four images may not be synchronized, so a thread synchronization mechanism is required. The implementation of this project is a relatively primitive one, and its core code is as follows:

```python
class MultiBufferManager(object):

    ...

    def sync(self, device_id):
        # only perform sync if enabled for specified device/stream
        self.mutex.lock()
        if device_id in self.sync_devices:
            # increment arrived count
            self.arrived += 1
            # we are the last to arrive: wake all waiting threads
            if self.do_sync and self.arrived == len(self.sync_devices):
                self.wc.wakeAll()
            # still waiting for other streams to arrive: wait
            else:
                self.wc.wait(self.mutex)
            # decrement arrived count
            self.arrived -= 1
        self.mutex.unlock()
```
A `MultiBufferManager` object to manage all threads, each camera thread will call its `sync` method every time it loops, and notify this object by adding 1 to the counter to "report, I have done the last task. , please add me to the dormant pool for the next task". Once the counter reaches 4, it will trigger a task loop that wakes up all threads and enters the next round.

2. Build a lookup table to speed up operations. The image of the fisheye lens needs to be corrected, projected, and flipped before it can be used for splicing. These three steps involve frequent image memory allocation and destruction, which is very time-consuming. Grab threads were consistently stable at a little over 30fps in my tests, but processing threads per frame were only around 20fps. This step is best accelerated by precomputing a lookup table. Do you remember the `cv2.fisheye.initUndistortRectifyMap` function？It returns `mapx, mapy` which are two lookup tables. For example, when you specify that the matrix `cv2.CV_16SC2` type it returned `mapx` is , it returns a pixel-by-pixel lookup table, `mapy` a one-dimensional array for interpolation smoothing (you can throw it away). Similarly, `project_matrix` it is not difficult to obtain a look-up table for , and the two can be combined to obtain a look-up table directly from the original picture to the projected picture (of course, the information used for interpolation is lost). In this project, because Python is implemented, and Python's `for`-loop efficiency is not high, this lookup table method is not used.

3. The four weight matrices can be compressed into one image as `RGBA` four channels, which is convenient to store and read. The same

    <img style="margin:0px auto;display:block" width=250 src="./img/masks.png"/>

    goes for the mask matrix corresponding to the four overlapping regions:

    <img style="margin:0px auto;display:block" width=250 src="./img/weights.png"/>


# Real car setup


You can run [run_live_demo.py](https://github.com/neozhaoliang/surround-view-system-introduction/blob/master/run_live_demo.py) on the real car to verify the final effect.

You need to pay attention to modifying the camera device number and the way OpenCV opens the camera. The usb camera can be opened directly with `cv2.VideoCapture(i)` (`i` it is the usb device number), and the csi camera needs to be called gstreamerto open, the corresponding code is [here](https://github.com/neozhaoliang/surround-view-system-introduction/blob/master/surround_view/utils.py#L5) and [here](https://github.com/neozhaoliang/surround-view-system-introduction/blob/master/surround_view/capture_thread.py#L75)。


# Appendix: List of Project Scripts

The scripts currently in the project are arranged in the order of execution as follows:

1. `run_calibrate_camera.py`: Used for camera internal parameter calibration.
2. `param_settings.py`: Used to set the parameters of the projection area.
3. `run_get_projection_maps.py`: Used to manually calibrate the projection matrix obtained to the ground.
4. `run_get_weight_matrices.py`: Used to calculate the weight matrix and mask matrix corresponding to the four overlapping areas, and display the stitching effect.
6. `run_live_demo.py`: The final version for running on a real car.
