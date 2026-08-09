# Jitter

> Jitter is a small random delay added to retry wait times to prevent many clients from retrying at exactly the same moment.

Difficulty: 🟢 Beginner
Time: 8 min
Prerequisites:
- [Retry](../distributed-systems/retry.md)
- [Exponential Backoff](../distributed-systems/exponential-backoff.md)

---

## Definition

Jitter is randomness deliberately introduced into retry timing. When added to exponential backoff, it desynchronizes clients that would otherwise retry in synchronized waves — spreading load across time instead of concentrating it at predictable intervals.

---

## Why it Exists

[Exponential backoff](../distributed-systems/exponential-backoff.md) is a significant improvement over fixed delays. Doubling the wait time after each failure gives a struggling server progressively more room to recover.

But there's a subtle problem. Imagine 10,000 mobile clients all receive a `503` error from the same server at the same moment — say, during a database failover. With exponential backoff (base = 1s):

- All 10,000 clients retry at t = 1s → server gets slammed
- All retry again at t = 3s (1+2s) → slammed again
- All retry again at t = 7s (1+2+4s) → slammed again

The waves are further apart, but they're still synchronized. Every client computed the exact same delay using the exact same formula. The server never gets a smooth trickle of traffic — it gets a series of hammers, just slower hammers.

This is the **thundering herd** problem. Jitter breaks synchronization by making each client's delay slightly different from every other client's.

---

## Intuition

Picture 500 people trying to get through a single revolving door at the same moment. Even if they all wait 10 seconds before trying again, they still arrive at the door together — the synchronization is the problem, not the wait time.

Now imagine each person waits a random amount between 5 and 15 seconds. Some try at 6 seconds, some at 9, some at 14. The door handles a steady flow instead of a synchronized surge.

Jitter is that randomness. It doesn't reduce the total number of retry attempts — it spreads them out over time so the server sees a manageable drip rather than a wave.

---

## Engineering Story

A notification service sends push alerts to 200,000 devices simultaneously — a flash sale announcement. Every device fires an API call to confirm delivery at the same instant. The confirmation endpoint briefly returns `429 Too Many Requests` due to the burst.

Without jitter, all 200,000 clients back off for exactly 1 second, then all retry together. The server gets the same 200,000-request wave one second later. It returns `429` again. The wave repeats.

With full jitter, each client picks a random delay between 0 and 1 second. Retries spread across an entire second. The server sees roughly 3,300 requests every 17 milliseconds instead of 200,000 at once. The endpoint handles it. Most clients succeed on the first retry.

---

## How it Works

The simplest form of jitter adds a random value on top of the exponential delay:

```
delay = exponential_delay + random(0, max_jitter)
```

But a better approach — **full jitter** — randomizes across the entire range from zero to the exponential delay:

```
delay = random(0, min(base * 2^attempt, maxDelay))
```

Full jitter is more effective because some clients retry very quickly (helping clear backlogged requests fast) while others wait longer (reducing the peak load). The distribution is flat, not clustered near the top of the range.

### Comparing the approaches

**No jitter — synchronized waves:**
```
Attempt 1: All clients wait exactly 1s
Attempt 2: All clients wait exactly 2s
Attempt 3: All clients wait exactly 4s
```

**Jitter added to backoff — spread but still partially clustered:**
```
Attempt 1: Clients wait 1.0s to 1.5s
Attempt 2: Clients wait 2.0s to 2.5s
Attempt 3: Clients wait 4.0s to 4.5s
```

**Full jitter — maximally spread:**
```
Attempt 1: Clients wait anywhere from 0s to 1s
Attempt 2: Clients wait anywhere from 0s to 2s
Attempt 3: Clients wait anywhere from 0s to 4s
```

---

## Diagram

```mermaid
graph LR
    A["Request fails"] --> B["Calculate exponential delay\ndelay = base × 2ⁿ"]
    B --> C["Apply full jitter\ndelay = random(0, delay)"]
    C --> D["Wait for jittered delay"]
    D --> E["Retry request"]
```

---

## Code Example

