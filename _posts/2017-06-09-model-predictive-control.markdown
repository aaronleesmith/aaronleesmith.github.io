---
layout: post
title: Model Predictive Control
date: 2017-06-09
image: /assets/images/2016/11/Nanodegree.png
---

There are many ways to drive a car -- and I don't mean with your feet. This post explores a new method of following and maintaining a trajectory: model predictive control.

I've explored navigating the Udacity Simulator using a Convolutional Neural Network (CNN), an Unscented Kalman Filter (UKF), and a PID control.

Each of these approaches has its benefits and biases. The CNN approach needs a lot of varied training data and it's easy to overfit on a single track, but it learns by *seeing* the road, which feels like magic. My PID control was heavily reliant on finding optimal tuning parameters, which can often be a chore on its own, but the implementation is quite simple.

In this two part post I will explore a new way to control an autonomous vehicle: **model predictive control.** This post gives a high level overview of the algorithm and background knowledge. The next post dives head first into the implementation of MPC in C++.

# Follow along
To follow along, you'll need the following:

* cmake $\geq$ 3.5
* make $\geq$ 4.1
* gcc/g++ $\geq$ 5.4
* uWebSockets $==$ 0.14
* ipopt $\geq$ 3.12.1
* CppAD
* Eigen
* [Udacity Simulator](https://github.com/udacity/self-driving-car-sim/releases)

For more info, see my [project on github](https://github.com/aaronleesmith/CarND-MPC-Project).

# Overview of the MPC approach
Model Predictive Control takes advantage of very useful information detailing the current position of the vehicle in the global (map) space and it's relation to a target trajectory. This information is sent from the simulator in the form of waypoints -- points that designate where the car *should* be. In a real world scenario, sensor data -- such as GPS, lidar, and radar -- and on-board cameras, combined with localization and image recognition algorithms, would inform the MPC. Therefore, the MPC controller is one part of a broad and robust pipeline.

The overall goal is to get the vehicle to plan ahead at every timestep and then take the best course of action to correct its trajectory. Choosing the number of steps and the time between steps is an open question which varies depending on the needs of the vehicle and the circumstances of the road. The variables $N$ and $dt$ are used to specify steps and timestep size respectively.

After planning ahead, the vehicle will only consider the very next (or sometimes average the next few) trajectory points and attempt to correct its orientation and velocity to drive along the safest path. 

The bird's eye view of the algorithm is:

1. Collect waypoints from the map.
2. Fit a polynomial to these waypoints.
3. Calculate the cross track and orientation errors.
4. Optimize a cost function given a set of model constraints (target velocity, max steering angle, change in acceleration, etc).
5. Calculate actuator values for steering angle and acceleration.
6. Send the actuator commands to the vehicles.
7. Repeat.

Before writing any code, it's important to understand the underlying motion model and state which evolves over time.

## Kinematic Models vs. Dynamic models

To represent the vehicle's state and how it changes over time, a set of variables must be chosen to accurately and efficiently represent the state. There are two approaches one can take: **Kinematic models** and **Dynamic models**. In a nutshell, Kinematic models ignore forces like gravity, mass, tire pressure, friction, etc. They are simplifications of Dynamic models, which attempt to accurately model the real world as closely as possible.

Of course, Kinematic models are simpler and easier to understand, but they also leave out a lot of information that might impact the vehicle's state in the real world. This implementation focuses on a **Global Kinematic Model**.

## The vehicle's state
What sorts of state information can / should be tracked about a vehicle? A natural place to start is with the vehicles position. This is represented by $x$ and $y$. It's also helpful to know the vehicle's heading, *orientation* is tracked as well. If the vehicle is moving (good vehicles move) it would be good to know the rate at which its moving, velocity = $v$. Initially, the state vector will be $[x, y, \psi, v]$. With a Kinematic model, simple formulae can be used to model the change in each of these state values over time.

## Actuators
When a car is being controlled, what kinds of *inputs* to the car can a human influence (not including the radio). Fundamentally, a human can accelerate, brake, and steer. If acceleration is considered as a positive input to an actuator and braking as a negative input to the same, then there are 2 inputs to control: acceleration and orientation. These will be represented as $\delta$ and $\alpha$.

## Modeling the state over time
To go from time $t$ to time $t+1$ the state vector must be changed over time. Imagine the car following the trajectory below with orientation $\psi$ and velocity $v$.

![Modeling the state over time. Image copyright Udacity.](/assets/article_images/2017-06-03-model-predictive-control/building-a-model.png)

Using some trigonometry, the following updates to to $x$ and $y$ can be derived:

$$
x = x + v * cos(\psi) * dt \\
y = y + v * sin(\psi) * dt
$$

For orientation, take an additional bit of information into consideration: $L_f$. $L_f$ is a measure of the distance between the front of the vehicle and its center of gravity. This is the turn rate. We need to know this because a vehicle's orientation will change at a different rate depending on how quickly it *can* turn. That might seem obvious, but leaving this off will lead to bad orientation mistakes. Furthermore, at higher speeds the vehicle cannot turn as quickly. Therefore, the change in orientation is modeled as follows $\psi$ as follows:

$$
\psi = \psi + \frac{v}{L_f} * \delta * dt
$$

Last but not least, change in velocity can be calculated. Velocity is the derivative of acceleration, so it's modeled as:

$$
v = v + a * dt
$$

With all of these calculations, the complete state of the vehicle can be changed over time given just a few external inputs.

## Fitting polynomials
For this project, an n-order polynomial must be fit to a set of points and later evaluated for a given value of x. The following C++ code is used for each case, respectively.

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

## Solving the problem
To satisfy step 4 of the steps above, it is necessary to define and solve a linear programming problem. This implementation uses *Ipopt*. In general, to solve this sort of problem the engineer will define a set of constraints and an objective cost function which the solver will attempt to optimize. In this case, a *cost function* is derived which the solver will attempt to minimize. It must do this without breaking any constraint that are defined.

### Setting up the cost function
If the goal is to optimize the cost function (minimize the cost, in fact) a decision must be made on how much things *cost* and what things to consider. If a policeman were driving behind you, waiting for you to make mistakes, what sorts of things would increase the chance of you getting pulled over? Abrupt turns and abrupt accelerations? Speeding? Driving too far from the center of the road?

In this case, the vehicle is it's own policeman -- incurring penalities when our car breaks some rules. In this implementation, the following aspects are considered when calculating the cost function:

- Cross track error
- Orientation error
- Vehicle speed
- Overall steering angle
- Overall acceleration
- Change in steering angle
- Change in acceleration

The cost of each of these is chosen to be the squared difference between the target state and the vehicle's current state. Using a squared error makes the cost function monotonically increasing as we add each segment. For example, $cte - target_cte)^2 could be added to the total cost to cause the solver to attempt to reduce the squared difference to the cross track error.

It is possible (and recommended) to weight these costs. For example, a sudden change in acceleration -- even a jerky one -- is nowhere near as bad as a sudden change of steering angle. Not only does this make for a very unpredictable and dangerous vehicle, but it can throw the car so far off the model may not be able to bring it back. To keep this from happening, a positive weight can be applied to the *change* in steering angle to add additional penalities for this value being too large. This will force the optimizer to be a bit more careful about change in steering than the other state variables, leading to a less jerky driver.

In part 2 of this post, the implementation, I go into detail about the individual cost weights and how the effect the car's ability to drive effectively. 

### Defining constraints
The constraints passed into the linear program solver are limits given to the various state variables the solver is able to modify when minimizing the cost. 

There are a number of options for constraints that can be defined. They can even be interdependent if necessary. 

The most impactful constraints in this case are the steering angle and the acceleration actuators. The car should never make too sharp a turn in either direction, and it shouldn't floor the gas or slam on the brakes. To convey this to the solver, the following constraints are defined:

$$
-25^\circ \leq \delta \leq 25^\circ \\
-0.5 \leq \alpha \leq 0.5
$$

# Implementation
Please see part 2 of this 2 part post (coming soon), where I go through the C++ code to implement and experiment with an MPC in the Udacity simulator.
