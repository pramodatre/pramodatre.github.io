---
layout: post
title:  "Porting Motion Planning Project to Crazyflie"
date:   2020-10-07 06:02:41 -0400
summary: In this project of Udacity Flying Car and Autonomous Flight Engineer nanodegree, we will make the drone fly through an obstacle course.
use_math: true
---

Motion Planning essentially answers the question `How to get from point A to point B?` for the drone. In this project, the drone had to navigate through an urban environment with tall buildings to reach the destination specified as GPS coordinates. However, for Crazyflie, we use local coordinates indoors derived from the flow deck for lateral position (x and y coordinate) and laser range sensor for height (z coordinate).

We assume the map of the environment is given to us -- this is an assumption we make to simplify the problem. In reality, the map of environment may be stale and there may be obstacles (e.g., cars, people, trees) that are not even captured in the map. This simplicity allows us to deal with the problem in a manageable way and learn the basics of motion planning.

# Motion Planning terminology
This description is only to provide an overview of related topics used in this projects and by no means exhaustive or complete description of Motion Planning techniques. You can get a comprehensive overview of this space by referring to [^1] and reading through Motion Planning algorithm implementations such as Python Robotics[^3] and OMPL (Open Motion Planning Library)[^2].

## Representation
We need to represent the surroundings of the UAV in some form that we can store and perform operations to answer any queries related to obstacles and possible paths in the environment.

### Grids
We can split the environment into grids and build an occupancy map using various obstacles in the environment. Grids work well when the UAV is confined to a smaller space such as indoors but doesn't scale well if we need to model a large area such as city. In such a grid, each cell is either occupied or not occupied. We can find paths from point A to point B by traversing the cells that are feasible from the starting point to the destination point. In this project starts with a grid representation and later extends the solution to a graph representation for scalability. 

### Graphs
Graphs are much more compact than grids where each node is a possible waypoint for the UAV. Each node is chosen such that it is indeed feasible for the UAV to be at that location. For example, in this project, an occupancy map of the city is used to sample nodes and edges are checked for collision with obstacles. You can imaging this kind of look up, i.e., looking up if a node is free of any obstacles is quite expensive due to 3D nature of the world. You need to track the lateral span of an obstacle such as a building and it's height and quickly able to answer the question if a node coordinate $(x_{1}, y_{1}, z_{1})$ is away from obstacles (some buffer introduced, e.g., 5 meters). This project was a great way for me to understand the role of [k-d trees](https://www.quora.com/What-is-a-kd-tree-and-what-is-it-used-for), a data structure that is very much suitable for solving such problems.

There may be ample number of waypoints regardless of your representation (grid or graph). We use [Bresenham](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm) algorithm to condense waypoints to minimal points enabling UAV to fly without unnecessarily stopping at extraneous waypoints.

Probabilistic Road Map (PRM) scales well for large spaces compared to approaches such as [Voronoi graphs](https://en.wikipedia.org/wiki/Voronoi_diagram) or grid based approaches. However, with the [FCND simulator](https://github.com/udacity/FCND-Simulator-Releases/releases), I had difficulty using PRM so ended up using Voronoi graphs and refinements of paths to come up with the final paths for the UAV. When we port this code to Crazyflie, we will use PRM instead of Voronoi graphs as it's a more practical and scalable approach.

## Search
There may be multiple alternatives for an UAV to go from point A to point B. Planning is posed as a search problem over Grids or a Graph. A popular approach to search paths is $A^{*}$ search which takes initial position, the goal position, and a heuristic function as inputs. Returns the shortest feasible path which can be used by the drone for its navigation. 

# Simulator to Crazyflie
Following changes has to be made to the project in addition to steps in the
[backyard flyer project](https://pramodatre.github.io/2020/10/03/backyard-flyer-project):
* Implement Probabilistic Road Map (PRM) to find best course of waypoints to reach the goal location from a start location. Once implemented, we initialize all waypoints for the Crazyflie like this
```python
self.all_waypoints = self.plan_path_graph()
```
* Initialize the grid by setting the following variables before constructing the PRM
```python
# convert meters to inches
TARGET_ALTITUDE = 0.3 / 0.0254
SAFETY_DISTANCE = 0.25 / 0.0254
GRID_NORTH_SIZE = 130
GRID_EAST_SIZE = 86
NUMBER_OF_NODES = 200
OUTDEGREE = 4
grid, G = construct_road_map_crazyflie(data, GRID_NORTH_SIZE, GRID_EAST_SIZE, TARGET_ALTITUDE, SAFETY_DISTANCE, NUMBER_OF_NODES, OUTDEGREE)
```
* Initialize start and end location using a measuring tape -- the current scale is inches but you can change this to any scale that works for you (e.g., based on your measuring tape). Note that the coordinates are in (NORTH, EAST, ALTITUDE) since this the format used by Crazyflie.
```python
# set start and goal locations
grid_start = (32, 40, 0)
grid_goal = (110, 40, 0)
```

Here is the complete code for [motion planning with Crazyflie](https://github.com/pramodatre/FCND-projects-crazyflie-port/blob/master/crazyflie_motion_planning.py) and the [utility methods for planning](https://github.com/pramodatre/FCND-projects-crazyflie-port/blob/master/planning_utils.py). Here are examples of paths generated from PRM for an obstacle course you will see in the next section. You can run PRM module separately even if you don't have the Crazylie using the following command
```code
python planning_utils.py 
```
You will see PRMs generate a path first and when you close the visualization, the waypoints from PRM are reduced using Bresenham to generate realistic waypoints that the Crazyflie can follow.

{% include image-small.html img="images/2020-10-07/prm_path.png" title="prm_path" caption="Path from start (32, 40, 0) to goal (110, 40, 0) location using PRM. Coordinates in (north, east, altitude) format." %}

{% include image-small.html img="images/2020-10-07/prm_path_condensed.png" title="prm_path" caption="Path from start (32, 40, 0) to goal (110, 40, 0) location after reducing the waypoints using Bresenham. Coordinates in (north, east, altitude) format." %}

# Crazyflie in action

Once the map is initialized along with start and destination locations in `crazyflie_motion_planning.py`, you can invoke the motion planning and execution script to make the Crazyflie navigate through the obstacle course to reach and land at the desired location.
```code
python crazyflie_motion_planning.py
```

{% include image-small.html img="images/2020-10-07/crazyflie_motion_planning.gif" title="crazyflie_motion_planning" caption="Crazyflie using PRM for creating and executing a plan to reach the set destination from the start location" %}

# Conclusion
Porting Motion Planning project to Crazyflie required additional steps such as creation of a 3D map of the environment populated with approximate location of obstacles, choosing coordinate system units, adjusting the altitude appropriate for indoor flight, and visualizing the waypoints for clarity on Crazyflie behavior. Building such a map manually is not practical for unknown cluttered environments. Even if we have such a map, it's probably going to get stale pretty quickly in such a dynamic environment. Further, there may be non-static obstacles such as people moving around which are never captured in a static map. All these challenges motivates us to pursue approaches to create maps of unknown cluttered environments using various sensors on the UAV. Such a dynamic map creation would enable UAVs to truly explore an unknown environment.

# References
[^1]: Choset, H, Lynch, KM, Hutchinson, S, Kantor, G, Burgard, W, Kavraki, L & Thrun, S 2005, Principles of Robot Motion: Theory, Algorithms, and Implementations. MIT Press.
[^2]: [The Open Motion Planning Library](https://ompl.kavrakilab.org/index.html)
[^3]: [Python sample codes for robotics algorithms](https://github.com/AtsushiSakai/PythonRobotics#path-planning)
