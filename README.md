<p align="center"><img src="docs/parkour-crab.png" alt="Faith Runner: a crab with Faith's hair and red arm wraps, mid-parkour" width="440"></p>

# Faith Runner

Mirror's Edge-style first-person movement in Rust, with a Bevy greybox course to test it in.

The movement lives in its own crate, `crates/faith_move`, which only depends on `glam`. It knows nothing about Bevy, so the same controller can later be dropped into IW4L or anything else that can hand it collision boxes.

## Run it

You need Rust (https://rustup.rs). From this folder:

```
cargo run --release
```

The first build takes a few minutes (Bevy is big); after that it's quick.

It opens on a **settings menu**: master volume, music, effects, mouse sensitivity and field of view. Use the arrow keys (or click − / +), then Enter or **PLAY**. **Esc** brings the menu back and pauses. Settings are saved to `settings.txt` in the folder you run from.

## Controls

| Action | Keyboard / mouse | Controller |
|---|---|---|
| Move / look | WASD / mouse | Left stick / right stick |
| Jump, vault, wallrun, wallclimb, pull up | Space | A or LB |
| Crouch, slide, coil in air, roll | Shift, C or Ctrl | B or LT |
| Dodge (sideways jump) | A or D + Space (W can stay held); a stick must be pushed fully sideways | |
| 180° turn (on the ground, in the air moving forward, on a wallclimb, ledge or pole; not mid-slide). On a wallrun: look straight out from the wall | Q | Y |
| Kick / punch / barge a door | Left mouse (or F), as in the game | X |
| Idle animation (standing still) | G cycles Faith's first-person idles | |
| Respawn at checkpoint | R | Select |
| Jump to checkpoint 1–6 | 1–6 | |
| Next map | M | |
| Settings menu (pauses) | Esc | |
| Toggle help | F1 | |
| Mirror's Edge body ↔ procedural arms | F2 | |

## The moves

- **Sprint build-up**: as in the game, running straight follows Mirror's Edge's speed curve (`TdPawn.SpeedCurve_LightWeapon`): 4 m/s at 0.4 s, 5.2 m/s at 1 s, 6.5 m/s at 3.5 s, full sprint (7.2 m/s) at 7 s. Changing direction carries your speed into the new one, but whipping the view round sheds it, and letting go brakes hard. The red bar in the HUD is your sprint energy (speed above 4 m/s), which drains if you strafe or stop pushing forward.
- **Dodge**: jump while holding A or D (holding W as well is fine) to dodge sideways. Rebuilt from Mirror's Edge's `TdMove_DodgeJump`:
  - **Trigger**: it needs a full sideways input (`bMoveActionMax`, over 0.96; a key is always full), and it's checked before the springboard and the plain jump.
  - **Launch**: you keep 30% of your old speed and add 6 m/s sideways (3 m/s up).
  - **Flight**: the dodge then flies on its own, with no steering and no grabs or wall moves, until you're falling faster than 1.9 m/s and the normal fall takes over.
  - **Repeat**: another dodge needs 0.3 s after the last one ends (`RedoMoveTime`). So while running, look 90° right and dodge left (back along your run) to jump straight to full sprint, 7.2 m/s. Turn back forward in the air and the speed carries into your run. It works off walls too: strafe away from the wall during a wallrun and jump, or strafe during a wallclimb and jump for a big sideways kick up the wall.
- **Vault and step up**: jump at an obstacle while holding W. As in the game, the jump itself vaults you: every frame of a jump or fall it checks what's ahead (`TdMove_SpeedVault`). It then picks one of the game's six vault types (`VaultTypes`) from the obstacle's height, whether you'd go over it or onto it, your speed and how fast you're rising. Each type has its own animation, three timed phases (to the hand-plant, over, down) and exit speed:
  - up to 48 cm: a step up (`autostepuprightleg`);
  - 48 cm-1.48 m at a jog (under 2 m/s): a slower step up (`stepuprightleg88`);
  - 64 cm-1.48 m: `VaultOver` if it's thin enough to clear at your speed (its far side within your speed x 0.2, so a 40 cm rail needs about 3.5 m/s), otherwise `VaultOnto`;
  - 1.45-1.92 m, only while you're still rising: `VaultOverHigh` / `VaultOntoHigh`.
  You come out at the speed it took to cover the last leg. Higher than that and you grab the ledge instead.
- **Wallrun**: run at a wall at an angle (anywhere from just off parallel to 57° into it; squarer than that is a wallclimb), jump, and hold forward. Running dead parallel past a wall finds nothing, as in the game; to wallrun a wall you're already right beside, hold a little A or D toward it as you jump. You need to be doing at least 2 m/s and moving into the wall, the wall has to be at least 1.9 m tall, and you can't catch a wall when you're falling and the ground is right underneath. These are the game's own wall checks (`FindWallForward`/`FindWallSide`, native code). It lifts you up to 1.7 m above where you jumped (less if you were already rising for a while). Jump again to kick off it. Looking along the wall keeps your speed along it; looking away pushes you out harder and higher but trades that speed away, down to 10% looking straight out (`TdMove_WallrunJump`). Kick off one wall straight into a wallrun on another and it's a chained run: no lift, and each one in a row pulls you down harder and jumps you off lower.
- **Turn timing**: every 180 turns to the right and follows its animation's own rotation (`UseRootRotation`), so each turn has the game's timing:
  - **Running**: `RunTurn180`, round in about 0.17 s.
  - **Standing**: `StandTurn180Right`, about 0.27 s.
  - **In the air**: `JumpTurnFly`, about 0.3 s.
  - **Wallclimb**: `wallrunvertical180turn`, about 0.6 s.
  - **Swing bar**: `Swing180`.

  (Animation root rotations were read back to front before, which is why the turn went left.)
- **180 in the air**: Q while you're in the air moving forward turns you round (`JumpTurnFly`). Until you land you hold the game's 180-in-air pose, legs out in front as you fall backwards (`jumpturnflyend`; `fallinguncontrolledbwd` on a long drop). Attack during it and Faith taunts (`Taunt`, `TdMove_180TurnInAir`). As in the game, that means you land on your back (`TdMove_Landing.LandBackwards`): a dead stop with the camera going down with you (`JumpTurnLanding`), no roll and no hard landing, and you lie there (`TdMove_LayOnGround`). Jump, or hold W once you're down, to get up (`JumpTurnLandingStand`, 1.3 s), or pull back on S to roll backwards about 3.6 m into a crouch (`EvadeRoll`, `TdMove_LayOnGround.GetUpBack`).
- **Wall to wall**: on a wallrun, Q doesn't turn you round. As in the game (`TdMove_WallRun`'s turn), the view eases round to look straight out from the wall and you keep running. The game's look-at moves `dt / 0.15` of the remaining angle each frame, so it slows as it arrives. Jump then, however far the view has come, and it's the full push-off (`bTurned90FromWall`): out hard and high across to the opposite wall, where you can wallrun again.
- **Wallclimb**: jump straight at a wall while holding forward (past 0.8 on a stick). The game's start check (native code): you're still rising and moving forward, facing the wall within 33°, the wall is at least 1.8 m tall, and it's right in front of your face. If there's a ledge in reach you grab it.
- **Climb, turn, kick**: during a wallclimb press 180° turn, then jump to kick off backwards.
- **Ledge hang**: forward or jump pulls up, crouch drops, A/D shimmies. Turn + jump kicks off backwards. The shimmy works as in the game (`TdMove_Grab`): not for the first 0.6 s after you grab, and only while you're looking within 90° of the wall. Then it goes hand over hand, 60 cm per step with each step taking 1.07 s (the `HangStrafeLeft`/`Right` animation, kept in step with the movement). A step always finishes even if you let go, and holding the key keeps stepping. Where the ledge goes round an outside corner you shimmy round it onto the next face (`HangCornerOutSideLeft`/`Right`); a wall across the ledge stops you. Hanging, your body stays facing the wall and only the view turns. Look more than 90° to either side and a hand comes off the ledge (`HangTurnLeft`/`RightStart`, then the turned idle) and swings round to reach out where you're looking, as the game's first-person arms follow your aim; look back and it returns (`…End`). Jump while turned like that to kick off backwards.
- **Slide**: crouch while running forward at 3.5 m/s or more (speed along your view: sideways momentum doesn't count). Rebuilt from the game's own code (`TdMove_Slide`, its native tick and abort check, the walking physics' slide friction, and the controller's slide input):
  - W and S don't push you (move input is ignored). It brakes with the ground friction x 0.1, so from a full sprint it lasts about 1.3 s and ends below 2.5 m/s, or when you pull back on S.
  - Steer by looking: your body turns toward the view at 0.2 x the angle between them per second, and A/D turn it 11°/s. You can look 55° either side of where you're sliding.
  - Let go of crouch to stand up (not in the first half second). **Jump does nothing mid-slide**: the game's slide only answers to letting go of crouch and to melee (the slide kick).
  - However it ends, it halves your speed and plays `crouchslidetocrouch`. Sliding (and crouching) you're 1.22 m tall (`TdMove.ShrinkCollision`), so the maps' slide-under gaps are 1.3 m.
- **Coil**: press crouch in the air (and hold it) to tuck your legs up 0.6 m and clear things. Only out of a jump (not after running off an edge), and moving forward. Crouch still held from a slide doesn't coil when you jump out of it.
- **Barge and kick doors**: melee (F) at a door. Running at it (over 2.5 m/s) you shoulder through it at your speed plus 2 m/s (up to 5 m/s), `BargeInLeft` then `BargeOutLeft`; standing, you kick it open (`MeleeKickObject`, 0.6 s). From `TdMove_Barge`. Try it on the Moves map's last roof.
- **Barbed wire**: walk into it and you trip: forward over it (`StumbleFwd`, about 2.6 m with no control) or back off it if it's behind you (`GetHitStumbleBwd`). From `TdBarbedWireVolume` and `TdMove_Stumble`.
- **Soft landing**: a drop big enough for a hard landing onto something soft (a mattress) is a soft landing instead (`FallingLandSoftLanding`, 1.5 s), and falling toward one you brace for it (`fallinglandintosoftlanding`). From `TdMove_Landing.LandOnSoftObject` and `TdMove_SoftLanding`. On the Moves map, drop off the right side of the swing roof onto the mattress.
- **Roll**: drops of 5.3 m or more cause a hard landing that stops you dead: as in the game (`TdMove_Landing.LandHard`), you can't move or look until the animation (`FallingLandHard` or `FallingLandHard2`) is done, 1.8 s, and the view levels out. Press crouch in the last 0.2 s before you land (on any fall over 2 m) to roll out of it and keep going. A press only counts 0.6 s after the last one did, so mashing crouch on the way down doesn't work: time it (`TdPawn.CanSkillRoll`).
- **Landing animation**: as in the game's first-person animation setup, a normal landing doesn't play a landing animation over your arms. It dips your spine and eye a little (the `LandNode` offset), and landing on the run mixes a little of `FallingLandMedium` into the run for a moment. Both scale with how far you fell: a normal jump is the minimum, 20% (`TdMove_Landing.GetLandingAmount`). Only hard landings play the full `fallinglandhard`.
- **Jump landings**: a plain jump adds 1 m/s forward, and landing it hands that back plus 0.65 m/s (`TdMove_Landing.SubtractLandingSpeed`), so bunny-hopping slowly costs speed. Dodges and wall jumps don't pay this.
- **Springboard** (`TdMove_SpringBoard`): needs two things in a row, a low step (about 64 cm, ±20) and, with its front about 1.12 m behind the step's, something 0.8–1.48 m high. Run at them and press (or hold) jump any time you'd reach the step within a second, up to about 7 m out at a sprint. Faith runs in, puts a foot on the step, then on the top, and launches up at 9.5 m/s, about 2.7 m higher than a normal jump. A chest-high block on its own isn't a springboard: you just climb onto it.
- **Balance beam**: the game's own balance model (`UTdMove_Balance`'s native tick). You walk at up to 2.45 m/s (`SpeedModifier` 0.34 × `GroundSpeed`). Each frame the lean moves by four influences:
  - **Look**: looking off the beam, 0.3 × the angle over 45°.
  - **Gravity**: 0.3 × sin(lean × 90°); the further you lean, the faster it tips.
  - **Input**: your left/right input, × 1.5.
  - **Speed**: all of that is scaled by 1 + 2.5 × how fast you walk (out of 3 m/s).

  There's no random wobble. Step on at an angle and you start leaning that way (up to 0.2). At full lean you're losing it: Faith windmills (`walkbalancelosebalanceleft`/`right`) and you have 0.8 s to get back under 0.9 or you fall off.
