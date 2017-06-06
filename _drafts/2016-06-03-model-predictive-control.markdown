---
layout: post
title: Model Predictive Control
date: 2016-06-03
image: /assets/images/2016/11/Nanodegree.png
---

There's many ways to drive a car, and I don't mean with your feet. I've explored navigating the Udacity Simulator using a Convolutional Neural Network, an Unscented Kalman Filter, and a PID control.

Each of these approaches has its benefits and biases. The CNN approach needs a lot of varied training data, and it's easy to overfit on a single track, but learns by *seeing* the road, which feels like magic. My PID control was heavily reliant on finding optinal tuning parameters, which can often be a chore on it's own, but the implementation is quite simple.

In this post I will explore a new way to drive, one used as part of a larger pipeline in current autonmous vehicles: **model predictive control.**

# Follow along
If you would like to try this out yourself, you'll need the following resources:

* cmake >= 3.5
* make >= 4.1
* gcc/g++ >= 5.4
* uWebSockets == 0.14
* ipopt >= 3.12.1
* CppAD
* Eigen
* [Udacity Simulator](https://github.com/udacity/self-driving-car-sim/releases)

For more info, see my [project on github](https://github.com/aaronleesmith/CarND-MPC-Project).

# Overview of the MPC approach
MPC will take advantage of very useful information about the current position of the vehicle in the global (map) space. This information is sent in the form of waypoints -- points that designate where the car *should* be. This information can be gained by using sensors to understand the car's global position (GPS) and local position (using localization against some known map data). In this case, the waypoints are provided by the simulator.

What we want the car to do is plan ahead. Choosing the number of steps and the time between steps to do so is an open question which will vary depending on the needs of the vehicle. I use the variables *N* and *dt* to specify steps and time-step-size respectively.

After planning ahead, the car will only consider the very next (or sometimes the next few) trajectory points and attempt to correct it's orientation and velocity to drive there. 

The bird's eye view of the algorithm is:

1. Collect waypoints from the map.
2. Fit a polynomial to these waypoints.
3. Calculate the cross track error and orientation error.
4. Optimize a cost function given a set of model contraints (target velocity, max steering angle, change in acceleration, etc).
5. Generate the vehicle's change in steering and acceleration by updating the model state according to the results of the optimized function.
6. Repeat.

