# Strangs

All offsets below are for the Steam version.

## Handler Skips

A handler skip is a single frame where the difference in frame time was too small, so the game skips the handler function for Faith's current state.

The game calcuates the difference in frame time as described by the pseudo-code below:

```cpp
double seconds_per_tick = 1.0 / QueryPerformanceFrequency; // 0x2020738

double current, // 0x2027FA8
       last,    // 0x1F98618
       diff;    // 0x1F723E0
		   
for (;;) {
	/*** 0x404140 ***/
	current = (double)QueryPerformanceCounterQuadPart * seconds_per_tick + 16777216.0;
	diff = current - last;
	Tick(diff);
	last = current;
}
```

The skip in calling the handler caused by a too small difference in frame time can be seen in the pseudo-code below:

```cpp
/*** 0x12B0960 ***/
void __thiscall CallStateHandler(byte *this, float diff, int a3) {
	if (diff >= 0.0003) { // A handler skip will occur because diff is smaller than 0.0003
		... // Call corresponding handler
	}
}
```

## Handler Skip for Strangs

For a strang, a handler skip occurs on the frame Faith is walking between two jumps:

- ```(jump) -> (falling) -> (walking, but handler is skipped) -> (jump)```

Once Faith lands, this function in `TdGame.u->TdMove_Landing` is called:
```js
simulated function SubtractLandingSpeed() {
    local TdMove_Jump JumpMove;
    local TdMove_Falling FallMove;
    local Vector NewVelocity;
    local bool bShouldSubtract;

    FallMove = TdMove_Falling(PawnOwner.Moves[2]);
    bShouldSubtract = (PawnOwner.OldMovementState == 2 && (FallMove.PreviousMove == 11 || FallMove.PreviousMove == 32)) || PawnOwner.OldMovementState == 61 || PawnOwner.OldMovementState == 32;
    if (bShouldSubtract && PawnOwner.GetWeaponType() != 1) {
        JumpMove = TdMove_Jump(PawnOwner.Moves[11]);
        // LandingSpeedReduction is 65u/s
        if ((JumpMove.PreJumpMomentum - LandingSpeedReduction) < VSize2D(PawnOwner.Velocity)) {
            NewVelocity = PawnOwner.Velocity;
            NewVelocity.Z = 0.0;
            NewVelocity = Normal(NewVelocity) * (JumpMove.PreJumpMomentum - LandingSpeedReduction);
            NewVelocity.Z = PawnOwner.Velocity.Z;
            PawnOwner.Velocity = NewVelocity;
        }
    } 
}
```

Essentially, the game will subtract speed upon landing if:
- Faith's last movement state was free fall (2), which will be triggered if her vertical velocity is less than `-400u/s`, AND her previous move before the free fall was a standard jump or a kick.
- OR if Faith's previous movement state was a jumping coil.
- OR if Faith's previous movement state was a kick.
- AND if Faith's `PreJumpSpeed - 65.0u/s < CurrentSpeed`

Then the walking handler (`0x12BEF70`) is supposed to be called, but is skipped. Simply put, the walking handler would have adjusted Faith's speed.

### Walking Handler

If Faith's X, Y, and Z acceleration are less than `0.0001u/s^2`, which signifies a smooth continuous jump, the game will adjust Faith's acceleration by the following:
```c
/*** 0x12B31AA ***/
float accel_const = *(float *)(faith_actor_base + 0x274); // 6144 by default
float mag = vx * vx + vy * vy + vz * vz;
float speed = sqrt(mag);
float gain = (3.0 - (((1.0 / speed) * mag) * (1.0 / speed))) * ((1.0 / speed) * 0.5);
float jerk = velocity * gain;
acceleration += (jerk * accel_const);
```

This change in acceleration causes a speed gain. However, if Faith does a non-smooth continuous jump, then she will not gain enough speed to cancel the landing speed reduction. 

The walking handler will finally cap Faith's speed at `720u/s`.

## Conclusion

A frame where the walking handler is skipped would cause:
- Strang after a wallboost
	- Faith's last movement state was free fall but her previous move before the free fall was a wallrun jump.
	- No landing speed subtracted.
- Strang by jumping on a ledge/platform higher than Faith's initial position
	- Faith's vertical velocity does not go below `-400u/s`, so her last movement state is not free fall.
	- No landing speed subtracted.
- Strang falling from zipline
	- Faith's last movement state was free fall but her previous move before the free fall was a zipline hang.
	- No landing speed subtracted.
- Loss in speed while bunny hopping in a straight line
	- Since the walking handler was skipped, there is no speed gain for the continuous jump.
	- Landing speed still subtracted.
	
Note: Faith's main state, which determines what actions are available (i.e. jumping) and Faith's physics, is still changed even when the movement state is not.
