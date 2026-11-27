# Week 1 evidence

Keep this short. Bullet points, a small table and one diagram or code trace are enough. This is your checkpoint record, not an extra report. Replace the prompts as you work.

Name: Casper Aaröe
Exact Unity version: 6000.3.26f1
Checkpoint commit / repository: https://github.com/Svenebanan1/Casper_AI_SalvageFix (exact commit hash in my Omniway submission)
Current stage and anything unfinished: All stages done

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

What I saw in the F1 panel: Travel went Pending -> Moving. Decisions kept going up while Requests stayed at 1, and Requests went to 2 when it switched to Deliver.

### Stage 1 experiments

- The NavMesh baked on Yard Geometry decides where the rival can walk. The NavMeshAgent on the rival does the moving. When I moved a shelf and rebaked, the route then changed.
- Speed, acceleration and angular speed change how the rival moves, not where it can go.
- With a stopping distance above about 1.7, the rival stops too far from the scrap. It thinks it has arrived, but it never picks the scrap up because pickup needs 1.1 m. I think it still works at 1.6 because it slides a bit when it brakes.
- When the player took the scrap the rival was going for, the rival stopped. The code should make it forget that scrap and pick another one or explore. (What the panel showed is in the F5 trace under Stage 3.)

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

### Stage 4: recover from failed routes

- Where `Blocked` comes from (supplied code): when a route fails, `RivalBrain.Update` sets `failedUntil[id] = Time.time + 6` and `PickupSensor.Capture` marks that pickup `Blocked`. The 6 seconds is only a wait before trying again. It does not prove the route will work later.
- Before: `IsEligible` ignored `Blocked`, so in F6 my Stage 3 keep-target loop kept #0 even after its route failed, and the rival never went for #1.
- Changed rule (`DecisionPolicy.isEligible`, which replaced the old `IsEligible`): a pickup only counts if it is not `Blocked` and `Available && (Visible || Remembered)`. Both loops in `Infer` use it, so a blocked pickup can't be kept or newly chosen. If every known pickup is blocked, nothing is selected and the rival explores.
- F6 after: the rival still tries #0 first, because it can't know the route is impossible until it fails. Then #0 is blocked, so it goes for #1, picks it up, delivers it and explores.
- Limitation: after 6 seconds #0 is no longer blocked, so the rival can try it again and fail again. It never learns that the platform is disconnected.
- Tests: 17 pass, 0 fail.

## Scenario record

| Scenario | Predicted result | Observed result and short evidence / remaining issue |
|---|---|---|
| Normal loop and restart | In F2 the rival should go for the nearest scrap and bank it at the depot, and keep doing this until someone has 5 or the 2 minutes run out. R should restart the match. | The rival collected scrap and banked it at its depot over and over until one side reached 5. R restarted the match. |
| Carrying cargo | When carrying cargo the rival goes into the delivery phase and prioritizes leaving the scrap at depot before doing anything else | The rival went straight to its depot first (Task: Deliver, reason "Cargo held: return to depot") and only went for scrap after banking it. |
| Hidden / remembered pickup | In F4 the rival should ignore the nearer hidden scrap and go for the visible one further away | After restart in F4 the rival ignored the nearer scrap #0 behind the test wall and went for the visible scrap #1 further away. In F2 all nearby scrap is visible, so it still picks the nearest one (#10) as before. |
| Target taken | In F5 the rival should forget the taken scrap and pick another known scrap, or explore if it knows none | The rival was going for #0. After the helper gave it to the player, the rival switched to Explore, found new scrap #1 and collected it. Panel: Memory was 1, went to 0 when #0 was taken, and back to 1 when the rival chose #1. Reason "New target: nearest eligible candidate". |
| Unreachable target | In F6 the rival should ignore the unreachable scrap and pick another known scrap, or explore if it knows none | The rival first went for #0 on the platform. The route failed, #0 was blocked for 6 seconds, and it went for #1 instead, picked it up, banked it and started exploring. |
| No usable pickup | In F7 there is no scrap, so the rival should only explore the survey points without crashing or inventing a target, and the 20-second match should end 0-0 as a draw. | The rival explored the survey points and the match ended in a draw. |

Supplied test result, with any added tests identified: 17 pass, 0 fail (expected 17 / 0 after Stage 4), no tests added. All three Stage2_ tests and both Regression_ tests pass. `Stage2_PreviouslyKnownPickupMayRemainKnownOutsideSight` already passed in the starter (its only candidate is available, so the naive rule picked it anyway). My change made `Stage2_HiddenUnknownPickupIsNotAUsableTarget` and `Stage2_VisibleAlternativeBeatsNearerHiddenPickup` pass. For Stage 3, `Stage3_UsableCurrentTargetSurvivesCloserDistraction` now passes because of my change; `Stage3_UnavailableRememberedTargetIsReleased` already passed in the starter. The three Stage4_ tests now pass because of my Stage 4 change.
Live walkthrough checked by / recording link if used:

## One correction and one independent check

The original weakness and why it happened: in F6 the rival kept going for #0 after its route failed, because the policy ignored `Blocked`, so the failure feedback never reached the decision.
What I changed and why it addresses the cause: I added `Blocked` to `isEligible`. Both the keep-target loop and the nearest search use it, so a pickup whose route just failed can't be kept or chosen for 6 seconds.
My counterexample or experiment: prediction, observed result, and one limitation: two equally near visible pickups (squared distance 4). Prediction: because the nearest search uses `<`, not `<=`, the first one in the list wins, not the lower ID. Observed (decision snapshots): list [#3, #1] chose #3, list [#1, #3] chose #1, and with #3 as the current target it kept #3 ("Keeping current target: still usable"). Limitation: a tie is decided by list order, which comes from the order of `match.Pickups`, so it is an arbitrary choice, not a smarter one like "closer to my depot". Stage 3 commitment at least stops the rival switching back and forth between the two.

## Collaboration and help

People I discussed or playtested with: 
Code, sources or substantive tools/AI assistance I used, and how I verified them: I used Claude Code (AI) to help me understand the code and to fix my Stage 1 trace and notes. I checked it by reading the code and running the game to see if it matched my understanding. For Stage 2, Claude Code drafted the Stage 2 text in this file; I checked it against `DecisionPolicy.cs`, `PickupSensor.Capture`, the Stage 2 tests and the F4 scene. For Stage 3, Claude Code explained the steps, gave me the code for the retained-target loop in `Infer` and the reason text in `Choose`, fixed two typos in `DecisionPolicy.cs`, ran the EditMode tests (14 pass, 3 fail) and drafted the Stage 3 text in this file. I verified it by running the game and checking the F5 scene and the Stage 3 tests. For Stage 4, Claude Code walked me through the steps, checked my change by running the supplied tests outside Unity, and drafted the Stage 4 text in this file. For Stage 5, Claude Code filled in the scenario table from my notes and the code, and ran the tie experiment outside Unity.
If unfinished: what works, exact reproduction steps, and the help I need:
