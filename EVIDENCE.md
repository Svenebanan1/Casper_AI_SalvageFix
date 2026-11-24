# Week 1 evidence

Keep this short. Bullet points, a small table and one diagram or code trace are enough. This is your checkpoint record, not an extra report. Replace the prompts as you work.

Name: Casper Aaröe
Exact Unity version: 6000.3.26f1
Checkpoint commit / repository: 
Current stage and anything unfinished: Stage 1

## Initial prediction

Three observations the rival needs, including one it might not have: Location of pickups, location of targets, and whether the target is reachable. The rival might not have the last one.

Seeing a pickup across a gap proves / does not prove reachability because: The rival can see the pickup but cannot reach it due to the gap. The rival needs to determine if there is a path to the pickup, which may not be obvious from visual observation alone.

One question about NPC behaviour: how does the rival decide which pickup to go for when multiple pickups are visible? Does it prioritize based on distance, value, or some other factor?

## One complete trace

Scene: F2 (normal). The first thing the rival does after I press Play. It starts at its depot with no cargo.

1. **Sees:** 3 scrap inside its 10 m range (#7, #10, #11), all visible. (`PickupSensor.Capture`) 
2. **Thinks:** it isn't carrying anything, so it picks the closest one, #10. (`DecisionPolicy.Infer`)
3. **Decides:** Collect #10, reason "Nearest candidate; reselected each tick". (`DecisionPolicy.Choose`)
4. **Asks for a route:** `RivalBrain` calls `Navigator.Request`, which calls `SetDestination`. Travel = Pending, Requests = 1.
5. **Moves:** Travel changes to Moving (`Navigator.Poll` -> `NavigationFeedback.Classify`). It keeps deciding every 0.15 s, but Requests stays at 1 because the target is the same.
6. **Result:** it picks up #10 when it gets within 1.1 m (`MatchController`). On the next decision it has cargo, so it switches to Deliver and Requests goes to 2.

Explore -> Collect -> Deliver -> Collect / Explore

What I saw in the F1 panel: [fill in]

### Stage 1 experiments

- The NavMesh baked on Yard Geometry decides where the rival can walk. The NavMeshAgent on the rival does the moving. When I moved a shelf and rebaked, the route [fill in].
- Speed, acceleration and angular speed change how the rival moves, not where it can go.
- With a stopping distance above about 1.7, the rival stops too far from the scrap. It thinks it has arrived, but it never picks the scrap up because pickup needs 1.1 m. I think it still works at 1.6 because it slides a bit when it brakes.
- When the player took the scrap the rival was going for, the rival stopped. The code should make it forget that scrap and pick another one or explore. [fill in what the panel said]

## Scenario record

| Scenario | Predicted result | Observed result and short evidence / remaining issue |
|---|---|---|
| Normal loop and restart | | |
| Carrying cargo | | |
| Hidden / remembered pickup | | |
| Target taken | | |
| Unreachable target | | |
| No usable pickup | | |

Supplied test result, with any added tests identified:
Live walkthrough checked by / recording link if used:

## One correction and one independent check

The original weakness and why it happened:
What I changed and why it addresses the cause:
My counterexample or experiment: prediction, observed result, and one limitation:

## Collaboration and help

People I discussed or playtested with: 
Code, sources or substantive tools/AI assistance I used, and how I verified them: I used Claude Code (AI) to help me understand the code and to fix my Stage 1 trace and notes. I checked it by [fill in].
If unfinished: what works, exact reproduction steps, and the help I need:
