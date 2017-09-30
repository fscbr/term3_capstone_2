# Coding Self-Driving Car CARLA

The goal of this project is to code a real self-driving car to drive itself on a test track using [ROS](http://www.ros.org/) and [Autoware](https://github.com/CPFL/Autoware). The project is coded to run on Udacity simulator as well as on Udacity's own self-driving car _CARLA_.

![demo_animation](animation.gif)

### Installation

* Be sure that your workstation is running Ubuntu 16.04 Xenial Xerus or Ubuntu 14.04 Trusty Tahir. [Ubuntu downloads can be found here](https://www.ubuntu.com/download/desktop).
* If using a Virtual Machine to install Ubuntu, use the following configuration as minimum:
  * 2 CPU
  * 2 GB system memory
  * 25 GB of free hard drive space

  The Udacity provided virtual machine has ROS and Dataspeed DBW already installed, so you can skip the next two steps if you are using this.

* Follow these instructions to install ROS
  * [ROS Kinetic](http://wiki.ros.org/kinetic/Installation/Ubuntu) if you have Ubuntu 16.04.
  * [ROS Indigo](http://wiki.ros.org/indigo/Installation/Ubuntu) if you have Ubuntu 14.04.
* [Dataspeed DBW](https://bitbucket.org/DataspeedInc/dbw_mkz_ros)
  * Use this option to install the SDK on a workstation that already has ROS installed: [One Line SDK Install (binary)](https://bitbucket.org/DataspeedInc/dbw_mkz_ros/src/81e63fcc335d7b64139d7482017d6a97b405e250/ROS_SETUP.md?fileviewer=file-view-default)
* Download the [Udacity Simulator](https://github.com/udacity/CarND-Capstone/releases/tag/v1.2).

### Usage

1. Clone the project repository

	```bash
	git clone https://github.com/ucatie/CarND-Capstone
	```

2. Install python dependencies
 
	```bash
	cd CarND-Capstone/CarND-Capstone
	pip install -r requirements.txt
	```
3. Make and run styx

	```bash
	cd ros
	catkin_make
	source devel/setup.sh
	roslaunch launch/styx.launch
	```
4. Run the simulator

5. In order to run the code on CARLA, it will be necessary to use a different classifier: a SVM is used in the simulator environment, while a FCN is used to detect traffic lights in the real world. Therefore, it is necessary to download the trained FCN network (based on VGG) snapshot. Due to the size of the file, it cannot be hosted in Github, so please use the followig link to download: [trained FCN snapshot](https://drive.google.com/open?id=0B-varHdnnrqsMDI2QVM4bEo3VUU)

### Real world testing

1. Download [training bag](https://drive.google.com/file/d/0B2_h37bMVw3iYkdJTlRSUlJIamM/view?usp=sharing) that was recorded on the Udacity self-driving car (a bag demonstraing the correct predictions in autonomous mode can be found [here](https://drive.google.com/open?id=0B2_h37bMVw3iT0ZEdlF4N01QbHc))
2. Unzip the file
```bash
unzip traffic_light_bag_files.zip
```
3. Play the bag file
```bash
rosbag play -l traffic_light_bag_files/loop_with_traffic_light.bag
```
4. Launch your project in site mode
```bash
cd CarND-Capstone/ros
roslaunch launch/site.launch
```
5. Confirm that traffic light detection works on real life images

## ROS Nodes Description

#### Waypoint Updater node (waypoint_updater)

This node publishes next 100 waypoints that are closest to vehicle's current location and are ahead of vehicle.This node also considers obstacles and traffic lights to set the velocity for each waypoint.

This node subscribes to following topics:

1. **/base_waypoints:** Waypoints for whole track are published to this topic. This publish operation is a one time only operation. Waypoint updater node receives these waypoints, stores them for later use and uses these points to extract next 100 points ahead of vehicle.

2. **/traffic_waypoint:** to receive the index of the waypoint in the base_waypoints list, which is closest to the red traffic light so that vehicle can be stopped. Waypoint updater node uses this index to find out distance vehicle from light in case traffic light is red and car needs to be stopped.
3. **/current_pose:** to receive cuurrent position of vehicle.
4. **/current_velocity:** to receive current velocity of the vehicle which is used to estimate time to reach the traffic light.

Waypoint updater node first converts all the waypoints to vehicle coordinate system then picks 100 points that are closest and ahead (x >=0) of vehicle. It also sets the velocity of those waypoints so if a red light is ahead it sets the velocity of those waypoints so that vehicle stops before the signal stop line. **The threshold for traffic light distance is 3**.

This node publishes to following topics:

1. **/final_waypoints:** Selected 100 waypoints are published to this topic.  


#### DBW Node (dbw_node)

This node is responsible for vehicle control (acceleration, steering, brake). 

This node subscribes to following topics:

1. **/final_waypoints:** Next 100 waypoints ahead of vehicle's current position are published to this topic. DBW node uses these points to fit a polynomial and to calculate cross track error based on that polynomial and vehicle's current location.
2. **/current_pose:** to receive current position of vehicle.
3. **/current_velocity:** to receive current velocity of the vehicle.
4. **/twist_cmd:** Target vehicle linear and angular velocities in the form of twist commands are published to this topic.
5. **/vehicle/dbw_enabled:** Indicates if the car is under dbw or driver control.

And publishes to following topics:

1. **/vehicle/steering_cmd:** Steering commands are published to this topic.
2. **/vehicle/throttle_cmd:** Throttle commands are published to this topic.
3. **/vehicle/brake_cmd:** Brake commands are published to this topic.

To calculate vehicle control commands steering value, throttle and brake this node makes use of _Controller_ which in turn use following 4 PID controllers and a low pass filter.

1. **PID Controller for velocity** to drive vehicle with target velocity and it uses this PID controller with following parameters.
 
	```
	VELOCITY_Kp = 2.0
	VELOCITY_Ki = 0.0
	VELOCITY_Kd = 0.0
	```
2. **PID Controller for acceleration** to accelerate vehicle smoothly and it uses this PID controller with following parameters
 
	```
	ACCEL_Kp = 0.4
	ACCEL_Ki = 0.1
	ACCEL_Kd = 0.0
	```
	
3. **PID Controller for steering angle** to reduce error from target steering angle calculated using _Yaw Controller_ and it uses it with following parameters
 
	```
	STEER_Kp = 0.8
	STEER_Ki = 0.1
	STEER_Kd = 0.3
	```
4. **Yaw Controller** to calculate steering angle based on current linear velocity and target linear and angular velocity. The result of this controller is added to error value received from _PID Controller for steering angle_.

5. **Low pass filter for acceleration** is used in conjuction with _PID controller for acceleration_ to calculate acceleration commands. It uses following parameters

	```
	ACCEL_Tau = 0.5
	ACCEL_Ts = 0.02
	```

PID controllers are reset if a DBW status is changed like for example if safety driver takes over.

#### Traffic light detection node (tl_detector)

This node is reponsible for detecting and classifying traffic lights. If the traffic light is identified as red then it finds the closest waypoint to that red light and publishes the index of that waypoint to _/traffic_waypoint_ topic. 

This node subscribes to following topics:

1. **/base_waypoints:** Waypoints for whole track are published to this topic. This publish operation is a one time only operation.

2. **/current_pose:** to receive current position of vehicle.

3. **/image_color:** To receive camera images to identify red lights from. Every time a new image is received, traffic light detection node saves it so that it can process it in next iteration of its processing loop which runs at rate of 5Hz.

3. **/vehicle/traffic_lights:** if ground truth data has been created. This topic provides you with the location of the traffic light in 3D map space and helps you acquire an accurate ground truth data source for the traffic light classifier by sending the current color state of all traffic lights in the simulator. When testing on the vehicle, the color state will not be available. You'll need to rely on the position of the light and the camera image to predict it.

And this node publishes to following topics:

1. **/traffic_waypoint:** to publish the index of the red traffic light waypoint.
2. **/traffic_light:** to publish upcoming traffic light info (TrafficLight)
3. **/traffic_light_image:** to publish cropped images of traffic lights for testing. This topic is just for testing purposes in case someone wants to view the images with tools like rviz.

Parameters are used to define and create ground truth and/or training data. Please check launch file

- Node starts by loading parameters from parameter server to check if ground truth images needs to be created and if node is running with simulator or real world to use classes accordingly.
- Traffic light detection node runs at rate of 5Hz.
- Each traffic light has to be detected `STATE_COUNT_THRESHOLD=3` times before we can start using it otherwise we will keep using the previous predicted light.


- In each iteration of processing loop
	- It extracts all the waypoints where the traffic lights are at
	- Transform each traffic light waypoint to vehicle coordinate system
	- Find the traffic light waypoint that is closest to the vehicle. It finds the closest waypoint by transforming the waypoints to vehicle coordinates.
	- After finding the traffic light waypoint if the distance from that traffic light waypoint is less than a certain threshold then it is considered otherwise discarded.
	- If traffic light is close then traffic light state (red, green, yellow) is identified using the last image received from on vehicle camera and the `TLClassifier` class present in `tl_detector/light_classification/tl_classifier.py`. 
	- This identified state of traffic light has to be detected for a certain threshold `STATE_COUNT_THRESHOLD` for it to be accepted.
	- If the new traffic light state is detectd more than `STATE_COUNT_THRESHOLD` times then we check previous light state also to classify new light state as _green to yellow or red light_
	- If red light (green to yellow or red light) state is detected then its corresponind waypoint is published to `/traffic_waypoint`


#### SVM Training Node for Traffic Light Detection and Classification (tl\_detector\_train)

Classification model used for real world is different than that of simulator.

#####For Simulator

Node to train an SVM for four states (red, green, yellow, unknown). Features like HOG, spatial and histogram are collected and in different color spaces. 
Data is read as defined by parameters in launch file so that different techniques, features and color space selection can be tried just by changing parameters as parameters allow different operations on the SVM training. Run it with following command.

```
rosrun tl_detector tl_detector_train.py
```

If task "best" is executed a trained svc.p file is written, which is used in the tl_classifier. Using HSV color space, all channels, and histogram only features for feature space we got `93% accuracy` on red 1290 green 213 yellow 203 unknown 56 images.

It might require installation of these packages:

```
sudo pip install -U scikit-learn
sudo pip install -U scikit-image
sudo pip install -U Pillow
sudo pip install -U matplotlib
```

##### For Real World
An Fully Convolutional Neural Network is trained for real world images and classification of traffic lights as SVM accuracy was not good on real world images. `light_classification\tl_fcn_classifier.py` contains the code for FCN training.

#### TLClassifier class (tl_classifier.py)

This class is used by _Traffic light detection node_. This class is used to classify traffic lights and uses a trained SVM model to do that. It starts by loading the trained SVC model, trained FCN and flag indidcating whether code is running in simulator or not as classification model is different for simulator (binary images + SVM) and real world (FCN + SVM).

Classification is done in 2 steps.

1. **find_classification():** Given an image as input it returns the area containing the traffic light. This area of the image can then be fed to a classifier to classify.

	As we are using two different models (SVM and FCN) for simulator and real world respectively so process is different for simulator and real world.
	
	- **For Simulator:** 
		- Three binary images are created with each containing pixels that have value that lies in range of red, green and yellow color and rest of the pixels are zerod out. 
		- After that, from these three binary images (red, green, yellow), binary image with maximum on pixels is selected.
		- Then sliding window technique is used to make up multiple windows to scan through region of interest in selected binary image. Windows with maximum on pixels are selected and a mean window of the selected windows is returned as region containing the traffic light.
		
	- **For Real World:** 
		- The process is almost same for real world except instead of finding 3 binary images and then selecting one image, FCN is used to return segmented image. 
		- Then sliding window technique is used to make up multiple windows to scan through region of interest in the segmented image. Windows with maximum on pixels are selected and a mean window of the selected windows is returned as region containing the traffic light.



2. **get_classification():** This method takes the area identified by `find_classification()` method as input and returns the state/class of the traffic light. Here is how it works.
	- Image is normalized first.
	- Then sliding window technique is used with sliding window size of 64 and stride 1.
	- `feature_detection.py` is used to collect features like HOG, spatial features and histogram. Feature detection is done with following parameters.
	
	```
	orient = 9  # HOG orientations
	pix_per_cell = 8 # HOG pixels per cell
	spatial_size = (16, 16) # Spatial binning dimensions
	hist_bins = 16    # Number of histogram bins
	```
	
	- SVM is used to classify all those windows into desired classes (red, green, yellow, unknown)
	- Class with maximum windows votes is selected as the final class.

#### tl\_classifier\_test Node

This node is for validating and testing of trained SVM model. Node to classify images of the data_gt or data_test folder. Uses the trained svc and calls tl_classifier code. Parameters are used to define the data folder and svc model file. To run it, execute following command.

```
roslaunch tl_detector test_classifier.launch
```

### Team Members

| Name                | Email                             | Slack        | TimeZone |
|:-------------------:|:---------------------------------:|:------------:|:--------:|
| Frank Schneider     | frank@schneider-ip.de             | @fsc         | UTC+2 |
| Sebastian Nietardt  | sebastian.nietardt@googlemail.com | @sebastian_n | UTC+2 |
| Sebastiano Di Paola | sebastiano.dipaola@gmail.com      | @abes975  	 | UTC+2 |
| Juan Pedro Hidalgo  | juanpedro.hidalgo@gmail.com       | @jphidalgo   | UTC+2 |
| Ramiz Raja          | informramiz@gmail.com             | @ramiz       | UTC+5 |

### Task break down table

Task break down and management is done at:

[https://waffle.io/ucatie/CarND-Capstone](https://waffle.io/ucatie/CarND-Capstone)

## Contributing

1. Fork it (<https://github.com/ucatie/CarND-Capstone/fork>)
2. Create your feature branch (`git checkout -b feature/fooBar`)
3. Commit your changes (`git commit -am 'Add some fooBar'`)
4. Push to the branch (`git push origin feature/fooBar`)
5. Create a new Pull Request