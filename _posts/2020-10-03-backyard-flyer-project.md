---
layout: post
title:  "Porting Backyard Flyer Project to Crazyflie"
date:   2020-10-03 06:02:41 -0400
summary: In this first project of Udacity Flying Car and Autonomous Flight Engineer nanodegree, we will make the drone fly in a predetermined square trajectory.
use_math: true
---
Udacity Flying Car and Autonomous Flight Engineer (FCND) nanodegree starts out with basics of autonomous flight and provides a broad overview of UAVs and their history. The first project called the `Backyard Flyer` is mostly to understand ways to interact with the simulator though even-driven python code.

One of the valuable tool for learning is the [FCND simulator](https://github.com/udacity/FCND-Simulator-Releases/releases) that can be used to control a drone in a virtual 3D world. Simulations are critical for UAVs/robotics as they provide a low or no-risk environment to develop and test out algorithms before deploying them on hardware. Once deployed on hardware, we run the risk of damaging the done or worse risk safety of people around the hardware.

For a complete description of the project and it's solution, you can refer to the code [here](https://github.com/pramodatre/FCND-Backyard-Flyer). Here is the outcome on the simulator.

{% include image.html img="images/2020-10-03/backyard_flyer_solution.gif" title="backyard_flyer_solution" caption="Backyard Flyer project working with FCND simulator" %}

In this post, I will focus on porting this project to work with Crazyflie which was relatively straightforward due to the use of [Udacidrone API](https://udacity.github.io/udacidrone/docs/getting-started.html) for this project using the simulator. Udacidrone API provides a protocol agnostic API to control a drone in the FCND simulator or any supported hardware such as PX4 powered drone or Crazyflie. So, any code written to work with the simulator should also work for Crazyflie. 