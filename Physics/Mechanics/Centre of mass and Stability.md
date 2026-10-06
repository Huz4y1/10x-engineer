Robots tip over for one reason, and once you can see it you'll design it out without thinking.

Centre of mass in plain terms

The centre of mass (CoM) is the single point where all of an object's weight effectively acts. It's the balance point. Balance a ruler on your finger and your finger is under its centre of mass.

Put heavy things low and central and the CoM sits low and central. Bolt a battery on top and the CoM climbs.

The support polygon

Draw a line on the ground connecting all the points where your robot touches it, wheels, feet, casters. That shape is the support polygon.

```
   two wheels + a rear caster           four wheels

        L ──────── R                    ┌──────────┐
         \        /                     │          │
          \      /                      │    ×     │
           \    /                       │          │
             ○  caster                  └──────────┘
```

The rule is simple. Drop a vertical line down from the centre of mass. If it lands inside the support polygon, the robot stands up. If it crosses outside the edge, it tips.

That's the whole of static stability. A wider stance means a bigger polygon, which means more room for the CoM to move before you're in trouble.

Why robots tip when accelerating

Accelerating, braking or turning effectively shifts where the weight is pushing. Brake hard and the robot pitches forward, turn hard and it leans outward. Get enough of that and the CoM line leaves the polygon.

Worked example

A robot 0.2m wide, so the distance from its centre to the tipping edge is 0.1m, with its centre of mass 0.15m off the ground.

It tips when the sideways acceleration divided by gravity exceeds (half-width / CoM height):

0.1 / 0.15 = 0.67

so it tips at a = 0.67 × 9.81 = 6.5 m/s² of sideways acceleration.

Now move the battery to the floor pan so the CoM is only 0.05m up:

0.1 / 0.05 = 2.0

a = 2.0 × 9.81 = 19.6 m/s², three times more before it goes over.

Nothing was made heavier or wider. The battery just moved down. This is the single cheapest stability fix there is.

The same maths gives you the ramp angle it can sit on: it tips on a slope steeper than the angle whose tangent is half-width over CoM height. For the low version that's about 63°, for the tall version about 34°.

|Change|Effect on stability|
|---|---|
|Move batteries and motors low|Big improvement, free|
|Widen the wheelbase|Big improvement|
|Lengthen the wheelbase|Helps pitch stability under braking|
|Tall mast, camera or arm on top|Hurts a lot, it raises the CoM|
|Adding weight up high|Actively worse than adding no weight at all|
|Accelerate and turn gently|Avoids the problem entirely|

Watch out for arms. A robot arm that's stable folded up can tip the whole machine when it extends, because extending moves the CoM sideways out towards the edge of the polygon. If the arm carries a payload, it's worse.

The self-balancing robot

A two-wheeled balancing robot has a support polygon that isn't a polygon at all, it's a single line between the two wheel contacts. In the forward and backward direction there's essentially zero margin, so the CoM is *always* about to fall outside it.

That's an inverted pendulum: a pendulum balanced upside down. It has no stable resting position. The moment it leans a fraction of a degree, gravity pulls it further that way, and the lean accelerates.

The only way to keep it up is to actively drive the wheels *towards* the direction it's falling, moving the support point back underneath the CoM. Exactly what you do balancing a broom on your palm, you chase the base under the top.

```mermaid
flowchart LR
  A[IMU measures lean angle] --> B[Controller]
  B --> C[Drive wheels toward the lean]
  C --> D[Base moves back under the centre of mass]
  D --> A
```

This has to run hundreds of times a second, and it's the classic PID project because you can see the tuning quality instantly, see [[Feedback and PID control]]. Counter-intuitively, a *taller* balancing robot is easier to control, because it falls more slowly and gives your loop more time to react, same as a long broom being easier to balance than a pencil.

Why you care when building

Battery on the bottom plate, always. It's usually your heaviest single part.

Wide and low beats narrow and tall for anything that moves quickly or turns.

If your robot tips when it stops, brake more gently or lower the CoM, both work.

A drone's CoM should sit at the geometric centre of the four motors, otherwise the flight controller fights a permanent imbalance and burns throttle authority correcting it, see [[Thrust, Lift and how Drones fly]].