```javascript
async function fetchWithJitter(url, options = {}) {
  const MAX_RETRIES = 5;
  const BASE_DELAY_MS = 1000;   // 1 second
  const MAX_DELAY_MS = 30_000;  // 30 second cap
  const RETRYABLE_CODES = new Set([429, 500, 502, 503, 504]);

  for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
    try {
      const response = await fetch(url, options);

      if (!RETRYABLE_CODES.has(response.status)) {
        return response;
      }

      if (attempt === MAX_RETRIES - 1) {
        throw new Error(`Failed after ${MAX_RETRIES} attempts: ${response.status}`);
      }
    } catch (err) {
      if (attempt === MAX_RETRIES - 1) throw err;
    }

    // Full jitter: random delay between 0 and the capped exponential delay
    const exponentialDelay = Math.min(BASE_DELAY_MS * Math.pow(2, attempt), MAX_DELAY_MS);
    const jitteredDelay = Math.random() * exponentialDelay;

    await sleep(jitteredDelay);
  }
}

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
```

The one-line change from exponential backoff: `Math.random() * exponentialDelay` instead of `exponentialDelay`. That single multiplication desynchronizes every client in the system.

---

## Advantages

- **Eliminates thundering herd.** Spreading retries across a time window turns a synchronized wave into a smooth flow the server can actually handle.
- **Practically free to implement.** One call to `Math.random()` is all it takes to add jitter to an existing backoff implementation.
- **Reduces retry collisions.** Even when clients share the same error and the same backoff formula, jitter makes each client's timing unique.

---

## Limitations

- **Increases average latency slightly.** Full jitter randomly delays some clients longer than exponential backoff alone would. A client that would've retried at 1s might wait 0.8s or 0.1s — but on average, full jitter introduces the same total delay as half the exponential interval.
- **Doesn't fix the underlying problem.** Jitter buys time for a recovering server, but it doesn't stop retries from piling up if the outage is long. A [circuit breaker](../distributed-systems/circuit-breaker.md) is still needed for sustained failures.
- **Hard to reason about timing in tests.** Randomness makes retry behavior non-deterministic, which can complicate unit tests. Use seeded random functions or mock `Math.random` in tests.

---

## Common Mistakes

- **Adding jitter only to the first retry.** Jitter needs to be applied to every attempt, not just the first. Clients synchronized on attempt 1 will re-synchronize on attempt 2 if only the first delay is randomized.
- **Using a jitter range that's too narrow.** Adding `random(0, 100ms)` to a 4-second delay barely spreads anything. The jitter range should be proportional to the delay — full jitter (0 to full delay) works best.
- **Forgetting jitter entirely when switching to exponential backoff.** A common mistake is to implement exponential backoff as an improvement over fixed delays and stop there, not realizing that thundering herd still applies to synchronized exponential waits.
- **Applying jitter without a max delay cap.** Jitter and exponential backoff both require a cap — without it, delays grow unboundedly across many retries.

---

## Related Concepts

- **[Exponential Backoff](../distributed-systems/exponential-backoff.md)** — Jitter is applied on top of exponential backoff. The two are almost always used together.
- **[Retry](../distributed-systems/retry.md)** — The broader pattern. Jitter is a timing detail within a retry strategy.
- **[Circuit Breaker](../distributed-systems/circuit-breaker.md)** — Jitter helps during recoverable spikes; a circuit breaker stops retrying entirely during sustained outages.
- **Thundering Herd** — The problem jitter solves: many clients retrying simultaneously and overwhelming a server they're all trying to reach.

---

## Interview Questions

- **What is the thundering herd problem, and how does jitter solve it?** *(Tests understanding of the core motivation — synchronized retries create load spikes; jitter spreads them across time.)*
- **What's the difference between "jitter added to backoff" and "full jitter"?** *(Tests depth — added jitter still clusters near the top of the range; full jitter randomizes the entire interval, giving a flatter distribution.)*
- **Why isn't exponential backoff alone enough to prevent thundering herd?** *(Tests precision — exponential backoff spaces waves further apart but clients still synchronize on the same delays since the formula is deterministic.)*

---

## TLDR

- Jitter adds randomness to retry delays so clients don't all retry at the exact same moment.
- Without it, exponential backoff still creates synchronized retry waves — the thundering herd problem.
- **Full jitter** (`random(0, exponential_delay)`) gives the best load distribution.
- One `Math.random()` multiplication is all it takes to add to an existing backoff implementation.
- Always pair jitter with a max delay cap, and with a circuit breaker for long outages.
