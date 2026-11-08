# Appeal_Default_3D_Mechanics

Standalone default ground mechanics for 3D players and NPCs. This project has its
own local Git repository and no remote. It requires C++17 and Make, with no
external library or dependency on another plugin repository.

## Build and install

Run `make` to build and `make test` for native tests. Windows produces
`Appeal_Default_3D_Mechanics.dll`; Linux produces `Appeal_Default_3D_Mechanics.so`.
Keep the project beneath `Resources/Modules/Appeal_Module_Mechanics/`. On Windows,
run `powershell -NoProfile -ExecutionPolicy Bypass -File Packaging/Windows/Install.ps1`
to build in place; `-AppealResources <path>` copies into another installation.
The current Mechanics module bridge is required and is built separately.

Select the orientation in setup:

```text
Add_Dynamic_Module Mechanics Appeal_Default_3D_Mechanics
```

## Basic functions

```text
Mechanics Select Player
Mechanics Move_Forward
Mechanics Strafe_Right_Analog 50
Mechanics Turn_Right_Analog 25
Mechanics Look_Up_Analog 10
Mechanics Update All
Mechanics Stop
Mechanics Jump
Mechanics Update All
```

X/Z is the ground plane and +Y is up. At yaw 0, forward is +Z and right is +X.
Positive yaw rotates forward toward +X. Pitch looks up/down without changing
ground movement: looking up does not make the entity fly.

All actions have a zero-argument digital export/command and a one-integer analog
version with `_Analog` appended. They act on the selected entity. Digital uses
its configured values at full strength. Analog accepts -100..100 including zero;
movement scales by Value / 100 without overwriting configured Speed. Input does
all mapping and dead-zone filtering; this plugin does neither. Out-of-range
inputs and unknown actions are ignored.

| Action | Behavior |
| --- | --- |
| Stand, Stop | Stop both ground axes and stand upright |
| Walk | Set walking gait and forward input |
| Run | Set running gait and forward input |
| Duck_Stand | Stop ground movement and duck |
| Duck_Walk | Set duck-walking gait and forward input |
| Move_Forward, Move_Backward | Set forward input relative to yaw; preserve strafe |
| Strafe_Left, Strafe_Right | Set sideways input relative to yaw; preserve forward |
| Move_Left, Move_Right | Aliases for Strafe_Left and Strafe_Right |
| Jump | Launch along +Y if grounded, preserving planar velocity |
| Turn_Left, Turn_Right | Change yaw by Turn_Speed * input fraction * Delta_Time per call |
| Look_Up, Look_Down | Change pitch by Look_Speed * input fraction * Delta_Time per call |

The latest command for an axis wins. Inputs remain active until that axis gets a
new value, zero, or Stand/Stop. Negative inputs reverse the named direction.
Combined forward/strafe movement is capped at the current gait's speed, avoiding
diagonal boosts. Run uses Speed * Run_Multiplier; duck walking uses Speed *
Duck_Multiplier. Direction commands preserve the gait. Turning recomputes movement
relative to the new yaw. Turning/looking are per-call increments, not latched
angular velocities; invoke them each input frame while a button or axis is held.

Analog ducking uses abs(Value / 100) as Duck_Amount. Jump uses that absolute
fraction of Jump_Speed; zero does not jump and airborne jumps are ignored.
Gravity and landing restore the previous ground gait. Stand/Stop stops planar
movement without canceling vertical flight. Yaw wraps and pitch clamps to -89..89
degrees. Look changes published pitch; it does not control a Graphics camera.

`Select Entity` selects an entity, attaching new entities to Default. `Use Entity
Plugin_Name` attaches and selects an entity in a registered slot. `Digital Entity
Action` and `Analog Entity Action Value` address NPCs without changing selection.
`Update Entity` or `Update All` advances movement. `Release Entity` exits its
plugin; re-entering preserves settings. Select the player before processing its
input callbacks, since module selection is shared.

## Advanced functions

Use `Mechanics Set Entity Key Value` to tune attached entities. Positions are
game units, speed is units/second, gravity is units/second squared, and angles
are degrees. Defaults need no Advanced setup.

| Setting | Default | Accepted values |
| --- | --- | --- |
| Speed | 4 | Nonnegative finite number |
| Run_Multiplier | 2 | Nonnegative finite number |
| Duck_Multiplier | 0.5 | Nonnegative finite number |
| Jump_Speed | 6 | Nonnegative finite number |
| Gravity | 9.8 | Nonnegative finite number |
| Delta_Time | 1/60 | Positive finite seconds per Update/input frame |
| Ground_Height | 0 | Finite Y height; applied on the next grounded Update |
| Position_X, Position_Y, Position_Z | 0 | Finite number |
| Grounded | 1 | 0 or 1; 1 snaps to ground and zeros vertical velocity |
| Mode | Ground | String label, separate from action |
| Yaw | 0 | Finite degrees, wrapped by remainder into -180..180 |
| Pitch | 0 | Finite degrees, clamped to -89..89 |
| Turn_Speed | 180 | Nonnegative finite degrees/second |
| Look_Speed | 90 | Nonnegative finite degrees/second |

Invalid values are ignored. Changes refresh effective planar movement.
Standard C hooks: Constructor, Destructor, Debug(bool), Set_Paths(string),
Enter(Entity), Exit(Entity), Update(Entity), Digital(Entity, Action),
Analog(Entity, Action, int), Set(Entity, Key, Value). Optional Select(Entity) and
Get(Entity, Key) support native action calls and shared state publication.

## Globals

The bridge publishes `Mechanics.Selected_Entity` (string, initially empty),
`Mechanics.<Entity>.Plugin` (string slot), and every setting above as
`Mechanics.<Entity>.<Key>`. Mode is a string, Grounded is bool, and numeric
settings are floats. Additional globals are:

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| Orientation | string | 3D | Fixed coordinate convention |
| Action | string | Stand | Ground gait, stationary posture, or Jump |
| Velocity_X, Velocity_Y, Velocity_Z | float | 0 | Effective movement per axis |
| Duck_Amount | float | 0 | Ducking blend, 0..1 |
| Input | int | 0 | Last accepted action value |

Globals refresh on Select, Use, actions, Set, and Update. Release clears Plugin,
Mode, Action, Orientation, and selection when appropriate; numeric values remain
inactive snapshots. Other orientations' unsupported globals are empty. Direct C
calls publish on the next module refresh.

## Scope and validation

Movement is kinematic with a configurable flat ground height. It does not detect
collisions, move World entities, drive Physics bodies, resize collision shapes,
or switch Graphics animations/cameras. Collision code can supply Ground_Height
and Grounded. Swimming, ladders, vehicles, and state transfer between plugins
remain future integration work. Mode chooses the eventual animation set; Action
chooses its clip.

`make test` covers action exports, analog bounds, diagonals, yaw-relative movement,
pitch limits, jumping/landing on Y, and independent entities. The checkout's
`Tests/Default_Mechanics_Tests.ps1` loads all three orientation DLLs together and
checks module routing and globals through Appeal.
