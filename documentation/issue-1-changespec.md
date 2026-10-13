# Issue #1 - Client timeout returns an empty reply with no error (change spec)

**Problem.** When a client request gets no reply after all retries, `Read Message.vi` returns `Command = ""`, an empty `Data` and **no error**. `Send and Receive Message.vi` (RemoteControl) passes that up as a successful reply, and the caller fails later and somewhere else. In the field this shows up as error 91 in the Lockin API's `getStatus.vi`, from Transport `Lockin_sweep`.

**Scope.** The ZMQ client timeout path only. The wire format and the server (REP) path do not change. The locations below were read with lvkit on `develop` at 1.3.7.30, which matches the copy installed in `user.lib\LevyLab\RemoteControl.ZMQ`. The two key frames were confirmed with LabVIEW 2019 renders on the same day. Paths are relative to `src/RemoteControl.ZMQ/`.

## Current behaviour

`Methods (Overrides)/Read Message.vi`, While loop, case on `zmq_recv_timeout` "timed out OR error":

- **False** (reply received): `Parse Response` (client) or `Parse Request` (server), then stop.
- **True**, inner case on `endpoint`:
  - `ZMQ_REP, ZMQ_SERVER`: pass the receive error through and stop.
  - `ZMQ_REQ, ZMQ_CLIENT`, case on the loop counter `i`:
    - `0..5`: `ZMQ.Reconnect` -> `Read Message (sent)` -> `Send Message` (re-send), and do not stop.
    - `Default`: an Error Ring creates error **123** ("LabVIEW: A timeout occurred."), and stop.
  - The case output error then goes through `ZMQ.Handle Error.vi`.

`private/ZMQ.Handle Error.vi`: after `zmq_check_term`, a case on the error code:
- `"Error 123"`: the `error out` tunnel is unwired, so it outputs **no error**.
- `"Error 1556"`: `ZMQ.Reconnect`.
- `Default` and `No Error`: pass the error through.

So the timeout error made in the `Default` frame is cleared one node later.

## Edits

### 0. Check (no edit)

Find out why `ZMQ.Handle Error.vi` clears 123 at all; `Send Message.vi` calls it too. Search its callers (`Read Message.vi`, `Send Message.vi`, and anything in `ZMQ.ContextManager` / `ZMQ.SocketMonitor`). The comment "timeout error was cleared in Receive with Timeout" in `Read Message.vi` suggests that `zmq_recv_timeout` already clears its own timeout. The 123 frame may then exist only for this Error Ring, or for a server idle timeout.

### 1. `Methods (Overrides)/Read Message.vi`: client timeout error must survive

In the `ZMQ_REQ, ZMQ_CLIENT` -> `Default` frame:

1. Change the Error Ring to code **56** ("The network operation exceeded the user-specified or system time limit"), keeping a message that names the command and the timeout, e.g. `RemoteControl.ZMQ: no reply after 7 attempts`. Code 56 is what LabVIEW's own TCP/network VIs return for the same thing, and callers can already recognise it.
2. If edit 2 does not remove the 123 frame, wire this error around `ZMQ.Handle Error.vi`: merge it with that VI's `error out` (Merge Errors, ring error first) rather than passing it through.

### 2. `private/ZMQ.Handle Error.vi`: do not clear errors that callers need

Depending on what step 0 found:

- If nothing else depends on the `"Error 123"` frame, delete it. 123 then falls into the default (pass-through) frame.
- If something does depend on it (for example, a server idle timeout), keep the frame but make sure the client path never sends 123 here (edit 1.1).

### 3. `Methods (Overrides)/Read Message.vi`: never return an empty message without an error

As a guard, after the loop: if `error out` has no error and the loop's last iteration took the timeout branch, set error 56. With edit 1 this should never fire, but it closes the "empty data, no error" outcome for good.

## Tests

Use `Tests/RemoteControl.ZMQ.TestServer.vi` with a delay before it replies, and `Tests/RemoteControl.ZMQ.TestClient.vi`.

1. Reply delay 0: the reply arrives and there is no error (regression check).
2. Reply delay 150 ms with client timeout 100 ms: one retry, then the reply arrives with no error. Check that the server log shows the command **twice**; this is the known re-send behaviour, see "Later".
3. No reply at all (server stopped): the client returns **error 56** after about `7 x timeout`, with `Command` and `Data` empty. Before the fix it returns no error.
4. Server comes back after test 3: the next request succeeds (reconnect works).
5. Through the Lockin API against a stopped Multichannel Lockin: `getStatus.vi` returns error 56, not 91.

## Later, not in this round

- **Re-sending runs commands twice.** The lazy-pirate retry re-sends the request. For commands that are not idempotent (`startSweep`, `startDAQ`, set commands), a slow server runs them again, up to 7 times. Options: retry only commands marked idempotent, or have the server drop request IDs it has already seen. The second needs the server to cache recent IDs.
- **`Send and Receive Message.vi` in levylabpitt/RemoteControl** turns a reply whose `Message.error` is set into `Command = "error"`, `Data = <error cluster>`, and leaves `error out` clean. ZMQ never sets `Message.error`, so it does not apply here, but other transports would hide server errors the same way.

Related: levylabpitt/Instrument-Framework#200, levylabpitt/Multichannel-Lockin#183, levylabpitt/Transport#180.