- **Swing pole**: jump at a horizontal bar to catch it. Hold forward to pump up to the game's top swing speed, and jump on the forward swing to fly off at 6 m/s with lighter gravity for 0.7 s (`TdMove_Swing`). Crouch lets go; Q turns around on the bar.
  - **Grip**: the hands stay on the bar. As the game's `SwingControl` does, the whole skeleton pivots about the grip by the swing angle, and the view swings with it.
  - **Drawing**: while swinging the body is drawn in the world's depth (the move's `SDPG_Intermediate`), so the bar hides the fingers wrapped round it.
- **Zipline**: jump up to a cable to grab it and ride it down, speeding up (`TdMove_ZipLine`: at least 3 m/s, +4 m/s² plus gravity down the slope). Jump to leap off, crouch to drop. You let go at the end, or if you hit something.
- **Kicks and punches**: left mouse (or F), as in the game; the same button barges doors. Which attack depends on what you're doing, as in the game: punches on the ground, standing or running (alternating hands; press again within 0.33 s for another, `TdMove_Melee`'s combo window), a jump kick in the air (not in a jump's first 0.1 s), a sliding kick, a wallrun kick off the wall, a punch crouched. Each picks its target, tests the hit and deals the game's damage as Mirror's Edge does (`faith_move::melee`: `GetMeleeTarget`, `TestHit`, the limb sweep of `TdMove_MeleeBase`'s native tick); the maps have no one to hit, so here you see the misses.

