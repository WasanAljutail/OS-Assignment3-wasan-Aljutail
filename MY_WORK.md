# 📝 MY_WORK: Student Information, Development Log, Reflection & Answers

> This is the **only file** your instructor reads to grade Parts 3 and 4 (documentation and video). Everything you write here must be **in your own words**.

---

## 👤 Student Information

| Field | Your Answer |
|-------|-------------|
| **Full Name** | Wasan Abdulrahman Aljutail |
| **Student ID** | 446051756 |
| **University Email** | 446051756@std.psau.edu.sa |
| **GitHub Username** | WasanAljutail |
| **Repository Link** | https://github.com/WasanAljutail/OS-Assignment3-Wasan-Aljutail |
 
---

## 🎥 Video Link

**Video Link**: [https://drive.google.com/file/d/1NIzIU5Y0tnsRJeIE9LhIxptvjIDBJPR3/view?usp=drivesdk]

---

# Part A: Development Log (0.5 mark)

### Entry 1 - [November 4, 2026, 5:15 PM]
**What I did**: Forked the starter repository, configured my GitHub repository, set my student ID, and ran the broken starter code.

**Details**:
- Forked `makopt/OS-Assignment3-Starter-481` to my GitHub account and renamed it to `OS-Assignment3-Wasan-Aljutail` while keeping visibility set to Public.
- Opened `SchedulerSimulationSync.java`, set `studentID = 446051756` on line 304, and committed the change.
- Executed the starter code in my IDE and saved the console output showing race conditions, lost updates, and high busy-waiting poll counts.

**Challenges**: Seeing multiple CPUs attempt to grab and start the exact same process thread simultaneously, causing `IllegalThreadStateException` and inconsistent counter outputs.

**Solution**: Inspected `ReadyQueue.poll()` and `SharedStats` methods and identified the unprotected read-modify-write sequences around `RaceWindow.pause()`.

**Time spent**: 50 minutes

---

### Entry 2 - [November 6, 2026, 7:40 PM]
**What I did**: Implemented Feature 1 by protecting `ReadyQueue` with a `ReentrantLock`.

**Details**:
- Imported `java.util.concurrent.locks.ReentrantLock` and declared `private final ReentrantLock queueLock = new ReentrantLock();` inside `ReadyQueue`.
- Wrapped the bodies of `add()`, `poll()`, `processOf()`, and `snapshot()` in `queueLock.lock()` and `try { ... } finally { queueLock.unlock(); }`.
- Tested the code to verify that no two CPUs ever pick the same `Thread` object from `list`.

**Challenges**: Ensuring that every read-only helper method (`processOf` and `snapshot`) was also locked so no thread reads `HashMap` or `LinkedList` mid-modification.

**Solution**: Applied the same `queueLock` across all four methods in `ReadyQueue` and guaranteed lock release inside `finally` blocks even if exceptions occur.

**Time spent**: 1 hour

---

### Entry 3 - [November 9, 2026, 6:10 PM]
**What I did**: Implemented Feature 2 by adding fine-grained mutex locks to `SharedStats`.

**Details**:
- Declared three separate `ReentrantLock` instances (`contextSwitchLock`, `completedProcessLock`, `serviceTimeLock`) in `SharedStats`.
- Protected all three mutator methods (`incrementContextSwitch`, `incrementCompletedProcess`, `addServiceTime`) and all three getter methods (`getContextSwitchCount`, `getCompletedProcessCount`, `getTotalServiceTime`).
- Verified that the first three lines of the verification block consistently matched the expected values.

**Challenges**: Deciding between a single coarse-grained lock for `SharedStats` versus separate fine-grained locks for each independent counter.

**Solution**: Chose fine-grained per-variable locks because the three counters are completely independent, allowing concurrent updates to different statistics without unnecessary blocking.

**Time spent**: 55 minutes

---

### Entry 4 - [November 12, 2026, 8:20 PM]
**What I did**: Implemented Feature 3 by replacing busy waiting with a counting `Semaphore`.

**Details**:
- Declared `private final Semaphore availableProcesses = new Semaphore(0);` inside `ReadyQueue`.
- Added `availableProcesses.release()` at the end of `ReadyQueue.add()` and `availableProcesses.acquire()` in `CpuWorker.run()` instead of the `Thread.sleep(20)` polling loop.
- Added `wakeAllCpus(SharedStats.totalCPUs)` when `completedProcessCount == totalProcesses` so idle CPUs blocked on `acquire()` wake up and exit cleanly.

**Challenges**: Preventing CPUs from remaining blocked forever on `acquire()` after the final process completed, which initially triggered the watchdog timeout.

**Solution**: Released `totalCPUs` permits once the final process incremented `completedProcessCount` to `totalProcesses`, and added a termination check immediately after `waitForProcess()`.

**Time spent**: 1 hour 15 minutes

---

### Entry 5 - [November 15, 2026, 4:00 PM]
**What I did**: Ran 5 consecutive verification tests and completed the written documentation in `MY_WORK.md`.

**Details**:
- Executed the completed `SchedulerSimulationSync.java` five consecutive times in my IDE and confirmed all four verification checks showed `✔` with `Busy-waiting polls = 0`.
- Recorded the Development Log entries, Reflection answers, and Technical Answers in `MY_WORK.md`.

**Challenges**: Organizing clear, concise explanations linking my specific code structure and simulation output to Chapter 5 synchronization theory.

**Solution**: Reviewed each class method and verification output line by line to ground every explanation in concrete evidence from my runs.

**Time spent**: 1 hour 10 minutes

---

## Development Log Summary

**Total time spent on assignment**: 5 hours 10 minutes

**Most challenging part**: Designing the clean shutdown mechanism for Feature 3 so that CPU threads blocked on `availableProcesses.acquire()` exit gracefully once all processes finish without deadlocking or triggering the watchdog.

**Most interesting learning**: Observing firsthand how `RaceWindow.pause()` exposes read-modify-write race conditions and how combining mutex locks with a counting semaphore eliminates both data corruption and wasted CPU cycles.

**What I would do differently next time**: Write small helper test traces earlier when testing semaphore permit counts so edge cases during thread termination are caught immediately.

---

# Part B: Reflection (0.5 mark)

## Question 1: What did you learn about synchronization?

**Your Answer:**
Working on this assignment showed me how easily concurrent threads corrupt shared data when read-modify-write operations are left unprotected. In the starter code, `RaceWindow.pause()` simulated context preemption between reading a variable and writing back the updated value, causing lost updates in `SharedStats` and duplicate thread execution in `ReadyQueue`. I learned how `ReentrantLock` enforces mutual exclusion on critical sections, and why placing `unlock()` inside a `finally` block is essential to prevent permanent lock retention if an error occurs. I also learned the difference between mutual exclusion and thread coordination: locks protect shared state, whereas a counting `Semaphore` coordinates producer-consumer availability without wasting CPU cycles. Seeing the `Busy-waiting polls` drop from hundreds of wasted checks down to zero after adding `availableProcesses` made the theoretical benefits of blocking synchronization concrete.

## Question 2: What was the most challenging part of this assignment?

**Your Answer:**
The most challenging part of this assignment was implementing the clean shutdown logic for the CPU worker threads in Feature 3. When I first replaced the polling loop with `availableProcesses.acquire()`, the simulation finished executing all processes properly, but the remaining idle `CpuWorker` threads stayed blocked indefinitely waiting for new permits. Because those threads never returned from `acquire()`, the main thread's watchdog timer expired and flagged a synchronization hang. Solving this required carefully tracing the exact moment `completedProcessCount` reaches `totalProcesses` so that the finishing CPU calls `wakeAllCpus(SharedStats.totalCPUs)` to release enough permits for every waiting worker. Adding an explicit check on `completedProcessCount >= totalProcesses` right after `waitForProcess()` ensured that woken CPUs exit the loop cleanly without attempting to poll an empty queue.

## Question 3: How did you overcome the challenges you faced?

**Your Answer:**
I overcame these challenges by breaking the implementation into three isolated steps and verifying each feature thoroughly before moving to the next. Before writing any synchronization code, I read `SchedulerSimulationSync.java` from top to bottom and ran the starter version to record its failure modes and exact expected values for my student ID `446051756`. Whenever a thread blocked or a counter mismatched, I traced the sequence of `lock()`, `acquire()`, and `release()` calls across `ReadyQueue`, `SharedStats`, and `CpuWorker` to verify that every acquisition had a matching release path. Testing the program across at least five consecutive runs after each change confirmed that my fixes were deterministic rather than lucky timing outcomes. Reviewing the Chapter 5 concepts in `README.md` alongside my code also helped verify that my lock granularity and semaphore initialization followed standard operating system design patterns.

## Question 4: Where can synchronization concepts be applied in real-world applications?

**Your Answer:**
Synchronization concepts are critical in banking and payment systems where concurrent transfers read and update the same account balance simultaneously. Without mutual exclusion via mutex locks around the read-modify-write critical section, two simultaneous withdrawals could read the same initial balance and cause a lost update or overdraft. Similarly, online ticket or airline seat reservation platforms rely on locks to ensure that two users cannot book the last remaining seat at the exact same millisecond. Counting semaphores are widely used in web server thread pools and database connection pools to track available worker threads or connections without wasting CPU time on busy waiting. Finally, print spoolers and background task queues operate just like our `ReadyQueue`, using locks to protect queue pointers and semaphores to signal worker threads whenever new jobs arrive.

---

# Part C: Technical Answers (0.5 mark)

## Question 1: Race Condition in Your Own Output

**Your Answer:**
In the starter run with my student ID `446051756`, multiple `CpuWorker` threads concurrently accessed `ReadyQueue` and `SharedStats` without mutual exclusion, causing lost updates and thread collisions where observed counters fell below expected values (and `Busy-waiting polls` exceeded 0). For example, during a lost update on `contextSwitchCount`, `CPU-1` reads `contextSwitchCount = 12` and pauses at `RaceWindow.pause()`, while `CPU-2` simultaneously reads the same stale value `12` before `CPU-1` writes `13`, resulting in `CPU-2` also writing `13` and losing one increment. On real hardware without `RaceWindow`, the exact same bug occurs whenever the OS scheduler preempts a thread during the nanosecond window between memory load, register increment, and memory store instructions. Because hardware preemption timing depends on unpredictable system load, the bug manifests sporadically and is notoriously difficult to reproduce and debug.

Verification block of the **original starter**:
