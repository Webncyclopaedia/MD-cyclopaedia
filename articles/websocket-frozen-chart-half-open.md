[⬅ Back to articles](../README.md)


# The chart froze while the socket stayed OPEN: half-open connections and the heartbeat nobody wrote

> A real-world case: a trading dashboard (Vite + React + a candlestick chart) streaming live trades over WebSocket. Desktop, office network, behind a load balancer. Users reported "the chart is frozen" — the last candle stops moving, the price in the header goes stale, but the page is alive: buttons work, navigation works, no error toast, no reconnect. Refreshing fixes it instantly. It happens after a few minutes of sitting on the page, and always on the same floor of the building.

## Symptom

Three properties made this one expensive to diagnose:

1.  **There is no error.** No `onerror`, no `onclose`, no console message. `ws.readyState` is `1` — `OPEN`. The app believes it is connected.
2.  **It's a freeze, not a disconnect.** A real disconnect triggers the reconnect path, and the chart recovers in under a second. Here nothing triggers, so nothing recovers.
3.  **A refresh always fixes it**, which sends everyone hunting for a React bug: a stale closure, a memoised component, a chart library that stopped rendering. All red herrings — the data never arrived.

The tell that eventually cracked it: the freeze always came after roughly a minute of **inactivity**. Active traders rarely saw it. People who left the tab open and came back to it saw it every time.

## When it appears

The feed hook was the obvious, reasonable implementation:

```tsx
useEffect(() => {
  const ws = new WebSocket(FEED_URL);

  ws.onopen = () => ws.send(JSON.stringify({ type: "subscribe", channel: "trades" }));

  ws.onmessage = (event) => {
    const trade = JSON.parse(event.data);
    applyTrade(trade); // updates the candle series
  };

  ws.onclose = () => reconnect();
  // no onerror, no heartbeat, no watchdog

  return () => ws.close();
}, []);
```

The trigger combination:

1.  **An idle WebSocket** with no application-level traffic;
2.  **Something in the middle that drops idle connections** — a load balancer, a proxy, a NAT, a corporate firewall;
3.  **A client that only reacts to `onclose`**;
4.  **No heartbeat**, so the client never notices the peer is gone.

## The mechanics of the bug

### What actually happens on the wire

A WebSocket is a TCP connection with a handshake. TCP has no notion of "is the other side still there" — it only notices when it tries to send and gets no ACK, which can take minutes, or when it receives a FIN/RST. If the peer disappears **without closing** — the load balancer expired an idle flow, the office NAT dropped its mapping, the server was replaced — nobody sends anything. The connection is now **half-open**: locally it's a perfectly valid socket, remotely there is nothing on the other end.

### Why the browser can't tell you

- **`readyState === OPEN` means "the handshake completed and `close()` hasn't been called"** — not "the peer is alive". It is a local flag, not a liveness check.
- **`onclose` never fires.** There is no FIN to receive. It may fire minutes later, when the OS TCP stack gives up on retransmissions — or never, if no data is being sent at all.
- **`ws.send()` does not throw.** It hands bytes to the kernel, which buffers them and retransmits. The API has no idea whether they will ever be delivered. "The send succeeded" is not evidence of anything.
- **`bufferedAmount` grows**, but only if you're actually sending. An idle client sends nothing, so there's nothing to grow.

This is the trap: the client is written against a model where "connection" is a state, but on the wire it's a *process* that must be continuously verified.

### Why the timeouts are always around a minute

Idle-connection timeouts are a load-balancer and proxy feature, not a WebSocket one:

- nginx `proxy_read_timeout` defaults to 60s;
- cloud load balancers commonly default to 60s (some allow up to a few minutes, some 100s);
- corporate firewalls and NAT devices drop idle flows on their own schedule.

The rule of thumb: **if your connection can be idle for longer than 60 seconds, assume something will kill it silently.** Market data feeds are exactly the workload that sits idle during quiet periods — the busier the market, the healthier the connection, which is why the bug looked "random".