## Faith's body and sounds (from your copy of Mirror's Edge)

If Mirror's Edge is installed, the game loads **Faith's own first-person body, textures, animations and sounds** from it at startup. Nothing from Mirror's Edge is included in this project: each time it starts it reads these from your install:

| File (under `TdGame\CookedPC`) | What it provides |
|---|---|
| `Characters\CH_TKY_Crim_Fixer_1P.upk` | The first-person body: arms (`SK_UpperBody`), legs (`SK_LowerBody`), the shared skeleton, and her skin and glove textures |
| `Characters\CH_Faith_Cinematic.upk` | Her trousers texture (`Faith_Cine_Lower_C`) |
| `Characters\CH_TKY_Crim_Fixer.upk` | Her top/shoes material (`MI_Faith_Lowres_Upper`) |
| `Animations\AS_C1P_Unarmed.upk` | All 281 unarmed first-person animations, with their sound cues |
| `Audio\A_Material_Footstep.upk`, `A_Material_Handstep.upk` | Footsteps, landings, slides, hand grabs |
| `Audio\A_Character_Female_01.upk` | Breathing, effort and impact voice, cloth, rolls |
| `Audio\A_Character_Effects.upk`, `A_Ambience_Wind.upk`, `A_M_TimeTrial.upk` | Vault whoosh, rush of wind at speed, rooftop wind, time-trial music |
| `Audio\A_Character_Melee.upk`, `A_Kits.upk` | Kick and punch whooshes, the zipline's ride sound |

