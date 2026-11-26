# Week 1 evidence

Keep this short. Bullet points, a small table and one diagram or code trace are enough. This is your checkpoint record, not an extra report. Replace the prompts as you work.

Name: Casper Aaröe
Exact Unity version: 6000.3.26f1
Checkpoint commit / repository: 
Current stage and anything unfinished: Stage 0-3 done. Stage 4 and 5 not started. 

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

### Stage 2: respect perception and memory

- Changed rule (`DecisionPolicy.IsEligible`): a pickup only counts if it is `Available && (Visible || Remembered)`. Distance is only compared after this check, so a hidden pickup can no longer win just by being nearer.
- Before: the loop only checked `Available`, so the rival could choose a hidden pickup it had never seen, as if it could see through walls.
- Accepted: available + hidden + remembered -> eligible. The rival saw it earlier and selected it, so it is only acting on what it already knew.
- Rejected: available + hidden + not remembered -> not eligible. The rival has never seen it, so choosing it would use information it does not have.
- Memory: `PickupSensor.Capture` marks a pickup as remembered when its ID is the rival's current target (`remembered = item.Id == rememberedId`). The sensor only keeps one target, not a history. In the starter the rival could select an unknown hidden pickup, and the next tick would mark it remembered, so Memory did not prove it had been seen. Now a pickup can only be selected after it was visible, so the memory is legitimate.

### Stage 3: keep a useful target

- Changed rule (`DecisionPolicy.Infer`): before the nearest search, it looks for the candidate whose `Id` is the same as `observation.CurrentTargetId`. If that pickup is still eligible, it is selected again and `RetainedTarget` is true. If not, the Stage 2 nearest search runs and `RetainedTarget` is false.
- Before: every decision picked the nearest eligible pickup, so a closer scrap showing up made the rival drop the one it was already going for.
- Example: the rival is going for #4 (squared distance 16) and #5 becomes visible at squared distance 1. It keeps #4, because #4 is still available and remembered. If #4 is taken, it is not eligible any more, so the nearest search picks #5.
- I match on the pickup's `Id`, not the position in the array. If the current target is -1 or not in the list, nothing matches and it falls through to the nearest search.
- Changed reason (`DecisionPolicy.Choose`): Collect now says "Keeping current target: still usable" when the target was kept and "New target: nearest eligible candidate" when it was newly picked. The task and target are the same as before.
- Release is supplied code, not mine: in `RivalBrain.Update`, `if (rememberedId >= 0 && !match.IsAvailable(rememberedId)) rememberedId = -1;` runs before sensing. So when the scrap is taken, the rival forgets it, `CurrentTargetId` is -1 in the next observation, and my code picks something else. What I added is the commitment to a target that is still useful.

State transitions:

- Explore -> Collect: an eligible pickup (available, and visible or remembered) is selected.
- Collect -> Deliver: the rival picks the scrap up, so it has cargo, and cargo always goes first.
- Deliver -> Collect or Explore: after delivery the cargo is gone. If it knows an eligible pickup it collects, otherwise it explores.
- Collect -> another Collect target or Explore: the current target stops being usable (taken), so it is released and the nearest eligible pickup is chosen, or Explore if there is none.

F5 (target taken): the rival first chose scrap #0 with reason "New target: nearest eligible candidate", then "Keeping current target: still usable" on the next decisions. When the helper gave that scrap to the player, the rival switched to Explore, found scrap #1 and chose it with reason "New target: nearest eligible candidate". The rival did not try to go for the taken scrap #0, so it correctly released it.

## Scenario record

| Scenario | Predicted result | Observed result and short evidence / remaining issue |
|---|---|---|
| Normal loop and restart | | |
| Carrying cargo | | |
| Hidden / remembered pickup | In F4 the rival should ignore the nearer hidden scrap and go for the visible one further away | After restart in F4 the rival ignored the nearer hidden scrap #[id] and went for the visible scrap #[id] further away. Panel: Visible = [n], Memory = [n], reason "[reason]". In F2 all nearby scrap is visible, so it still picks the nearest one (#10) as before. |
| Target taken | In F5 the rival should forget the taken scrap and pick another known scrap, or explore if it knows none | The rival was going for #0. After the helper gave it to the player, the rival switched to Explore, found new scrap #1 and collected it. Panel: Memory was 1, went to 0 when #0 was taken, and back to 1 when the rival chose #1. Reason "New target: nearest eligible candidate". |
| Unreachable target | | |
| No usable pickup | | |

Supplied test result, with any added tests identified: 14 pass, 3 fail (expected 14 / 3 after Stage 3). All three Stage2_ tests and both Regression_ tests pass. `Stage2_PreviouslyKnownPickupMayRemainKnownOutsideSight` already passed in the starter (its only candidate is available, so the naive rule picked it anyway). My change made `Stage2_HiddenUnknownPickupIsNotAUsableTarget` and `Stage2_VisibleAlternativeBeatsNearerHiddenPickup` pass. For Stage 3, `Stage3_UsableCurrentTargetSurvivesCloserDistraction` now passes because of my change; `Stage3_UnavailableRememberedTargetIsReleased` already passed in the starter. The 3 remaining fails are the Stage 4 tests.
Live walkthrough checked by / recording link if used:

## One correction and one independent check

The original weakness and why it happened:
What I changed and why it addresses the cause:
My counterexample or experiment: prediction, observed result, and one limitation:

## Collaboration and help

People I discussed or playtested with: 
Code, sources or substantive tools/AI assistance I used, and how I verified them: I used Claude Code (AI) to help me understand the code and to fix my Stage 1 trace and notes. I checked it by reading the code and running the game to see if it matched my understanding. For Stage 2, Claude Code drafted the Stage 2 text in this file; I checked it against `DecisionPolicy.cs`, `PickupSensor.Capture`, the Stage 2 tests and the F4 scene. For Stage 3, Claude Code explained the steps, gave me the code for the retained-target loop in `Infer` and the reason text in `Choose`, fixed two typos in `DecisionPolicy.cs`, ran the EditMode tests (14 pass, 3 fail) and drafted the Stage 3 text in this file. I verified it by running the game and checking the F5 scene and the Stage 3 tests.
If unfinished: what works, exact reproduction steps, and the help I need:
