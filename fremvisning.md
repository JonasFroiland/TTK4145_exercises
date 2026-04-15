# Exercise 5 – Lab Approval Guide

---

## Step 1: Run all solutions and show correct output

```bash
# Part 1 – Semaphores (D)
cd semaphore && rdmd semaphore.d

# Part 2 – Condition Variables (D)
cd conditionvariable && rdmd condvar.d

# Part 3 – Protected Objects (Ada)
cd protectedobject && gnatmake protectobj.adb && ./protectobj

# Part 4 – Message Passing: Request (Go)
cd messagepassing && go run request.go

# Part 4 – Message Passing: Priority Select (Go)
cd messagepassing && go run priorityselect.go
```

**What to point out in the output:**
- Test 1: Users 0, 1, 2 run in that exact order (each alone, no contention)
- Test 2: Even IDs (high priority) all execute before odd IDs (low priority)
- Test 3: Users 6 and 7 (low priority) wait until users 1–5 (high priority) are all done
- Parts 1 and 2 print `All tests pass`

---

## Step 2: Explain the bug in the original semaphore code

The TA will almost certainly ask about this.

**The buggy pseudocode:**
```c
void allocate(int priority){
    Wait(M);
    if(busy){
        Signal(M);           // (A) releases the mutex
        Wait(PS[priority]);  // (B) blocks on priority semaphore
    }
    busy = true;
    Signal(M);
}

void deallocate(){
    Wait(M);
    busy = false;
    if(GetValue(PS[1]) < 0){   // checks if anyone is waiting
        Signal(PS[1]);
    } else if(GetValue(PS[0]) < 0){
        Signal(PS[0]);
    } else {
        Signal(M);
    }
}
```

**The bug:** Between (A) and (B), the mutex is released but the thread hasn't blocked yet.
If `deallocate` runs in this window, `GetValue` returns 0 (the thread isn't in the semaphore queue yet),
so it calls `Signal(M)` instead of `Signal(PS[priority])`.
The thread then blocks at (B) — and nobody will ever signal it. **Deadlock.**

**The fix (our solution):**
Increment `numWaiting[priority]` *before* releasing M:
```d
mtx.wait();
if(busy){
    numWaiting[priority]++;   // counted while mutex is still held
    mtx.notify();
    sems[priority].wait();
    numWaiting[priority]--;
}
busy = true;
mtx.notify();
```
Now `deallocate` checks `numWaiting` instead of `GetValue`, and `numWaiting` is always
accurate because it is only modified while M is held.

---

## Step 3: Explain each solution and why it works

### Part 1 – Semaphores
- `mtx` acts as a mutex (initialized to 1)
- `sems[0]` and `sems[1]` are the priority queues (initialized to 0)
- `numWaiting[p]` counts how many threads are waiting at priority `p`
- In `deallocate`, when handing off the resource we signal a priority semaphore but do **not** release M — the woken thread inherits M and releases it after setting `busy = true`

### Part 2 – Condition Variables
- `allocate` inserts the caller's id into a priority queue, then loops: `while(queue.front != id) { cond.wait(); }`
- `cond.wait()` atomically releases the mutex and sleeps — **this is what the semaphore solution has to simulate manually with `numWaiting`**
- `deallocate` pops the front of the queue and calls `notifyAll()` — all waiters wake and re-check; only the new front proceeds

### Part 3 – Protected Objects (Ada)
```ada
entry allocateHigh(val: out IntVec.Vector) when not busy is ...
entry allocateLow(val: out IntVec.Vector)  when not busy and allocateHigh'Count = 0 is ...
```
- `allocateHigh'Count` is the number of tasks queued at the `allocateHigh` entry
- `allocateLow` is blocked as long as there are high-priority waiters — no low-priority task can sneak through
- The runtime re-evaluates the guards atomically after every `deallocate` call
- Ada's protected object is essentially a **monitor** — locking is enforced automatically, which removes the entire class of bugs from parts 1 & 2

### Part 4a – Message Passing: Request
- The resource manager is a goroutine with a priority queue
- A user sends a `ResourceRequest{id, priority, replyChan}` on `askFor`
- If not busy: immediately sends resource to `replyChan`
- If busy: inserts request into priority queue
- On `giveBack`: dequeues highest-priority pending request and sends resource to it

### Part 4b – Message Passing: Priority Select
Go's `select` is random among ready cases, so we fake priority with `default`:
```go
for {
    // Non-blocking: give to high priority if anyone is waiting right now
    select {
    case takeHigh <- res:
        res = <-giveBack
        continue
    default:
    }

    // No high-priority waiter at this moment; wait for either
    select {
    case takeHigh <- res:
        res = <-giveBack
    case takeLow <- res:
        res = <-giveBack
    }
}
```
This is **best-effort**, not guaranteed. It works for the test cases because whenever
the resource becomes free, a high-priority user is already waiting on `takeHigh`.

---

## Step 4: Answer follow-up questions

**Why is `getValue` on semaphores dubious?**
> Its result is stale the moment you read it. By the time you act on it, another thread may have changed the count. It is inherently racy and is not even available on all platforms (Windows semaphores don't expose it).

**How would you extend to N priorities?**
| Mechanism | Extension |
|---|---|
| Semaphore | N semaphores + N `numWaiting` entries — scales but gets verbose |
| Condition variable | Just change the comparator in the priority queue — scales cleanly |
| Protected object | Needs N separate entries (`allocatePriority0`, `allocatePriority1`, ...) — not elegant |
| Message passing (request) | Just change the priority queue comparator — scales cleanly |
| Message passing (select) | Needs N channels + N nested `default` checks — gets unwieldy fast |

**How do condition variables, monitors, and protected objects differ?**
- A **condition variable** is a raw primitive: you must lock/unlock the mutex manually around every `wait`/`notifyAll`
- A **monitor** (Java `synchronized`) enforces the mutex automatically when entering a method, but the condition is still user-defined
- A **protected object** (Ada) goes further: it also enforces the *entry conditions* atomically via guards, so the runtime handles both locking and conditional access

**Which solution do you prefer and why?**
> Condition variables (or message passing with a request queue) generalize most cleanly to N priorities and keep the logic readable. Protected objects are the safest for simple cases because the runtime enforces correctness.