The body and animations need the first two rows; everything else is optional and simply skipped if missing.

**Finding the install.** It checks the `FAITH_ME_DIR` environment variable, then a `me_path.txt` file in the folder you run from (put the install path on one line), then common locations: `C:\Games\Mirror's Edge`, Steam's `steamapps\common\mirrors edge`, and the EA/Origin folders. If none has the files, the procedural arms below are used. **F2** switches between the two.

**How it works, like the game.** Faith's body stands in the world, turned with your yaw, and the camera *is* the skeleton's `CameraJoint` bone plus your look pitch. That means:

- Head bob, the wallrun tilt, landing dips and the roll's somersault all come from the animations.
- The hands land where the animation puts them: when you hang, her fingers are on the ledge lip.
- Look down and you see your legs; look up while running and your arms drop out of view, as in Mirror's Edge.
- The arms render on their own layer with depth cleared so they never clip into walls, at the game's first-person FOV (`Model1pFOV = 100`, the world is 90). The legs render in the world. As you look down, the arms' FOV blends to the world's so your torso and legs line up at the waist.
- **Camera rotation**: as in the game (`TdPlayerPawn.CalcCamera`, `GetCameraAnimation`), the camera sits at the eye joint, looks where you look, and adds the eye/camera joint's rotation in the skeleton's space. So body animations turn the view: the landing roll tumbles it a full turn, and hard landings dip it. The animation's yaw is already taken out of the body's placement. The camera channel adds its own clips on top: the jolt when you hit a wall for a wallrun (`wallrunimpactleft`/`right`) and standing up from a crouch (`CrouchIntoStand`).
- **Turning on the spot**: as in the game's `TdAnimNodeTurn`, standing still your legs stay put until the view is 25° off them. Between 25° and 65°, after 0.95 s they step round 45° (`StandTurn45Left`/`Right`); past 65° they step 90° at once (`StandTurn90`).
- **Swan neck** (`TdSwanNeck`): looking down past 15° slides the camera forward and down so you look over your chest at your legs instead of from inside your body, on the game's own curve (about 23 cm forward and 16 cm down near straight down). Hanging on a ledge it starts at once and goes further forward, so you can see the drop; pulling up it reaches a little further than running; a roll turns it off 0.2 s in. It eases in at the game's rate (a fraction `dt/0.07` of the way each frame).
- **Your top**: the legs mesh carries Faith's torso (her black top). It's always drawn, so looking down, standing, running or sliding, you see your top over your legs.
- **Look limits** per move, from each move class's `MinLookConstraint`/`MaxLookConstraint`: hanging you can only look about 18° down, sliding ±55°, on a wallrun you can't turn more than 90° from the wall direction, and so on.
- **Legs**: the legs face where you're running. Strafing turns them up to 90° from your view (`TdPawn.GoBackLegAngleLimit`); past that the backward run plays. Standing still they stay put until you've turned the view 25°, then catch up at the game's `LegRotationSpeed`.

Animations with root motion baked in (vaults, climbs, pull-ups, rolls, the ledge shimmy) are placed so the camera bone follows the gameplay path instead of doubling the movement.

**What plays when.** Each move uses the animation Mirror's Edge's own move class plays, under the game's names:

