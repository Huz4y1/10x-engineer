Three rules from 1687 that still decide whether your robot starts, stops and turns the way you told it to.

First law, things keep doing what they're doing

An object sits still or keeps moving at a constant speed unless a force acts on it. Nothing changes on its own.

For your robot: it doesn't start moving because you set the PWM to 50%, it starts moving because the wheels push against the ground. And when you cut the motors it doesn't stop, it coasts, because nothing is stopping it except friction. That gap between "I sent the stop command" and "it actually stopped" is why a robot overshoots the line it was supposed to halt on.

Second law, force equals mass times acceleration

F = m × a     (newtons = kilograms × metres per second squared)

Acceleration is how fast the speed changes, in metres per second, per second. This is the one you'll actually calculate with.

Worked example, getting a robot moving

A 2kg robot, and you want it to go from standstill to 1 m/s in 1 second.

a = change in speed / time = 1 / 1 = 1 m/s²

F = m × a = 2 × 1 = 2N

So you need 2 newtons of push at the wheels just to accelerate, plus whatever friction and drivetrain drag costs you, so budget maybe 3N in practice.

Turn that into a motor spec: with 30mm radius wheels (0.03m), the torque needed at the axle is

torque = force × radius = 3 × 0.03 = 0.09 Nm

Split across two driven wheels, that's 0.045 Nm per motor. A typical yellow TT gearmotor produces roughly 0.1 to 0.2 Nm, so you've got plenty of headroom. More on this in [[Torque and Gearing]].

Worked example, stopping

The same 2kg robot at 1 m/s, and the best braking force friction can give you is 8N.

a = F / m = 8 / 2 = 4 m/s² of deceleration

Time to stop = speed / deceleration = 1 / 4 = 0.25 seconds

Distance covered while stopping is roughly half the speed times that time = 0.5 × 1 × 0.25 = 0.125m, about 12cm of overshoot. If your line follower's sensor is 5cm ahead of the wheels, you now know why it keeps sailing past the target.

Third law, every push has an equal push back

Push on something and it pushes back on you just as hard, in the opposite direction.

This is not philosophy, it's the entire reason anything moves:

|Your robot does this|Which means|
|---|---|
|Wheels push backwards on the ground|The ground pushes the robot forwards|
|Drone prop pushes air downwards|The air pushes the drone upwards|
|Arm swings out to the left|The robot body twists to the right|
|Motor accelerates the wheels one way|The chassis feels a twist the other way|

The catch is that the ground can only push back as hard as friction allows. Ask for more force than the grip can supply and the wheel just spins. That limit is covered in [[Friction, Traction and Robot movement]].

Mass, weight and the number 9.81

Mass is how much stuff there is, in kilograms, and it's the same everywhere. Weight is the force gravity pulls it down with, in newtons, and it's mass × 9.81.

A 2kg robot weighs 2 × 9.81 = 19.6N. That 19.6N pressing down is what generates its grip, so heavier robots have more traction but need more force to accelerate. That trade-off never goes away.

Why you care when building

Heavier robot means slower acceleration for the same motor, and longer stopping distance. Every gram you add costs you responsiveness.

If your robot overshoots targets, that's the first law, not a code bug. You either brake earlier or add a controller that anticipates it, see [[Feedback and PID control]].

When a robot arm extends, the base feels a reaction twist. That's why arm bases need to be bolted down or heavily weighted.
