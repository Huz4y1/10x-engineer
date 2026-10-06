Your motors can only push the robot as hard as the floor lets them, and that limit is friction.

The idea

Friction is the resistance between two surfaces sliding across each other. For a robot it's the thing you're fighting when you push a box along the floor, and the thing you're relying on when a wheel grips.

The maximum grip a wheel can give you is:

friction force = µ × normal force

µ (mu) is the coefficient of friction, a number that depends on what's touching what. Normal force is how hard the surfaces are pressed together, which for a robot on flat ground is just its weight in newtons, mass × 9.81.

The important and slightly surprising part: grip does not depend on how big the contact patch is. Wide tyres help for other reasons (heat, wear, deformation), but the simple physics says area doesn't come into it.

|Surfaces|Rough µ|
|---|---|
|Rubber on dry concrete|1.0|
|Rubber on wood or lino|0.7 - 0.8|
|Rubber on smooth tile|0.5|
|Plastic wheel on tile|0.3|
|Plastic on polished floor|0.2|
|Rubber on wet tile|0.2 or worse|

Worked example, why a light robot spins its wheels

A 2kg robot with rubber tyres on a wooden floor, µ ≈ 0.8.

Weight pressing down = 2 × 9.81 = 19.6N

Maximum push available = 0.8 × 19.6 = 15.7N

So no matter what motors you fit, the floor will never give you more than about 15.7N of forward push. From [[Torque and Gearing]], one geared motor was producing 13.3N at the wheel rim, and two of them give 26.6N. That's well past the traction limit, so the wheels break loose and spin instead of accelerating the robot.

Now halve the weight to 1kg:

Maximum push = 0.8 × 1 × 9.81 = 7.8N

Half the grip. And on smooth tile with plastic wheels (µ = 0.3) it drops to 2.9N, at which point the robot barely moves at all and just squeals its wheels. Light robots on slippery floors is the standard beginner disappointment.

Also note that only the weight actually sitting on the *driven* wheels counts. A robot with two driven wheels and a free-spinning caster at the back only gets grip from whatever fraction of its weight is over those two wheels, so where you put the battery genuinely changes how well it accelerates.

Static versus kinetic friction

Static friction is grip while the surfaces aren't sliding. Kinetic friction is what's left once they *are* sliding, and it's always lower.

That's why slipping is a cliff edge rather than a slope. You push harder and harder, grip holds, then it lets go and suddenly there's noticeably less friction than a moment ago, so it slides even more easily. Same reason a car's ABS pulses the brakes, it's trying to stay on the good side of that edge.

For your robot: ramp the PWM up over 100-200ms instead of slamming it to full. Gentle acceleration keeps you in static friction and actually gets you moving faster than flooring it.

Differential drive steering

Two independently driven wheels plus a caster or ball wheel for balance. No steering mechanism at all, you steer by making the wheels turn at different speeds.

```
        [ L ]           [ R ]
          │               │
          └──── chassis ──┘
                  ○  caster
```

|Left wheel|Right wheel|Robot does|
|---|---|---|
|Forward, full|Forward, full|Drives straight|
|Forward, full|Forward, half|Curves gently right|
|Forward, full|Stopped|Pivots around the right wheel|
|Forward, full|Reverse, full|Spins on the spot, centre unchanged|
|Reverse|Reverse|Backs up|

Spinning on the spot is the big win of differential drive, no other steering layout can turn in its own footprint. The price is that going perfectly straight is hard: no two motors are identical, so it drifts. You fix that with wheel encoders and a controller that trims the two speeds against each other, see [[Feedback and PID control]].

Rolling resistance is separate

Rolling resistance is the small drag of a wheel deforming as it rolls, and it's why a robot slows down when you cut power. Soft squishy tyres have great grip but high rolling resistance and eat battery. Hard wheels roll efficiently but slip. Pick based on whether your problem is climbing or runtime.

Why you care when building

If the wheels spin, don't add power, add weight over the driven wheels, or use grippier tyres, or accelerate more gently.

If it's slow but doesn't slip, then it's a torque problem instead and you need more gearing.

A robot that turns fine on carpet and slides on tile hasn't got a code bug, µ just changed under it.

Uphill, the effective normal force drops, so grip drops exactly when you need it most.