| Move | Animation |
|---|---|
| Stand / walk / jog / run / sprint | The game's walking states (`TdPawn` Sneak/Walk/Jog/Run/SprintVelocity 5/50/260/400/630 uu/s), each crossfading in over the time the game's animation tree (`AT_C1P`, `TdAnimNodeWalkingState`) gives it: 0.35 s to walk, 0.6 s to run, 0.8 s to sprint. Forward `walkfwd`/`runfwd`/`SprintFwd`, strafing `walkfwdstiff`/`runfwdstiff`, backward `walkbwd`/`runbwd`, sharing one step phase with the tree's sync offsets |
| Jump / fall / coil | `JumpSlow` or `jumpfast`, `jumpair`, `fallinguncontrolled`, `jumpcoil` |
| Landing / hard landing / roll | `jumplandpose` (offset) and `FallingLandMedium` (part-mixed into the run), `fallinglandhard`, `fallinglandroll` |
| Wallrun | `wallrunleftstart`/`rightstart` into `WallrunLeft`/`Right`; jump off `WallrunJumpLeft`/`wallrunjumpright` |
| Wallclimb / turn / kick | `wallrunverticalstart`, `WallRunVertical`, `wallrunvertical180turn` (plays on through the kick until 0.6 s after Q), then `WallrunJumpLeft` (`TdMove_WallClimb180TurnJump`) |
| Ledge | `HangHardStart` into `Hang`, `HangStrafeLeft`/`Right`, `HangCornerOutSideLeft`/`Right`, `HangTurnLeftStart`/`Idle`, `HangTurnJump` (kicking off turned), `HangHeaveUp` |
| Vault / step up | `VaultOver`, `VaultOnto`, `VaultOverHigh`, `VaultOntoHigh`, `autostepuprightleg`, `stepuprightleg88` |
| Slide | `CrouchSlide`, `CrouchSlideEnd` |
| Dodges | `dodgejumpleft`/`right`, `wallrunverticaldodgeleft`/`right` |
| Springboard | `SpringBoardLeftLeg` |
| Balance | `walkbalancestill`/`fwd` mixed with the `…leanleft`/`…leanright` versions by how far you lean (played at speed / 2.2 m/s); `walkbalancelosebalanceleft`/`right` at the edge; `walkbalancefalloffleft`/`right` |
| 180 turns | `RunTurn180`, `StandTurn180Right`, `JumpTurnFly` then `jumpturnflyend`, `Swing180` |
| Idles | G: `standidle1`, `standidle2`, `standidle3`, `edgedetectionidle` (standing still; moving ends them) |
| Forearm twist | The arms' morph targets (`Female1p_UpperBody_MorphSet`): `<Side>ForeArmRollBlend90` / `…90m` blended in by how far each forearm roll bone twists (full at 90°), as `TdPlayerPawn.Init1pArms` sets up. Without them the forearm pinched where the wrist twists (left wallrun) |
| Camera and torso lag | The AnimTree's lazy springs (`TdSkelControlLazySpring`, native tick `0x1223210`). On `EyeJoint`, `CameraRoll` banks the camera into a turn: 0.15 × how far a lagging copy of your yaw trails (catching up `dt / 0.2` a frame), scaled by speed. On `SpineX`, `WeaponYaw`/`Pitch`/`Roll` make the torso and arms trail and bank against your turns and looks, by up to 5.5° |
| Against a wall | Walk up to a wall and the arm(s) in front of it come up, palms flat on it: the game's `againstwall` pose on that arm (in 0.35 s, out 0.55 s) with the hand on the spot its trace hit (`ATdPawn::CheckAgainstWall`, `TdSkelControlAgainstWall`). Walking or crouched only |
| Swing | `swinghardstart`, then the `swingposebacktop` / `swingposebackstraight` / `swingposefronttop` poses mixed by the swing angle (the game's `TdAnimNodeSwing`); `swingjumpoff` |
| Zipline | `ziplinestart`, `ZipLine`, `ziplinehitwall` |
| Attacks | `MeleeStartLeft`/`Right` → `MeleeHitLeft`/`Right` or `MeleeMissedLeft`/`Right` (punch, at 1.5×), `MeleeInAir` / `MeleeInAirStill` / `MeleeFromAbove` (→ `MeleeInAirHit`), `MeleeSlide`, `MeleeWallRunLeft`/`Right`, `MeleeCrouchStart` → `MeleeCrouchHit` |

Pull-up, slide and roll animations are driven by the move's progress, and vaults play their own animation at the game's rate (their phases are timed to it, and the clip's run-out keeps playing over your run after you land), so they finish exactly when the move does, and with the body loaded those moves take as long as the animation (a pull-up takes the game's 1.5 s). Blend times follow the move classes' own (`TdMove_Slide.AnimBlendTime` 0.5, `TdMove_Grab` 0.1, `TdMove_Jump` blend-in 0.1, and so on).

**Sound.** Faith's animations carry the game's own sound cues at the exact frame (`runfwd` has a run step at 0.02 s and 0.37 s; `HangHardStart` has the hand slaps). Those play as the animations hit them: footsteps by type (walk, run, sprint, sprint release, wallrun…), hand grabs, cloth rustle, effort and impact voice, the swing bar's hand slaps and release. Landings, rolls, slides, the rush of wind at speed, rooftop wind and the time-trial music are driven from gameplay. Breathing comes only from the animations' breath cues, as in the game; there's no extra breath timer (the game's native breathing log is never called).

