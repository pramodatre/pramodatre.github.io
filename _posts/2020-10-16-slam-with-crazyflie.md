---
layout: post
title:  "Simultaneous Localization and Mapping (SLAM) with Crazyflie"
date:   2020-10-16 06:02:41 -0400
summary: Navigating an unknown world using SLAM which enables UAVs like Crazyflie to perform GPS denied navigation.
use_math: true
---
Perception, Planning, and Control are some of the high-level tasks to be performed by a successful autonomous agent. The act of perceiving the environment state such as obstacles and their locations is a perception task. Perception is a necessary skill for any autonomous agent. In the previous post on Motion Planning, we hand-coded the map of an obstacle course and used it for planning safe paths for the Crazyflie. However, this approach doesn't scale when the environment gets larger. Let's explore the task of environmental perception to build a realistic map of the environment.

# Sensors
Crazyflie has flow-deck which consists of sensors to estimate height (z direction) and lateral displacement (x and y direction). A multi-ranger deck is added to the Crazyflie for receiving range observations in five directions (left, right, front, back, and up). These range observations along with flow-deck observations are utilized in this post for mapping the environment.

## Estimated position and attitude
When UAV is navigating in an space where there is no external support for localization such as GPS or cameras, only way to find the position of the UAV at any time is to estimate its position using onboard sensors and control commands issued to the UAV. Crazyflie does this estimation in it's firmware and we can enjoy the estimated positions for free :) Using the python library of Crazyflie, we can [subscribe to position, attitude, and range observations](https://github.com/pramodatre/crazyflie-multi-ranger-deck-slam/blob/master/mappingAndNavigation/crazy_explorer.py#L331) using a callback mechanism. Position observations consists of x, y, and z coordinate estimated using optical flow (x and y coordinate) and a laser range finder (z coordinate). Attitude estimates consists of roll, pitch, and yaw of the UAV. We will use these estimates for creating a map of the environment.

## Range sensor data
As noted earlier, laser range sensors provide distance to obstacles in five directions. Since the battery blocks the laser range sensor facing in the `up` direction, we cannot use this observation (we probably don't need this if we are working with 2D map like we will do in this post). For rest of the range sensor observations, we will have to transform them from body frame to global frame. This transformation requires that we have an estimated attitude of the vehicle, i.e., roll, pitch, and yaw of Crazyflie and estimated position, i.e., x, y, and z coordinate estimate of the Crazyflie.

# Mapping
We need a representation to capture the map of the environment. Representations that are continuous are much more expressive and may also scale for larger areas, e.g., representing obstacles as polygons. However, these methods are much more involved to implement for the first iteration of our map. A discrete representation such as a grid will enable us to represent the environment as smaller chunks called cells which may be either occupied or free of obstacles.

## Grid representation
Models robot environment as fixed sized grids where each grid can be modeled to be a binary random variable. The event of interest we want to model with the random variable for each grid is whether it is occupied or not-occupied.

## Continuous representation
In a cluttered environment, using lines to describe smaller objects and keeping them separate from larger objects may be challenging. Also, dealing with uncertainty of sensing and it's representation of lines/polygons is not intuitive. A grid based representation addresses both these concerns using a probabilistic occupancy representation for each cell in the grid.

## Occupancy grids implementation

### Initialization

### Update

### Numerical stability considerations

### Log-odds vs. raw sensor data
In a grid representation defined above, each cell represents a physical space. This space can be either occupied or unoccupied in the physical world. We need to use sensor data to estimate the true state of the cell space, when using Crazyflie, laser range sensors are used to estimate cell occupancies.

We have a choice of using raw sensor data as-is for determining cell occupancy. Why do we need probabilities? Here is a comparison of raw sensor data vs. log-odds of occupancy probability used to update an occupancy grid.

{% include image.html img="images/2020-10-26/center_obstacle_comparison.png" title="center_obstacle_comparison" caption="Comparing raw sensor data vs. log-odds of occupancy probabilities to update an occupancy grid." %}

## Evaluation

### Obstacle course (controlled environment)
{% include image.html img="images/2020-10-26/autonomous_nav_obstacle_course.gif" title="mapping_simple_2" caption="Crazyflie navigating an obstacle course using multi-ranger deck containing laser range sensors. Environment is mapped using SLAM where localization is done on Crazyflie (firmware) and mapping done on flight computer connected to the Crazyflie." %}

{% include image-small.html img="images/2020-10-26/mapping_simple_2.png" title="mapping_simple_2" caption="Occupancy map created by autonomous navigation of an obstacle course which has less clutter" %}
### Indoor environment with less clutter
{% include image.html img="images/2020-10-26/mapping_larger_area_1.png" title="mapping_larger_area_1" caption="Occupancy map created by autonomous navigation of an indoor space. The green color indicates uncertainty, dark blue indicates un-occupied cells, and yellow indicates occupied cells" %}


### Indoor environment with clutter (realistic)
{% include image-small.html img="images/2020-10-26/autonomous_crash_motivation_for_vision.gif" title="autonomous_crash_motivation_for_vision" caption="While navigating a realistic indoor environment with clutter, Crazyflie crashes due to it's inability to perceive objects in the environment" %}

{% include image-small.html img="images/2020-10-26/autonomous_crash_failing_optical_flow.gif" title="autonomous_crash_failing_optical_flow" caption="While optical flow is a reliable way of estimating x and y displacements in well lit environment, some failures like this may lead to unpredictable behavior and eventual crash!" %}

## Conclusion
