## Project: Search and Sample Return

---


**The goals / steps of this project are the following:**  

**Training / Calibration**  

* Download the simulator and take data in "Training Mode"
* Test out the functions in the Jupyter Notebook provided
* Add functions to detect obstacles and samples of interest (golden rocks)
* Fill in the `process_image()` function with the appropriate image processing steps (perspective transform, color threshold etc.) to get from raw images to a map.  The `output_image` you create in this step should demonstrate that your mapping pipeline works.
* Use `moviepy` to process the images in your saved dataset with the `process_image()` function.  Include the video you produce as part of your submission.

**Autonomous Navigation / Mapping**

* Fill in the `perception_step()` function within the `perception.py` script with the appropriate image processing functions to create a map and update `Rover()` data (similar to what you did with `process_image()` in the notebook). 
* Fill in the `decision_step()` function within the `decision.py` script with conditional statements that take into consideration the outputs of the `perception_step()` in deciding how to issue throttle, brake and steering commands. 
* Iterate on your perception and decision function until your rover does a reasonable (need to define metric) job of navigating and mapping.  

[//]: # (Image References)

[image1]: ./misc/nb_img1.JPG
[image2]: ./misc/nb_img2.JPG
[image3]: ./misc/nb_img3.JPG
[grid]: ./calibration_images/example_grid1.jpg
[rock]: ./calibration_images/example_rock1.jpg 

## [Rubric](https://review.udacity.com/#!/rubrics/916/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.

This part is basically a summary of the important functions needed to complete the project. The functions are ready to use but a few modifications are required to achieve the project requirements.

The first modification I made is in the **Perspective Transformation**.

    mask = cv2.warpPerspective(np.ones_like(img[:,:,0]), M, (img.shape[1], img.shape[0]))

This line of code creates a mask from the warped image which basically helps cancel out the area that isn't visible on the camera's field of view, shown on the image below.

![Figure 1][image1]

The next modification I did was create a function that finds rocks on an image. 

    def find_rocks(img, rock_thresh=(110, 110, 50)):
        color_select = np.zeros_like(img[:,:,0])
        rock_pix = ((img[:,:,0] > rock_thresh[0]) \
                    & (img[:,:,1] > rock_thresh[1]) \
                    & (img[:,:,2] < rock_thresh[2]))
        color_select[rock_pix] = 1
        return color_select

The function is basically a clone of the **color_thresh** function but with a different color threshold which is in the shade of dark yellow/gold. The result is shown on the image below.

![Figure 2][image2]

#### 2. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap.  Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result. 

This is the most challenging part of the notebook because this is where all the necessary functions come together to create the frames for the 3-panel video output.

I'm going to go through the important steps to produce an outcome.

    warped, mask = perspect_transform(img, source, destination)
    threshed = color_thresh(warped)
    obs_map = np.absolute(np.float32(threshed) - 1) * mask

First, I simply called the **perspect_transform** function and stored the returned values in *warped* and *mask* variables
(The *source* and *destination* variables I used were already defined previously in the **Perspective Transformation** section). 
Then, I identified the navigable terrain in the *warped* image using the **color_thresh** function then combined the opposite of the output with the *mask* to create a map of the obstacles called *obs_map*

    xpix, ypix = rover_coords(threshed)
    
The *threshed* image is then used to get the rover-centric coordinates
    
    world_size = data.worldmap.shape[0]
    scale = 2 * dst_size
    xpos = data.xpos[data.count]
    ypos = data.ypos[data.count]
    yaw = data.yaw[data.count]
    x_world, y_world = pix_to_world(xpix, ypix, xpos, ypos, yaw, world_size, scale)
    obsxpix, obsypix = rover_coords(obs_map)
    obs_x_world, obs_y_world = pix_to_world(obsxpix, obsypix, xpos, ypos, yaw, world_size, scale)
    
Which is then converted to world coordinates to determine on which part of the map the rover is.
The coordinates of the obstacles were also determined.

The world map is then updated.

    data.worldmap[y_world, x_world, 2] = 255
    
The position of the rover on the worldmap is colored *blue*
    
    data.worldmap[obs_y_world, obs_x_world, 0] = 255
    
The obstacles are red.

The last part of the function is already given which is just creating the 3-panel frame output for the video.

The output video is saved in the *output* folder as *mapping.mp4*

### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.

The **perception_step** function is identical to the **process_image** function from the notebook with few modifications.
For one, we are now using the *Rover* object in which contains attributes that we need to modify in order to produce the right output, whereas in the notebook, the object of the *Databucket* class was used.

    Rover.vision_image[:,:,2] = threshed * 255
    Rover.vision_image[:,:,0] = obs_map * 255

Here, the *vision_image* attribute is the image that appears on the left side of the simulation, the navigable areas are colored blue, and the obstacles are colored red.

Other attributes that are used are *worldmap*, for modifying the world map and *pos* which contains the position(x and y) of the rover.

    rock = find_rocks(warped)
    
Added to the perception step is the *find_rocks* function to identify rocks in the frame, if any.

    if rock.any():
        rock_x, rock_y = rover_coords(rock)
        rock_x_world, rock_y_world = pix_to_world(rock_x, rock_y, Rover.pos[0], Rover.pos[1],
                                                 Rover.yaw, world_size, scale)
        rock_dist, rock_ang = to_polar_coords(rock_x, rock_y)
        rock_idx = np.argmin(rock_dist)
        rock_xcen = rock_x_world[rock_idx]
        rock_ycen = rock_y_world[rock_idx]

        Rover.worldmap[rock_ycen, rock_xcen, 1] = 255
        Rover.vision_image[:,:,1] = rock * 255
    else:
        Rover.vision_image[:,:,1] = 0
        
Finally, this *if-else* statement is used to take action depending on if any rocks are found on the frame.
If rocks are indeed found, same procedures are done; get the rover-centric coordinates of the rock, then the world coordinates. The coordinates of the rock are then mapped on the world map as green, but would appear yellow because it would also be mapped as a navigable area, which has the color blue set.


For the **decision_step**, I adjusted the threshold on which the rover should stop, so it should not be colliding obstacles as much.

#### 2. Launching in autonomous mode your rover can navigate and map autonomously.  Explain your results and how you might improve them in your writeup.  

**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines!  Make a note of your simulator settings (resolution and graphics quality set on launch) and frames per second (FPS output to terminal by `drive_rover.py`) in your writeup when you submit the project so your reviewer can reproduce your results.**

*Simulator settings:*
* Resolution: **1280x720**
* Graphics quality: **Good**
* **Windowed**
* FPS output is **20-21**

I have successfully achieved the minimum of **40%** environment mapped with **60%** fidelity. The rover can also map all rocks found, not just one. A screen capture is shown below.

![Figure 3][image3]

One noticeable problem that I encountered in autonomous mode is the rover tend to go in circles on a certain part of the map. I think this is because the navigable area in the rover's vision, becomes almost the same anywhere the rover faces. I would probably solve this by detecting whether the rover has gone on a circle and then suggest a direction on where it could go, taking into consideration whether it has been visited or not.

There are still a lot to work on this particular project, including the picking up of rocks, and I'm definitely excited to work on it on my free time.