**Surfaces**: footsteps and hand grabs use the game's material sets by what you're touching: concrete roofs and walls, metal for girders and red runner-vision ledges, thin-pipe handsteps for rails, pipes and bars, airduct metal for vents and ducts. Where the game has no variant (airducts have no wallrun steps) it falls back to concrete.

**Looping sounds stop with their move**: the slide scrape loops in the game and the slide move's code stops it; here any looping sound a move starts is stopped when that move ends.

**Volume**: the settings menu's master, music and effects sliders apply straight away, including to sounds already playing.

Kicks and punches whoosh with the game's own swing sounds (`A_Character_Melee.upk`, played from the attack animations), and the zipline hums while you ride (`A_Kits.upk`, `TdMove_ZipLine.ZippingSound`).

**What's still approximate:**

- **Materials:** Faith's diffuse, normal and specular maps are all used (the normal maps with the mesh's own tangent basis, so the bumps light the way they do in the game). The specular map drives a standard material's roughness; her actual skin and cloth shaders (subsurface, fresnel rim) aren't reproduced.
- **Native code:** hand IK on ledges and foot placement on uneven ground are done in native code in Mirror's Edge and aren't reproduced.

Loader code: `crates/me_assets` (UE3 package, mesh, animation, texture and sound readers), `crates/faith_anim` (which animation plays when, and where the body and camera go: engine-independent), `src/me_viewmodel.rs` (skinning and drawing the body), `src/audio.rs` (sound). Its tests run against a real install when `ME_INSTALL` points at one: `$env:ME_INSTALL="C:\Games\Mirror's Edge"; cargo test --workspace`.

## Camera and arms (procedural fallback)

- **Arms**: procedural first-person arms with a pose for every move. They pump when running, keep a palm on the wall while wallrunning, reach while climbing, grip the lip when hanging, plant a hand on vaults, trail a hand on slides and swing into dodges. They lag slightly behind fast looks. They're drawn by a second camera on their own layer, so they never clip into walls and don't stretch with the sprint FOV. Code: `src/viewmodel.rs`.
- **Bob**: footstep-synced head bob (up-down, side to side, a little roll) that scales with speed. You lean into strafes and sharp turns.
- **Shake**: landings, hard landings, rolls, ledge grabs, wall kicks, slides and dodges each add screen shake that fades out, plus a low constant shake at full sprint and while sliding. Landings also dip the camera and kick it down.
- **Tilt and FOV**: the camera tilts away from the wall on wallruns, looks up to the wall top while climbing or hanging, does Mirror's Edge's forward somersault on a roll, and widens the FOV as you sprint.

All of it lives in `crates/faith_move/src/camera.rs` (`CameraFx`) with its numbers in `CameraFxSettings`, so it goes wherever the movement goes. Set `roll_flip: false` if the somersault makes you queasy.

## The maps

**M** switches map (Rooftops → Moves → Springboard → Training). Each has a time trial: the clock starts when you leave the start area and stops at the orange marker.

### Rooftops (default)

A run across a block of buildings, 190 m long, with choices. The street is 30 m down: falling off respawns you at the last checkpoint.

1. **R1 Start**: vault the rail between the AC units, mantle the skylight (or go round it), slide under the pipe (or coil-jump over it), jump the 4 m gap.
2. **R2 Billboards**: vault the low wall or take the open side. Then either wallrun the red billboard on the right across the 8 m gap, or walk the girder on the left.
3. **R3 Tanks**: round the water tank, then wallclimb the 3.3 m wall and pull up, or mantle the crate on the left and grab the roof from it.
4. **R4 Upper**: weave through the AC units, then cross the 12 m girder.
5. **R5 Ledge**: a 6 m drop. Roll it (tap crouch just before landing) or eat a hard landing.
6. **R6 Plaza**: slide the duct, hop the vents, climb the red-lipped wall.
7. **R7 Finish**: the orange marker.

Every route above (including the girder and crate alternatives) is beaten by scripted input in `crates/faith_move/src/rooftops_tests.rs`.

### Moves

A short course for the special moves, one after another (`crates/faith_move/src/moves.rs`):

