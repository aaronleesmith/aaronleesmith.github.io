---
layout: post
title: Model Predictive Control
date: 2017-06-03
image: /assets/images/2016/11/Nanodegree.png
---

There's many ways to drive a car, and I don't mean with your feet. I've explored navigating the Udacity Simulator using a Convolutional Neural Network, an Unscented Kalman Filter, and a PID control.

Each of these approaches has its benefits and biases. The CNN approach needs a lot of varied training data, and it's easy to overfit on a single track, but learns by *seeing* the road, which feels like magic. My PID control was heavily reliant on finding optinal tuning parameters, which can often be a chore on it's own, but the implementation is quite simple.

In this two part post I will explore a new way to drive, one used as part of a larger pipeline in current autonmous vehicles: **model predictive control.** This post gives a high level overview of the algorithm and background knowledge. The next post dives head first into the implementation in C++.

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
Model Predictive Control takes advantage of very useful information about the current position of the vehicle in the global (map) space. This information is sent from the simluator in the form of waypoints -- points that designate where the car *should* be. In a real world scenario, this information can be gained by using sensors to understand the car's global position (GPS) and local position (using localization against some known map data). The MPC controller would be one part of a broad and robust pipeline.

The overall goal is to get the car to plan ahead at every timestep and then take the best course of action to correct its trajectory. Choosing the number of steps and the time between steps to do so is an open question which will vary depending on the needs of the vehicle and the circumstances of the road. The variables $N$ and $\Delta t$ are used to specify teps and time-step size respectively.

After planning ahead, the car will only consider the very next (or sometimes average the next few) trajectory points and attempt to correct it's orientation and velocity to drive along the safest path. 

The bird's eye view of the algorithm is:

1. Collect waypoints from the map.
2. Fit a polynomial to these waypoints.
3. Calculate the cross track error and orientation error.
4. Optimize a cost function given a set of model contraints (target velocity, max steering angle, change in acceleration, etc).
5. Generate the vehicle's change in steering and acceleration by updating the model state according to the results of the optimized function.
6. Repeat.

# Getting off the ground

Before we can write any code, we have to understand the underlying motion model and state we want to evolve over time.

## Kinematic Models vs. Dynamic models

When we want to represent a vehicle's state and how it changes over time, we have to choose what variables to model as the vehicle moves through the world. There are two approaches one can take: **Kinematic models** and **Dynamic models**. In a nutshell, Kinematic models ignore forces like gravity, mass, tire preasure, friction, etc. They are simplifications of Dynamic models, which attempt to accuratly model the real world as closely as possible.

Of course, Kinematic models are simpler and easier to understand, but they also leave out a lot of information that might impact the vehicle's state in the real world. For this MPC implementation we will be focusing on a **Global Kinematic Model**.

## The vehicle's state
What sorts of state information can / should we track about a vehicle? A natural place to start is with the vehicles position. This is represented by $x$ and $y$. We also want to know which way the vehicle is facing, so we also consider the *orientation*. If the vehicle is moving (good vehicles move) it would be good to know the rate at which its moving, so we track velocity with $v$. To start, our state vector will be $[x, y, \psi, v]$. With a Kinematic model, we can apply simple mathematical formulae to change these state values over time.

## Actuators
When a car is being controlled, what kinds of *inputs* to the car can a human influence (not including the radio). Fundmentally, a human can accelerate, brake, and steer. If we consider accelerating as a positive input to an actuator and braking as a negative input to the same, we have 2 inputs to control: acceleration and orientation. These will be represented as $\delta$ and $\alpha$.

## Modeling the state over time
To go from time $t$ to time $t+1$ we have to know how our state vector changes over time. Imagine the car following the trajectory below with orientation $\psi$ and velocity $v$.

![Modeling the state over time. Image copyright Udacity.](/assets/article_images/2017-06-03-model-predictive-control/building-a-model.png)

Using some trigonometry, we can derive the following updates to $x$ and $y$:

$$
x = x + v * cos(\psi) * dt \\
y = y + v * sin(\psi) * dt
$$

For orinetation, we have to take an additional bit of information into consideration: $L_f$. $L_f$ is a measure of the distance between the front of the vehicle and its center of gravity. This is the turn rate. We need to know this because a vehicle's orientation will change at a different rate depending on how quickly it *can* turn. That might seem obvious, but leaving this off will lead to bad orientation mistakes. Furthermore, at higher speeds the vehicle cannot turn as quickly. Therefore, we model the change in orientation $\psi$ as follows:

$$
\psi = \psi + \frac{v}{L_f} * \delta * dt
$$

Last but not least, we need to model the change in velocity. Velocity is the derivative of acceleration, so we can model the change in velocity as:

$$
v = v + a * dt
$$

We now have all the information we need to update the vehicles state given the turn radius, acceleration, and change in time.

## Fitting polynomials
For this project, we will need to both fit an n-order polynomial to a set of points and evaluate a polynomial for a given value of x. The following C++ code is used for each case, respectively.

{% highlight cpp %}
// Evaluate a polynomial.
double polyeval(Eigen::VectorXd coeffs, double x) {
    double result = 0.0;
    for (int i = 0; i < coeffs.size(); i++) {
        result += coeffs[i] * pow(x, i);
    }
    return result;
}
{% endhighlight %}

{% highlight cpp %}
// Fit a polynomial.
// Adapted from
// https://github.com/JuliaMath/Polynomials.jl/blob/master/src/Polynomials.jl#L676-L716
Eigen::VectorXd polyfit(Eigen::VectorXd xvals, Eigen::VectorXd yvals,
                        int order) {
  assert(xvals.size() == yvals.size());
  assert(order >= 1 && order <= xvals.size() - 1);
  Eigen::MatrixXd A(xvals.size(), order + 1);

  for (int i = 0; i < xvals.size(); i++) {
    A(i, 0) = 1.0;
  }

  for (int j = 0; j < xvals.size(); j++) {
    for (int i = 0; i < order; i++) {
      A(j, i + 1) = A(j, i) * xvals(j);
    }
  }

  auto Q = A.householderQr();
  auto result = Q.solve(yvals);
  return result;
}
{% endhighlight %}