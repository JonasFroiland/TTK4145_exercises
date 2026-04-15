# Exercise 5

---

## Kjøre løsningene

```bash
# Del 1 – Semaphores (D)
cd semaphore && rdmd semaphore.d

# Del 2 – Condition Variables (D)
cd conditionvariable && rdmd condvar.d

# Del 3 – Protected Objects (Ada)
cd protectedobject && gnatmake protectobj.adb && ./protectobj

# Del 4 – Message Passing: Request (Go)
cd messagepassing && go run request.go

# Del 4 – Message Passing: Priority Select (Go)
cd messagepassing && go run priorityselect.go
```

Output:
- Test 1: brukerne 0, 1, 2 kjører én om gangen i rekkefølge
- Test 2: alle partallsbrukere (høy prioritet) før oddetallsbrukere (lav)
- Test 3: brukerne 6 og 7 venter til 1–5 er ferdig
- Del 1 og 2 printer `All tests pass`

---

## Bug i original semaphore-kode

```c
void allocate(int priority){
    Wait(M);
    if(busy){
        Signal(M);           // (A) slipper mutex
        Wait(PS[priority]);  // (B) blokkerer på semaphore
    }
    busy = true;
    Signal(M);
}

void deallocate(){
    Wait(M);
    busy = false;
    if(GetValue(PS[1]) < 0){
        Signal(PS[1]);
    } else if(GetValue(PS[0]) < 0){
        Signal(PS[0]);
    } else {
        Signal(M);
    }
}
```

Mellom (A) og (B) er mutex sluppet, men tråden har ikke blokkert ennå. Hvis `deallocate` kjører i dette vinduet ser den `GetValue == 0` og kaller `Signal(M)` i stedet for `Signal(PS[priority])`. Tråden blokkerer så på (B) og ingen signalerer den noen gang. Deadlock.

**Fix:**
Tell opp `numWaiting[priority]` mens vi fortsatt holder M:
```d
mtx.wait();
if(busy){
    numWaiting[priority]++;   // telt mens mutex holdes
    mtx.notify();
    sems[priority].wait();
    numWaiting[priority]--;
}
busy = true;
mtx.notify();
```
`deallocate` sjekker `numWaiting` i stedet for `GetValue`. `numWaiting` er alltid korrekt fordi det bare endres mens M holdes.

---

## Løsningene

### Del 1 – Semaphores
- `mtx` er mutex (init 1), `sems[0/1]` er prioritetskøene (init 0)
- `numWaiting[p]` teller ventende tråder på prioritet `p`
- I `deallocate`: vi signalerer en prioritetssemaphore **uten** å slippe M – den vekte tråden arver M og slipper den etter å ha satt `busy = true`

### Del 2 – Condition Variables
- `allocate` legger id inn i en prioritetskø, så looper: `while(queue.front != id) { cond.wait(); }`
- `cond.wait()` slipper mutex og sover atomisk – dette er det semaphore-løsningen må simulere manuelt med `numWaiting`
- `deallocate` popper fronten og kaller `notifyAll()` – alle vekkes og sjekker på nytt

### Del 3 – Protected Objects (Ada)
```ada
entry allocateHigh(val: out IntVec.Vector) when not busy is ...
entry allocateLow(val: out IntVec.Vector)  when not busy and allocateHigh'Count = 0 is ...
```
- `allocateHigh'Count` er antall tasks i køen på den entryen
- `allocateLow` er blokkert så lenge det er noen som venter på `allocateHigh`
- Ada re-evaluerer guards atomisk etter hver `deallocate`
- En protected object er i bunn og grunn en **monitor** – låsing skjer automatisk, så hele klassen av bugs fra del 1 & 2 forsvinner

### Del 4a – Message Passing: Request
- Resource manager er en goroutine med prioritetskø
- Bruker sender `ResourceRequest{id, priority, replyChan}` på `askFor`
- Hvis ikke opptatt: sender ressursen direkte til `replyChan`
- Hvis opptatt: legger requesten i prioritetskøen
- På `giveBack`: dequeur høyest prioritet og sender til den

### Del 4b – Message Passing: Priority Select
Go's `select` er tilfeldig blant klare cases, så vi simulerer prioritet med `default`:
```go
for {
    select {
    case takeHigh <- res:
        res = <-giveBack
        continue
    default:
    }

    select {
    case takeHigh <- res:
        res = <-giveBack
    case takeLow <- res:
        res = <-giveBack
    }
}
```
Best-effort, ikke garantert. Fungerer for testene fordi det alltid er en høyprioritetsbruker klar når ressursen frigjøres.

---

## Diverse

**`getValue` på semaphores** – verdien er utdatert i det du leser den, en annen tråd kan ha endret den. Iboende race condition, og finnes ikke engang på alle plattformer (Windows eksponerer det ikke).

**N prioriteter:**
| Mekanisme | Utvidelse |
|---|---|
| Semaphore | N semaphores + N `numWaiting` – skalerer men blir verbose |
| Condition variable | Endre comparatoren i prioritetskøen – skalerer fint |
| Protected object | Trenger N separate entries – ikke elegant |
| Message passing (request) | Endre comparatoren – skalerer fint |
| Message passing (select) | N kanaler + N nestede `default`-sjekker – rotete |

**Condition variables vs monitors vs protected objects:**
- **Condition variable**: rå primitiv, låser/låser opp mutex manuelt rundt `wait`/`notifyAll`
- **Monitor** (Java `synchronized`): mutex håndheves automatisk ved inngang, betingelsen er fortsatt brukerdefinert
- **Protected object** (Ada): håndhever også *entry-betingelsene* atomisk via guards, runtime tar seg av både låsing og betinget tilgang