1. **M1 Springboard**: sprint at the red step and block and jump to springboard up onto the next roof, 3.5 m up. (Without it you'd have to wallclimb.)
2. **M2 Balance**: walk the 8 m beam over the drop. Left/right against the lean.
3. **M3 Swing**: jump to the bar, hold forward, and jump off on the forward swing to reach the lower roof.
4. **M4 Zipline**: jump up to the cable and ride it down 6 m to the last roof.
5. **M5 Finish**: the orange marker. Room to try the kicks.

Every move is beaten (and the balance beam failed on purpose) by scripted input in `crates/faith_move/src/moves_tests.rs`.

### Springboard

A small test range just for the springboard (`crates/faith_move/src/springboard.rs`). Five lanes side by side, all starting on the same line: **1–5** put you at the start of a lane and **R** takes you back to it. Each red lane is a step with a taller block behind it: sprint at it and press jump anywhere on the painted stripe (or hold jump).

1. **Up**: 64 cm step, 1.2 m block, then a deck 3.5 m up.
2. **Lowest**: 50 cm step, 85 cm block, onto a 3 m deck.
3. **Highest**: 80 cm step, 1.45 m block, onto a 3.7 m deck.
4. **Pit**: 64 cm step, 1.2 m block, then across a 6 m pit.
5. **Not springboards** (grey): a thin rail you vault, a 1.2 m block with no step (you climb onto it), and a deep one you mantle onto.

There's no time trial here. Each lane (with jump pressed anywhere from 0.5 to 6 m out), standing still at a step (no springboard), and the near misses are checked by `crates/faith_move/src/springboard_tests.rs`. `FAITH_MAP=2` starts on it.

### Training

The original greybox course, one move per section:

1. **Roof A**: vault the red rail, slide under the pipe, mantle the crate, jump the gap.
2. **Roof B1**: head for the red wall on the right and wallrun over the pit.
3. **Roof B2**: wallclimb the red-lipped wall and pull up.
4. **Roof C**: on the left, climb the tall red wall, turn, and kick back onto the catwalk (bonus). On the right, drop 6 m to Roof D, rolling to keep your speed.
5. **Roof D**: stairs up to the finish marker.

## Tuning

Every number that affects feel is in `crates/faith_move/src/tuning.rs`: speeds, gravity, jump height, wallrun length, ledge reach, and so on.

After changing values, run the tests. They drive the controller through every move and every course section with scripted input, so they'll tell you if a change made something impossible (for example, the wallrun gap no longer clearable):

```
cargo test -p faith_move
```

## Where the numbers come from

Most of the values in `crates/faith_move/src/tuning.rs` are Mirror's Edge's own, read out of your copy of `TdGame.u` with the scripts in `tools/me-extract/`. Each one is marked `ME:` with the class and property it came from (for example `TdMove_WallRun.WallRunningHorisontalInitialZHeight = 170`). The rest are marked `guess`: those live in native code or animation data, not in the scripts.

**From the game's native code.** Some of that native code has been read too, with Ghidra (`tools/me-extract/ghidra/`): ground movement in `crates/faith_move/src/locomotion.rs` is a port of the game's `ATdPawn::GetSprintAcceleration`, `GetWalkAcceleration` and `CalcVelocity` (acceleration from the speed curve, steering, turn damping, sprint energy, friction and braking). Also from native code: air control while falling, the wallrun and wallclimb physics (the wallrun's two gravities and its friction along the wall, the wallclimb's gravity), the wallrun wall checks (`FindWallForward`, `FindWallSide`), and the swan-neck camera's smoothing.

Some highlights:

- Run is 4.0 m/s; sprint builds to 7.2 m/s over 7 s.
- Jump launches at 6.3 m/s and adds 1 m/s forward: a 1.2 m jump under 16.4 m/s² gravity.
- Air control is tiny (1.5 m/s²): you commit to your jumps.
- A wallrun lifts you 1.7 m and ends once you're falling at 5 m/s. Its gravity is its acceleration, 8.2 m/s², half of normal, so it lasts about 1.2 s. Looking away from the wall when you jump off pushes you out much harder.
- A wallclimb boosts you up 1.3 m if you hit the wall still rising, plus up to 0.6 m more for a sprinting run-up (`TdMove_WallClimb.ReachedWall`).
- The 180 kick off a wallclimb launches you 2.5 m up.
- Landings are judged by fall height: 2 m or more can be rolled; 5.3 m or more is a hard landing unless you roll; 10 m is fatal.

**Logic from the scripts.** Beyond the numbers, the moves follow the game's own UnrealScript, decompiled from your `TdGame.u` with `tools/me-extract/decompile.py`: how the wallrun lift and jump-off are worked out, the wallclimb boost, landing speed, the roll timing, the slide's rules. Comments in `controller.rs` name the function each rule comes from. The jumps follow what the scripts assume: every jump in them launches at `2·sqrt(g·h)`, which reaches `h` under Faith's doubled gravity. Where that doubling happens in the native code hasn't been found yet (see Gravity below).

**Gravity** comes from the game's `DefaultGame.ini`. It sets the world to 800 uu/s², but a developer comment beside Faith's pawn settings records that she hits 1600 uu/s after falling 780 cm. That works out to about 1641 uu/s², or 16.4 m/s², so she falls at roughly double the world value. We use the measured figure. It gives a 1.2 m jump with 0.77 s of airtime.

Where the doubling comes from is still unknown. Every path the game's code takes gives 800: `ATdPawn::GetGravityZ` is the world's 800 x `GravityModifier` (1), the falling physics adds it once per step, and no map, config file or move changes it. But the game's own formulas only work at double: the jump's 630 uu/s would rise 2.5 m and clear 13 m gaps at 800, and the wallrun and wallclimb jumps launch at `2·sqrt(g·h)`, which reaches `h` only at `2g`.

## Moving it into another game (IW4L)

`faith_move` only needs two things from a host game:

- A `World` implementation, the same two questions UE3 asks its collision:
  - `sweep(half, start, delta)`: move a box along a path and say where it first touches something (fraction along, surface normal, contact point). Anything it starts inside doesn't count, as with the game's extent traces.
  - `overlaps(region)`: is anything in this box?

  Every move (collide-and-slide, wall and ledge probes, vault and springboard checks) is built on those two, so any collision a host has will do:
  - `BoxWorld`: axis-aligned boxes, exact, with a grid index (the greybox maps).
  - `MeshWorld`: triangles at any angle. The tests run the climb, wallrun, vault, ramp and wall-slide on scenes turned 27° off the grid.
  - Surfaces at a slope up to `WALKABLE` (normal.y ≥ 0.7) are walked on, with the motion projected onto them. Steeper ones are walls you slide along.
- An `Input` each frame, plus reading back `feet`, `vel`, `state`, `events` and `view()` for the camera.

The Skate 3 mode in the mashup already hands control to a separate engine on a keypress. A "Faith mode" would do the same.

## Layout

```
crates/faith_move/src/
  controller.rs    the state machine: every move lives here
  locomotion.rs    running on the ground, ported from the game's native code
  camera.rs        bob, shake, tilt, landing kicks, FOV
  tuning.rs        all the feel numbers
  world.rs         the World trait (sweep / overlap), box and triangle worlds, collide-and-slide, wall and ledge probes
  mesh_tests.rs    the moves on triangle scenes turned off the grid
  greybox.rs       the Training course, as data (and the Level type)
  rooftops.rs      the Rooftops map
  rooftops_tests.rs every Rooftops route, beaten by scripted input
  moves.rs         the Moves map (springboard, beam, swing pole, zipline)
  moves_tests.rs   each special move, beaten by scripted input
  springboard.rs   the Springboard test range
  springboard_tests.rs every lane, beaten (and the near misses refused) by scripted input
  look.rs          per-move look limits and the swan neck
  vault.rs         the game's vault and step-up types (TdMove_SpeedVault)
  tests.rs         one test per move
  course_tests.rs  each course section, beaten by scripted input
src/main.rs        Bevy app: rendering, input, camera, HUD
src/viewmodel.rs   procedural first-person arms (fallback)
src/me_viewmodel.rs loads Faith's body from your install, skins and draws it where faith_anim places it
src/audio.rs       Mirror's Edge sounds: animation cues, surfaces, landings, breathing, wind, music
src/settings.rs    the launch/Esc settings menu (volumes, sensitivity, FOV)
crates/me_assets/  UE3 readers: packages, skeletal meshes, animations, morphs, textures, sounds
crates/faith_anim/ Mirror's Edge's animation driver, body and camera placement (Rig), and which sounds play when
tools/me-extract/  scripts that read movement values out of Mirror's Edge's TdGame.u, and decompile its move code
```

## Screenshot mode

`FAITH_CAPTURE=shots cargo run --release` (PowerShell: `$env:FAITH_CAPTURE="shots"; cargo run --release`) plays a scripted sprint, vault, slide, dodge, wallrun, wallclimb and ledge hang, saves a PNG mid-move for each into `shots/`, and quits. `FAITH_CAPTURE_FROM=5` starts from a later move. Add `FAITH_CAPTURE_TOUR=1` to photograph each Rooftops checkpoint instead, or `FAITH_CAPTURE_MOVES=1` to shoot the springboard, balance beam, swing, zipline and a kick on the Moves map. `FAITH_CAPTURE_SLIDE=1` slides down a Springboard-map lane and shoots what each key does mid-slide (look left/right, Q, F, A, S, jump, let go of crouch, look down); `FAITH_CAPTURE_DELAY=<frames>` sets how long after the key. `FAITH_CAPTURE_LOOK=1` shoots looking down in steps, standing and running. `FAITH_MAP=1` starts the game on the Moves map, `FAITH_MAP=2` on Springboard. Handy for checking the level renders after changes.
