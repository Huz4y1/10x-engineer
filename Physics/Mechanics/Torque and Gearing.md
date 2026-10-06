Torque is twisting force, and gears are how you trade the speed you have for the torque you actually need.

What torque is

Push on a spanner. The further from the bolt your hand is, the easier the bolt turns, even though you're pushing just as hard. Torque is that combination of force and distance from the axis.

torque = force × distance from the centre     (Nm = newtons × metres)

Worked example: you push with 5N on a lever arm 0.1m long.

torque = 5 × 0.1 = 0.5 Nm

Same force at 0.2m gives 1.0 Nm. That's why long spanners exist.

You'll also see kg·cm on cheap servo listings, which is how much weight it can hold at 1cm out. To convert: 1 kg·cm ≈ 0.098 Nm, so roughly divide by 10.

The trade-off nothing gets around

A motor produces a certain amount of mechanical power, and power is torque multiplied by speed. You can shuffle which is which, but you can't create more of both.

A bare hobby motor spins insanely fast with almost no twist. It will happily do 12,000rpm and be stopped dead by your fingertips. Bolt a wheel straight onto it and the robot doesn't move, because it can't produce enough force at the wheel to overcome the robot's own weight and friction. It just sits there whining.

Gears fix that:

|Gearing down (small gear driving big gear)|Gearing up (big driving small)|
|---|---|
|Output is slower|Output is faster|
|Output has more torque|Output has less torque|
|What every robot drivetrain does|Rare, used in things like bicycles going downhill|

Gear ratio is just how many turns of the input make one turn of the output. A 100:1 gearbox means the motor turns 100 times for one turn of the wheel.

output speed = input speed / ratio

output torque = input torque × ratio × efficiency

Efficiency matters. A plastic spur gearbox is maybe 70-80% efficient, a decent metal planetary gearbox 85-90%, a worm drive can be as low as 50% but it can't be back-driven, which is handy for arms that must hold position with the power off.

Worked example, a geared motor

A bare motor: 12,000rpm free speed, 0.005 Nm of torque, and a 100:1 gearbox at 80% efficiency.

output speed = 12,000 / 100 = 120rpm

output torque = 0.005 × 100 × 0.8 = 0.4 Nm

120rpm is a sensible wheel speed and 0.4 Nm is a usable amount of twist. Now turn that into push at the ground with a 30mm radius wheel (0.03m):

force at the wheel rim = torque / radius = 0.4 / 0.03 = 13.3N

From [[Forces and Newton's Laws]], a 2kg robot only needed about 3N to accelerate decently, so one motor alone has four times the force needed. It'll climb a ramp comfortably. The bare ungeared motor, by contrast, gives 0.005 / 0.03 = 0.17N, which wouldn't move a paperback.

And the speed: 120rpm on a 30mm radius wheel means the wheel circumference is 2 × 3.14 × 0.03 = 0.19m, so 120 × 0.19 = 22.6 metres per minute, about 0.38 m/s. Brisk walking pace for a small robot, which is about right.

Wheel size changes the deal too

Bigger wheels are effectively another gear-up. They cover more ground per turn, so more speed, but less force at the rim for the same torque. If your robot struggles up ramps, smaller wheels or a higher gear ratio both help, and both cost you top speed.

Common drive options

|Option|Ratio|Speed|Torque|Good for|
|---|---|---|---|---|
|Bare DC motor|1:1|Silly fast|Useless|Props and fans only|
|TT "yellow" gearmotor|48:1|~200rpm|Decent|First robot, cheap|
|N20 metal gearmotor|50:1 to 300:1|30-500rpm|Good for size|Small tidy builds|
|Planetary gearmotor|100:1+|Low|High|Climbing, heavy robots|
|Standard servo|Internal|~60rpm|High, but limited to ~180°|Arms, steering, gimbals|
|Stepper|Direct|Slow|Good, holds position|Precise positioning, printers|

Why you care when building

If your robot can't climb or start on carpet, you need more gear reduction, not more voltage.

If it's fast but the wheels spin uselessly, you've hit a traction limit instead, see [[Friction, Traction and Robot movement]].

More gear reduction means more current at stall too, and gearboxes multiply torque right up until the plastic gears strip. The teeth are usually the weakest link in a cheap drivetrain.