### Why dev never sees it

`localhost` has no load balancer, no NAT, no idle timeout, and the developer is usually interacting with the page, which keeps traffic flowing. The bug is created by the deployment topology, not by the code.

## How we found it

The decisive measurement is not "is it connected" but "when did we last hear from it":

```js
// in the browser console, on a frozen tab
ws.readyState;          // 1  ← "OPEN"
ws.bufferedAmount;      // 0
lastMessageAt;          // 184_000 ms ago ← four minutes of silence
```

Then the control experiment that closed it: leave the tab untouched for 90 seconds, then run

```js
ws.send(JSON.stringify({ type: "ping" })); // no error
ws.readyState;                              // still 1
```

and wait. No reply, no close, no error. The socket was a corpse that hadn't been told. Restarting the network stack (or the refresh button) was the only thing that changed anything.

Two more confirmations: the same page on a connection with no proxy in between never froze, and the server logs showed the session ending at exactly 60 seconds after the last client frame.

## The fix

A connection needs **two** independent mechanisms, and most codebases have zero:

1. **Heartbeat** — regular application-level traffic, so the connection is never idle long enough to be reaped, and so silence becomes detectable.
2. **Watchdog** — a timer that force-closes the socket when nothing has been heard for too long. Closing it locally fires `onclose`, which is the event your reconnect logic already listens to.

```ts
const HEARTBEAT_MS = 20_000; // below the shortest idle timeout you expect
const DEAD_AFTER_MS = 55_000; // how long silence is tolerated

function useMarketFeed(url: string, onTrades: (trades: Trade[]) => void) {
  const lastSeen = useRef(Date.now());
  const lastSeq = useRef(0);
  const attempt = useRef(0);
  const socket = useRef<WebSocket | null>(null);

  useEffect(() => {
    let disposed = false;
    let heartbeat: number | undefined;

    const connect = () => {
      const ws = new WebSocket(url);
      socket.current = ws;

      ws.onopen = () => {
        attempt.current = 0;
        lastSeen.current = Date.now();

        // resubscribe from the last applied sequence, not from scratch
        ws.send(JSON.stringify({ type: "subscribe", channel: "trades", since: lastSeq.current }));

        heartbeat = window.setInterval(() => {
          if (Date.now() - lastSeen.current > DEAD_AFTER_MS) {
            ws.close(); // force onclose → reconnect path
            return;
          }
          ws.send(JSON.stringify({ type: "ping", t: Date.now() }));
        }, HEARTBEAT_MS);
      };

      ws.onmessage = (event) => {
        lastSeen.current = Date.now(); // ANY frame counts as liveness
        const msg = JSON.parse(event.data);
        if (msg.type === "pong") return;
        if (msg.seq <= lastSeq.current) return; // duplicate replay after reconnect
        if (msg.seq !== lastSeq.current + 1) {
          void resyncSnapshot(); // gap: refill from REST, don't render a hole
          return;
        }
        lastSeq.current = msg.seq;
        onTrades(msg.trades);
      };

      ws.onclose = () => {
        window.clearInterval(heartbeat);
        if (disposed) return;
        // exponential backoff with jitter: no thundering herd after a restart
        const backoff = Math.min(30_000, 500 * 2 ** attempt.current);
        const delay = backoff * (0.5 + Math.random());
        attempt.current += 1;
        window.setTimeout(connect, delay);
      };
    };

    connect();

    // timers are throttled in background tabs — probe when the tab comes back
    const onVisible = () => {
      if (document.visibilityState === "visible" && Date.now() - lastSeen.current > DEAD_AFTER_MS) {
        socket.current?.close();
      }
    };
    document.addEventListener("visibilitychange", onVisible);

    return () => {
      disposed = true;
      document.removeEventListener("visibilitychange", onVisible);
      window.clearInterval(heartbeat);
      socket.current?.close();
    };
  }, [url]);
}
```

