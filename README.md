# Follow Target Project repository

## Development Process Notes
> These are all the notes I took down, recollecting the development process of Follow-Me mode!

### Key challenges that was addressed
1.	Target GPS Location Filtering
2.	Estimating the Target’s Heading
3.	Follow Perspective transitions

### Implementation details
1.	Target’s Altitude assumption problem
a.	GPS Altitude of QGC is in a wrong frame!
i.	Qt Documentation & QGC Debugging
b.	Terrain mode
i.	Aggressive flight & Jumpy behavior
c.	2D mode
i.	Most simple, but having constant altitude can result in drone going too high / low relative to the target’s actual height.
2.	Gimbal Pitch control
a.	Had 3 different Modes
b.	But didn’t create much difference in the end
c.	Was quite hard to debug exactly what the gimbal was doing (It’s logged in Quaternions!)
3.	uORB Logging
a.	Modifying the .msg file and incorporating it into the Code
b.	PlotJuggler is awesome for debugging what’s going on in the system!
i.	Flight Review web interface doesn’t show everything ;P
c.	Building the ‘Visualizer’ was helpful for debugging problems
i.	Required parsing the uORB topic data
d.	Making sure the logging is as fast as possible
i.	`logged_topics.cpp`
4.	2nd order filter Natural Frequency selection Problem (March 1st)
a.	Actual testing showed that 1 rad/s is ok
i.	2 rad/s has some choky- moments, and it’s unclear why
ii.	4 rad/s was almost unusable!
b.	Filter Response graphing program
i.	Showed an obvious tradeoff between the responsiveness of the filter & smoothness
1.	You can’t have both!
ii.	Nyquis Freqency (2*PI / 4 => 1.5 rad/s)
1.	But should it have been no problem since it was 100Hz update?
c.	Finally it got hard-coded in the Flight Task!
5.	Yaw setpoint jittering (March 8th)
a.	Tried Filtering approach. Didn’t make much difference.
b.	Noticed big undesirable movements when the follow distance was short
i.	The cinematic shot, which was ruined with the yaw setting 😉
c.	Also dependent on the Yaw tuning
6.	RC Adjustment (March 11th)
a.	Handling Parameter changed logic was tricky
i.	Understanding the Parameter Update logic
1.	Wrote a blog post about it
ii.	Still, if you don’t “change” the parameter value, there’s no way to check if a QGC has sent the new value or not ☹
1.	Potential for having a `updated` info in Parameters 😊
b.	It is really COOL to have some control over the Follow Me Behavior!
i.	User Interface is incredibly important
ii.	But now the flight mode is not *interruptable by RC Stick movements ☹
1.	Discussion of the ‘Auto’ mode in general
iii.	You can try creative things!
1.	Cinematic shots
c.	Making sure that adjustments are ‘responsive’ is hard!
i.	Arbitrary time-buffer to make sure that the adjusted setpoint is not too *ahead of actual position
7.	Velocity Ramp-down logic (March 26th)
a.	Orbit Angle error based Ramp-down calculation is error prone
b.	Time-window calculation can be very sensitive.
i.	Can be mere ~1 degrees error that makes the FlightTask enter the velocity ramp-down condition
ii.	Target course estimate is very noise and we can’t rely on this completely!
8.	Jerk limited Trajectory Control Idea
a.	Not sensitive to changes in Target course estimate
b.	More generic control scheme
i.	Actual kinematics based, logical control instead of arbitruary time-window!
c.	Flight Task Compatibility check
i.	Flight Task doesn’t do it’s own Jerk-limited Trajectory generation, so we don’t need to worry about having the trajectory generation applied twice!
d.	Calculation of the Velocity Setpoint
i.	Braking-distance based max vel calculation logic
1.	It is not optimal, as it doesn’t consider the derivative of the raw orbit angle setpoint
a.	Which results in the orbit angle following the raw angle setpoint very ‘sluggishly’
2.	However, calculating the derivative via Filtering is very noise and unreliable!
a.	Calculating derivatives reliably is hard!
i.	Especially with 100Hz update and rapidly changing target course estimate
9.	Acceleration Limited trajectory control being too aggressive
a.	Jerk limited control (April 21st)
i.	Was quite slow in the beginning
ii.	Multiple tests to find the ‘sweet spot’
iii.	High Jerk: Makes it bit more responsive
iv.	High Acceleration / Velocity limit : More responsive
v.	Opted for setting ‘Maximum Velocity’ around Target
1.	As it turns out, perspective-change wise, velocity has the biggest impact (in transition time)
10.	Orbit circle overshooting behavior
a.	Acceleration Feedback inclusion (April 19th)
i.	Helped a lot!
ii.	Same case for Orbit mode as well
b.	Position Error based Velocity & Acceleration feedforward ramp-in idea (TJ) (April 13th)
i.	It can be useful for Fixed Wings
ii.	But didn’t show big difference in practice with the tests (triggering Follow Me from far)
1.	However, TJ noted correctly that it should only come into effect when the follow perspective is actually getting changed, so my test was ill-posed
c.	Position Controller Tuning check
i.	MPC tuning page utilization
1.	It was ok but in the end had MAX gains :P
ii.	Overshooting behavior is ok, it’s a typical 2nd order system response (TJ)
11.	GPS Position Error problem
a.	Didn’t notice until a controlled test (standing at the home position where the drone took off & activating Follow-Me mode) was performed
i.	Around 5 ~ 6 meters of significant bias was observed
ii.	Led to setting follow distance ‘long enough’ (> 15 meters) to compensate for this error for the camera output (target centering)
12.	2nd order filter Position overshooting problem (April 25th)
a.	Totally unexpected, but it started to appear at the end of the development cycle
b.	Took quite some time to debug what was happening & the overshoot was clear!
i.	Had to add the logic to compare TargetEstimator’s velocity estimate as well
1.	Catching the Edge case
ii.	There were other ideas as well such as using the Hysteresis (TJ)
c.	Removed the annoying 180 degree flip in Follow Perspective Finally!
13.	Memorable Bugs
a.	Orbit angle / rate setpoint which was not taking the follow distance into consideration, and was commanding +1 rad or -1 rad orbit angle setpoints (SITL test)
i.	Since it was the output of the “sign” function, duh!
b.	Second Order filter reset logic depending on TargetEstimator output’s timeout in the unit ‘seconds’, but was comparing to ‘microseconds’, which resulted in the timeout on every loop, which resulted in 2nd order filter not getting used at all :D
i.	It took so long to realize that this was an issue! I thought the choppy, 1 Hz output was the *filtered output
ii.	It created a big difference after the fix, but also showed a significant delay in tracking target position. The tradeoff was clear!
c.	Velocity Ramp-down zone test in Hoengg (March 24th)
i.	It was intuitively inferable that as the drone reached the target orbit setpoint, it was oscillating around as the time-window logic was prone to jittery setpoints :0
d.	 

### Key Points – Process of getting it merged
1.	Review Processes
a.	1st pass
i.	Making Functions const, by having clear input & output
ii.	Removing unnecessary Functions
iii.	Not changing the internal variables silently
b.	2nd pass
i.	Commenting on the ‘units’ of the Constants used
ii.	Comments on the ‘Parameters’
c.	3rd pass
i.	Renaming the variables and enums to follow Google Code syntax
ii.	
2.	Commit cleanup
a.	Dealing with the fact that it’s a *stacked PR (on rebasing)
b.	Editing the commits to remove unnecessary changes & accidental Merge conflict messages in the file!
3.	Reducing Board FLASH usage
a.	Editing the Kconfig file to remove unnecessary system commands
b.	Excluding FollowMe from Constrained Flash boards
c.	Got it merged finally yesterday (June 16th, Thursday!!)
Key Points – Auxilary things to do
1.	MAVSDK Edits
a.	Proto file edit to match Parameter Configuration
b.	Impl file edit to match Parameter name
c.	Building MAVSDK Server and Library and Locally Testing
i.	Superbuild
ii.	Shared Libraries flag
iii.	Install location
iv.	CMake configuration
1.	MAVSDK Doc contribution.
v.	Removing the old MAVSDK
d.	Updating MAVSDK Integration Tests to match new API
e.	Updating FollowMe example script
2.	FollowMe Functional Tests
a.	Editing to use the updated MAVSDK API
b.	Adding RC Adjustment Test
c.	It’s painstaking but worth it when it works, since from that point on this test is always reproducible!
d.	Tests are powerful!
3.	Follow Me User Documentation
a.	It is essential for the Users to be able to use it!
b.	Adding diagrams and making it intuitive
i.	Making it user-friendly!
c.	Adding a video
d.	Explaining the expected behavior and Instructions
e.	Review process
i.	Forming the sentence
ii.	Being clear in instructions

