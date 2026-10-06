Moment of inertia is just "rotational heaviness", and it's the reason a long robot arm feels sluggish and a big drone prop feels lazy.

The plain-language version

Mass tells you how hard something is to get moving in a straight line. Moment of inertia tells you how hard it is to get something *spinning*.

The catch is that moment of inertia doesn't just depend on how much mass there is, it depends enormously on how far that mass sits from the axis it's spinning around. Mass out at the rim counts far more than mass near the centre.

Try it: hold a hammer by the handle and wave it side to side, then hold it by the head and do the same. Identical mass, wildly different effort, because the heavy end moved further from your wrist.

The formula, for one lump of mass

I = m × r²

Mass in kg, distance from the axis in metres, and I comes out in kg·m².

The r² is the whole story. Double the distance and you don't get twice the inertia, you get four times.

Worked example, a robot arm

A 100g gripper (0.1kg) on the end of an arm, 0.2m from the shoulder joint:

I = 0.1 × 0.2² = 0.1 × 0.04 = 0.004 kg·m²

Make the arm twice as long, 0.4m, same gripper:

I = 0.1 × 0.4² = 0.1 × 0.16 = 0.016 kg·m²

Four times harder to swing, for the same 100g on the end. Nothing got heavier. This is why serious robot arms put the heavy motors down at the base and drive the far joints with belts or cables, and why you should mount your battery near the centre of rotation, not out on an extremity.

Newton's second law, but spinning

Straight line: force = mass × acceleration

Rotating: torque = moment of inertia × angular acceleration

Same shape, different words. So if you want something to spin up twice as fast in the same time, you need twice the torque. Or you halve the inertia instead, which is usually cheaper than buying a bigger motor.

Why prop size changes how a drone feels

A drone doesn't steer by tilting a surface, it steers by changing motor speeds, so how fast the props can change RPM *is* how fast the drone responds.

|Small prop (5")|Large prop (10"+)|
|---|---|
|Low inertia, spins up in milliseconds|High inertia, takes noticeably longer|
|Snappy, twitchy, corrects instantly|Smooth, floaty, sluggish to correct|
|Less thrust per motor, needs high RPM|Lots of thrust, efficient, long flight times|
|Racing and freestyle quads|Camera and cargo drones|

A 10" prop has roughly four times the radius-squared of a 5", plus more mass, so the inertia difference is large. Your PID gains have to change to match, which is why a tune that flies beautifully on 5" props oscillates or feels dead on 7". See [[Feedback and PID control]].

Rules of thumb worth keeping

Mass near the axis is nearly free, mass at the tip is expensive. Move batteries and controllers inward.

A hollow ring is much harder to spin than a solid disc of the same mass, because all its mass sits out at the rim. That's exactly why flywheels are ring-shaped, they're trying to maximise inertia on purpose.

Spinning things store energy. A heavy flywheel or a big prop keeps going after you cut power, which is a safety issue with props and a coasting issue with heavy wheels.

Angular momentum is conserved, which is the ice skater pulling their arms in and speeding up. Same idea, and it's why a gyroscope resists being tilted, which is what an IMU exploits to sense rotation.

Why you care when building

Long arms and big props need bigger motors, or gentler control gains, or both.

If your drone feels sloppy after a prop change, the inertia changed and your tune no longer matches.

If your robot arm oscillates at full extension but is fine folded up, that's inertia changing with pose, which is genuinely hard to control with a single fixed set of gains.
