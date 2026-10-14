# Fixed-Step Time Control Plan for Mindustry v160.7

## Status

Planning only. No implementation described here has started.

## Objective

Replace the current large-delta JavaScript implementation with a desktop Java
mod that advances single-player simulation through complete fixed 60 Hz ticks.
For equal simulated time, supported accelerated speeds must produce the same
observable conveyor, production, unit, and physics results as normal speed.

## Problem statement

The current implementation multiplies `Time.delta` once per rendered frame.
At high speeds, Mindustry receives one oversized update rather than many normal
updates. Systems that perform bounded discrete work per update can discard work
or skip state transitions.

The correction is architectural: accumulate requested simulation time and call
the original game logic once for every complete 1/60-second step. Rendering and
one-shot input remain once per rendered frame.

## Initial support boundary

- Mindustry desktop build 160.7 only.
- Single-player gameplay only.
- Standard SDL desktop graphics, `DesktopInput`, `Logic`, and `AsyncCore` only.
- Requested range remains `1/256x` through `256x`.
- Actual achieved speed may be lower than requested speed under CPU saturation.
- Multiplayer and the editor always use vanilla timing.
- Android, iOS, headless servers, custom input implementations, modified async
  cores, and other Mindustry versions are out of scope for the first release.

The installer must fail before mutating runtime state when any prerequisite does
not match the supported boundary.

## Design

### 1. Java mod conversion

Create a minimal Java mod build targeting the exact Mindustry 160.7 API. Keep
the existing repository and mod identity so current users can update normally.
Move the existing UI behavior into Java without changing the visible control
model unnecessarily.

The old JavaScript entry point must not coexist with the fixed-step runtime in a
release artifact.

### 2. Fixed-step clock

Maintain a fractional pending-tick accumulator:

```text
pending += min(realFrameSeconds, 0.25) * 60 * requestedSpeed

while pending >= 1 and simulation is still eligible and budget remains:
    run one complete simulation step
    pending -= 1
```

Required clock behavior:

- Every executed simulation step uses `Time.delta == 1` and graphics delta
  `1/60` while the step is active.
- Slow speeds naturally produce rendered frames with zero simulation steps.
- Invalid, negative, NaN, and infinite frame durations are rejected safely.
- Suspend, pause, reset, world replacement, and speed changes clear pending
  backlog.
- A rendered frame processes at most 1,024 ticks and spends at most 8 ms in the
  repeated-step loop by default.
- When saturated, discard whole pending ticks and retain only the fractional
  remainder. The game slows down instead of replaying a burst later.
- Record and display actual achieved speed separately from requested speed.

The CPU budget and hard tick cap must be named constants with direct unit tests.

### 3. Logic-loop integration

Wrap the original `Vars.logic` `ApplicationListener` and replace its entry in
`ApplicationCore.modules`. The wrapper delegates lifecycle methods and retains
the original `Logic` instance as the owner of game state, saves, and events.

Before installation, verify all reflection targets, concrete runtime types, and
the exact game build. Installation is transactional: if any check fails, leave
`Vars.logic`, the application module array, input, physics, and timing untouched.

During fixed-step mode, the wrapper executes the original logic once per clock
step. In multiplayer, editor mode, menus, and unsupported states, it resets the
clock and delegates one vanilla update.

All temporary changes to `Time.delta` and graphics delta must be restored in a
`finally` block, including when a simulation update throws.

### 4. Async physics integration

Move the relevant `AsyncCore` processes from once-per-render execution into each
simulation step. Preserve their original ordering and ensure no async job
outlives a pause, reset, world transition, or exception boundary.

Do not assume process indices without validating their concrete types first.
If v160.7 no longer provides the expected physics and avoidance processes, fail
installation rather than partially enabling fixed-step mode.

### 5. Input separation

Keep discrete UI, build, selection, and action input at one execution per
rendered frame. Advance continuous player movement, rotation, aiming, weapon
state, and boost state once per simulation step.

Use behavior derived from the exact Mindustry v160.7 `DesktopInput` source.
Record the upstream file and license in `NOTICE`, and keep derived code isolated
so its GPL obligations are explicit.

### 6. Mode transitions

Define fixed-step eligibility as all of the following:

- no active network session;
- not in the editor;
- an active, playing single-player game state;
- all validated runtime components still match the installed wrappers.

Re-evaluate eligibility before every step in a batch. If a step pauses, resets,
or replaces the world, stop the batch immediately and clear pending time.

Multiplayer entry must execute vanilla timing before its first world update.

### 7. User interface

Preserve the logarithmic speed slider and reset control. Add a second line that
shows one of:

- actual achieved speed;
- paused;
- normal speed enforced for multiplayer or editor;
- unsupported runtime with a concise failure reason.

The UI must not mutate shared Mindustry styles. It must remain hidden outside a
game HUD and must not overlap the mobile UI because mobile is unsupported.

## Delivery phases

### Phase 0: v160.7 runtime contract

- Pin official Mindustry 160.7 artifacts by SHA-256.
- Record the exact private fields, module ordering, input type, async process
  types, and lifecycle callbacks required by the design.
- Resolve licensing before source work: either avoid copying Mindustry input
  code or publish the combined Java rewrite under GPL-compatible terms while
  preserving the original TimeControl attribution and Apache-2.0 notices.
- Create an executable compatibility probe that performs no mutation.
- Stop if the runtime contract cannot be validated without unsafe assumptions.

**Exit:** The probe passes on official desktop 160.7 and fails cleanly on an
intentionally mismatched build.

### Phase 1: deterministic clock

- Implement the accumulator independently of Mindustry runtime classes.
- Cover slow speed, fast speed, fractional remainder, pause, reset, invalid
  input, CPU budget, hard cap, and saturation backlog policy.

**Exit:** Clock tests are deterministic and independent of wall-clock timing.

### Phase 2: logic wrapper

- Install the wrapper transactionally.
- Run complete fixed steps with scoped delta overrides.
- Restore all state on normal return and exceptions.
- Preserve vanilla delegation outside eligible single-player gameplay.

**Exit:** Tick count and state tick progression match requested fixed steps at
30, 60, and 144 rendered FPS.

### Phase 3: physics and input

- Integrate async physics per simulation step.
- Separate frame input from continuous simulation input.
- Add pause/reset/world-transition interruption tests.

**Exit:** Player motion and collision results remain equivalent across supported
render rates and requested speeds.

### Phase 4: gameplay parity suite

Create disposable fixtures using real Mindustry blocks and entities. Compare
equal simulated time at `1x`, `4x`, `32x`, `64x`, `128x`, and `256x`, rendered
at 30, 60, and 144 FPS.

Required fixtures:

- conveyor, junction, router, and distributor transport;
- multi-input factory production and output blocking;
- unit movement, collision, death, and removal;
- wave progression and scheduled events;
- player movement and rotation;
- pause during a multi-step frame;
- world reset or replacement during a multi-step frame;
- CPU saturation without delayed backlog replay.

**Exit:** Fixed-step results match the 1x reference after equal executed tick
counts. Any intentional tolerance must be numeric, justified, and fixture-local.

### Phase 5: UI, packaging, and migration

- Build the Java UI and actual-speed display.
- Remove the JavaScript runtime from the packaged artifact.
- Add deterministic packaging, checksums, and client smoke tests.
- Update metadata, compatibility text, installation instructions, attribution,
  and license notices.
- Use a new semantic version and a `[v160]` release title.

**Exit:** A clean Mindustry 160.7 profile installs the release from the Mod
Browser, starts without script or class errors, runs the parity fixture, and
returns to vanilla timing in multiplayer and editor modes.

## Acceptance gates

The implementation is not complete until all gates pass:

1. No oversized simulation delta is used during fixed-step mode.
2. Equal executed ticks produce parity across supported rendered frame rates.
3. Conveyor, production, physics, and unit fixtures pass through `256x` when the
   machine can sustain the requested speed.
4. CPU saturation reports lower actual speed and does not accumulate a future
   burst.
5. Multiplayer and editor execute vanilla timing from their first update.
6. Pause, reset, world replacement, and exceptions stop the current batch and
   restore all temporary global state.
7. Unsupported environments fail before runtime mutation.
8. The release artifact is reproducible from its tag and its checksum is
   published.
9. A real desktop client smoke test passes on official Mindustry 160.7.

## Known risks

- Private-field and module-array integration is inherently version-specific.
- Replacing input or async components can conflict with other Java mods.
- Very high requested speed can consume substantial CPU even with a frame
  budget; actual speed will vary by world complexity and hardware.
- Copied or adapted Mindustry input logic carries GPL licensing obligations.
- A technically passing headless or unit test does not replace desktop runtime
  validation because this design depends on SDL graphics and input internals.

## Release policy

- Do not overwrite v0.3.0 or claim that it uses fixed steps.
- Publish the Java rewrite as a new version-specific release.
- Keep the last JavaScript release available for rollback, with its simulation
  limitation documented.
- Do not expand support beyond v160.7 until each additional build receives its
  own runtime-contract probe and full parity run.

## Explicit non-goals

- Multiplayer acceleration.
- Server-side time control.
- Mobile support in the first Java release.
- Automatic compatibility with arbitrary game builds.
- Perfectly achieving requested speed when hardware cannot execute the required
  number of complete simulation ticks.
- General-purpose patching of Mindustry's engine or unrelated gameplay bugs.