### Key Points – Testing and Logistics
1.	Dog peed on my bag while out in the park ☹
2.	Running around and checking the behavior is quite fun
a.	But I ran around so much 😉
3.	Having a bike was very helpful!
4.	Flashing the new firmware and testing was a cycle of waiting for the Mantis to boot-up, etc
a.	But the WiFi telemetry capability made the FollowMe testing procedure so effortless!
b.	Mantis takes quite some time to boot-up (sometimes 2 minutes, I think)
5.	After rebasing on master, you notice subtle differences
a.	E.g. Arming & Take-off behavior, due to the landing detection logic and parameter settings
6.	Having a drone with the camera integrated is extremely helpful for checking the behavior!
7.	It’s quite fun to have the drone follow you around 😊
What’s there left to do
1.	Adding the ‘leash’ mode, so you can drag around the drone like a dog!
2.	Fixing the GPS Altitude Error in QGC and utilizing the full potential of 3D Tracking!
3.	Removing the Terrain mode, as it is terrible 😉
4.	Adding better GUI in QGC to show how the Drone is following you around
5.	Making the orbit angle tracking snappier by having a better orbital rate setpoint calculation logic (from current braking-distance based logic)
6.	Test it on more agile platform with crazier targets
a.	Boat, Racing car
b.	Racing Quadcopter
7.	Figure out why 2nd order filter acts bad / jittery with 2 rad/s or higher natural frequency constant ☹
a.	This can lead to having a much more snappy Follow-Me behavior (with minimal delay)
8.	

### What I learned
1.	Development Process Cycle
a.	Algorithm brainstorming process
i.	Creating scripts and actually checking how the *ideal system behavior would be
1.	Implementing edge cases (e.g. Target course estimate trigger at the Velocity threshold)
ii.	Whiteboard sessions and discussing if each step makes sense
1.	E.g. Using the Filtered vs Raw velocity / position for Gimbal, etc.
b.	Review process
i.	You catch a lot of little errors and it’s really helpful to have comments!
1.	E.g. Mathieu pointing out in the `updateParams()` function, the fact that param-diff check was completely wrong!
ii.	You get insight into how other think differently
1.	E.g. Mathieu saying there’s way too much commit
c.	Testing
i.	A LOT of testing 😉
d.	Debugging
i.	uORB messages, and creating tools to analyze the log
e.	Rebasing and resolving merge-conflicts
2.	Git magic
a.	How to rebase: squash, edit freely!
i.	`rebase -i` and doing the `edit` really blew my mind. You can CHANGE the commit’s content!
ii.	Once you edit the commit to remove the accidental changes like not removing the Merge Conflict messages in the file, the commit that removed it later on will get discarded automatically!
1.	It’s not so painful to remove unintentional changes 😊
b.	Git checkout <file> can remove all the changes in that file!
i.	Git restore also works
1.	But not always :P
c.	What Merge Conflicts actually mean
i.	It’s when you try to change the same part of the file at the same time and the changes are *different!
1.	Easy in theory, hard to grasp in practice!
d.	How git treats commits with different hash differently!
i.	It doesn’t matter what the content of that commit is, if it has a different hash (e.g. can happen frequently while rebasing), Git will think that it’s a completely new / different change!
e.	How git commit & checkout & HEAD actually works
i.	All the default `git diff` originates from the difference between the HEAD commit and current status of the repository!
f.	Git amend
i.	It’s really useful for making sure the commit history is clean (no redundant commits)
1.	But of course you need to ‘force push’ a lot, since the commit hash changed!
g.	Git stage utility
i.	You can `pop` but also `apply` the staged changes!
ii.	It’s not just for having your mess cleaned up 😊
h.	How to cherry-pick new features & updated library commits
i.	They will show up as the ‘different’ commit from what’s in the master branch, and this must be resolved with the rebase, there’s no other way!
1.	Since hashes are just so deterministic.
ii.	How to develop a new feature that is useful for the PR & adding it to the PR after having it open as a separate PR
iii.	Parallel development & keeping everything in sync.
3.	Git submodules ☹
a.	How to remove / add the git submodule change commits
i.	At first it was really not clear!
b.	What `git submodule update –recursive` really means
i.	`--init` is only necessary when the submodule doesn’t exist (isn’t cloned yet)!
c.	Git doesn’t checkout the submodule commits when you change branch!
4.	Git good practices
a.	Before pushing, do a ‘dry run’ by `-n`, to make sure no accidental changes occur
i.	It even shows that the new branch will be created in the remote repository!
b.	Force pushing should be done carefully!
i.	It’s necessary for updating the `rebase` process, but 
c.	When you switch branches, it’s best to remove files & undo the diffs / stage them to have the `git status` output nothing
i.	Makes it easier to do git commits (`git commit -a`)
ii.	But still, I don’t always succeed, especially with submodule ‘dirty’ commits 😊
5.	Collaboration / Feedback are powerful
a.	It really unlocks the stale stage you may get into
b.	Fresh ideas and discussions are helpful!
