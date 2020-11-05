---
layout: post
title:  "Simultaneous Localization and Mapping (SLAM) with Crazyflie"
date:   2020-10-16 06:02:41 -0400
summary: Navigating an unknown world using SLAM which enables UAVs like Crazyflie to perform GPS denied navigation.
use_math: true
---
Perception, planning, and control. The act of perceiving the environment state such as obstacles and their locations is called perception. Perception is a necessary skill that an autonomous agent should posses. In the previous post on Motion Planning, we hand-coded map of an obstacle course and used it for planning safe paths for the Crazyflie. However, this approach doesn't scale when the environment gets larger. Let's explore perceiving the environment to build a realistic map of the environment.

# Sensors

## Estimated position and pose

## Range sensor data

# Mapping

## Grid representation
Models robot environment as fixed sized grids where each grid can be modeled to be a binary random variable. The event of interest we want to model with the random variable for each grid is whether it is occupied or not-occupied.

## Continuous representation
Approximates lines for points received from range sensor readings.

In a cluttered environment, using lines to describe smaller objects and keeping them separate from larger objects seems challenging. Also, dealing with uncertainty of sensing and it's representation of lines/polygons is not intuitive. A grid based representation addresses both these concerns using a probabilistic occupancy representation for each cell in the grid. 

## Occupancy grids

### Initialization

### Numerical stability considerations

### Log-odds vs. raw sensor data
In a grid representation defined above, each cell represents a physical space. This space can be either occupied or unoccupied in the physical world. We need to use sensor data to estimate the true state of the cell space, when using Crazyflie, laser range sensors are used to estimate cell occupancies.

We have a choice of using raw sensor data as-is for determining cell occupancy. Why do we need probabilities? Here is a comparison of raw sensor data vs. log-odds of occupancy probability used to update an occupancy grid.

{% include image.html img="images/2020-10-26/center_obstacle_comparison.png" title="center_obstacle_comparison" caption="Comparing raw sensor data vs. log-odds of occupancy probabilities to update an occupancy grid." %}

## Evaluation

### Obstacle course (controlled environment)
{% include image.html img="images/2020-10-26/autonomous_nav_obstacle_course.gif" title="mapping_simple_2" caption="Crazyflie navigating an obstacle course using multi-ranger deck containing laser range sensors. Environment is mapped using SLAM where localization is done on Crazyflie (firmware) and mapping done on flight computer connected to the Crazyflie." %}

{% include image-small.html img="images/2020-10-26/mapping_simple_2.png" title="mapping_simple_2" caption="Occupancy map created by autonomous navigation of an obstacle course which has less clutter" %}
### Indoor environment without clutter

### Indoor environment with clutter (realistic)

## Conclusion