Notes on the pieces that matter:

- **Any frame resets the watchdog**, including the pong. That's why `lastSeen` is updated before the message type is inspected.
- **`HEARTBEAT_MS` must be shorter than the shortest idle timeout in the path.** 20s is safe against 60s balancers; 45s is not.
- **The browser cannot send ping frames.** There's no `ws.ping()` in the WebSocket API. Servers *can* send RFC 6455 pings, and browsers answer them automatically — but the browser doesn't expose that exchange to JavaScript, so a server-side ping protects the server's view of the client, not your chart. You need application-level messages.
- **Timers are throttled in background tabs**, so the watchdog may fire late. The `visibilitychange` probe covers the "user came back to a dead tab" case, which is exactly the reported scenario.
- **Close, don't reconnect directly from the watchdog.** Let `onclose` own the reconnect path, so there's exactly one place that schedules retries.

### The chart half: bucket by event time, not arrival time

Once the data flows again, the second bug appears. The naive candle builder uses the moment of arrival:

```ts
const bucket = Math.floor(Date.now() / 60_000) * 60_000; // ← wrong
```

Under jitter, a trade that happened at `10:00:59.980` arrives at `10:01:00.150` and is counted in the **next** minute's candle. The chart disagrees with the exchange by one candle's volume — and it disagrees *differently* on every reload, which makes it look like a rendering bug.

```ts
const bucket = Math.floor(trade.ts / 60_000) * 60_000; // ← the exchange's clock
```

And when a late trade lands in a candle that's already drawn, that historical candle must be re-rendered — not just the last one. Most chart libraries have an `update` call for the last bar and a `setData`/`applyNewData` for history; use the historical path when `trade.ts` is older than the current bucket.

A stale indicator is cheap insurance: if `Date.now() - lastSeen > 5_000`, dim the series and show "delayed". The user stops trusting a frozen chart the moment it admits it's frozen.

## Neighbouring traps in the same feed

- **Duplicate replay after reconnect.** Resubscribing without `since` sends the last N trades again, and an accumulator that adds volumes double-counts them. Dedupe by trade id or sequence number — never by timestamp.
- **A gap rendered as "no trades".** If the server skips `seq` 41–60, the chart shows a quiet period instead of missing data. Compare sequence numbers on every message and refill from a REST snapshot.
- **Thundering herd.** Every client reconnecting with the same fixed 5s delay after a deploy hammers the gateway. Exponential backoff plus jitter, always.
- **Backpressure.** If the feed sends 5,000 messages/s, `setState` per message re-renders the chart 5,000 times. Batch into `requestAnimationFrame`, and watch `ws.bufferedAmount` on the send side.
- **`JSON.parse` on the main thread** at that rate blocks the UI. Binary frames (`arraybuffer`) or a worker are the real answer; a 16ms frame budget disappears fast.
- **`onerror` is nearly useless.** It fires for handshake failures and protocol violations; a dead peer produces nothing. Don't build liveness on it.

---

## Takeaways (checklist)

1.  **`readyState === OPEN` is a local flag, not proof of life.** A half-open TCP connection looks open forever and never fires `onclose`.
2.  **Idle connections get reaped silently** — 60s is the common default for proxies and load balancers. Any feed that can be quiet for a minute needs a heartbeat.
3.  **`ws.send()` succeeding means nothing.** Bytes go into the kernel buffer; there is no delivery feedback in the API.
4.  **Heartbeat + watchdog, both.** Traffic keeps the connection alive; the watchdog turns silence into an `onclose` your reconnect logic already handles.
5.  **The browser can't send ping frames.** Server-side pings protect the server, not your UI — use application-level `ping`/`pong` messages.
6.  **Background tabs throttle timers**, so probe on `visibilitychange` instead of trusting the interval to fire on time.
7.  **Bucket candles by the event timestamp, never by arrival time**, and re-render historical candles when a late trade lands in them.


[⬅ Back to articles](../README.md)
