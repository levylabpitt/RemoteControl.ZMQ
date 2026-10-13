# Issue #1 - Client timeout returns an empty reply with no error (change spec)

**Status.** Implemented on `develop` on 2026-10-13. Covered by the Caraya suite `tests/unit tests/Run All Tests.vi` (7 tests, all passing). This document records what was found and what changed.

**Problem.** When a client request got no reply after all retries, `Read Message.vi` returned `Command = ""`, an empty `Data` and **no error**. `Send and Receive Message.vi` (RemoteControl) passed that up as a successful reply, and the caller failed later and somewhere else. In the field this showed up as error 91 in the Lockin API's `getStatus.vi`, from Transport `Lockin_sweep`.

Working on it turned up three more defects in the same client retry path and one in connection cleanup. All five are fixed here. Paths are relative to `src/`.

## What was wrong (1.3.7.30)

1. **The timeout error was swallowed.** In `RemoteControl.ZMQ/Methods (Overrides)/Read Message.vi`, when the retries ran out, the client's `Default` frame created error **123** ("LabVIEW: A timeout occurred.") and passed it to `private/ZMQ.Handle Error.vi`. That VI had an `"Error 123"` frame with an unwired `error out`, which turned it into no error. The frame was added in 8deca5b (2026-08-06) and shipped in 1.3.7.30, which is when the field reports started. Nothing else in the library or the zeromq addon raises 123.
2. **A retry sent `HELP`, not the original command.** The retry frame (`0..5`) ran `ZMQ.Reconnect` -> `Read Message (sent)` -> `Send Message`. The reconnect calls the base `Close Connection.vi`, which resets both `Message--Cluster` fields of the class, so the stored request was already empty when it was read. `Send Message.vi` turns an empty command into `HELP`. The server answered `HELP`, and the caller got that reply as if it were the answer to its own command.
3. **A retry replaced the caller's timeout with 500 ms.** `Open Client Connection.vi`, called by the reconnect, writes a fixed 500 ms timeout.
4. **The stored request was already encoded.** `Send Message.vi` stored the JSON request string as the sent message's `Data`, so a correct re-send would have encoded it twice.
5. **`Close Connection.vi` leaked sockets and released contexts it did not own.** With an error on its input, it skipped `zmq_close` (the zeromq wrappers skip their DLL call on an input error but still delete their handle), yet still called `Socket Closed` and `Release Context`. It also released when its socket was never open. Once fix 1 let error 123 reach callers, a client that timed out leaked its socket. When the last other holder then closed cleanly, `zmq_ctx_destroy` waited forever for the leaked socket, and LabVIEW hung (Windows logged an AppHang on 2026-10-13).

## What changed

| VI | Change |
| --- | --- |
| `RemoteControl.ZMQ/private/ZMQ.Handle Error.vi` | Deleted the `"Error 123"` frame. 123 now passes through like any other error. |
| `RemoteControl.ZMQ/Methods (Overrides)/Read Message.vi`, client `Default` frame | Calls `ZMQ.Reconnect` before returning, so the next request starts on a fresh REQ socket. Returns the Error Ring's 123, merged first with the reconnect error. The code stays 123, as before 1.3.7.30; 56 was considered and dropped to avoid surprising callers that check for 123. |
| `RemoteControl.ZMQ/Methods (Overrides)/Read Message.vi`, client `0..5` frame | New order: `Read timeout` -> `Read Message (sent)` -> `ZMQ.Reconnect` -> `Write Timeout` (saved value) -> `Send Message` (saved message). |
| `RemoteControl.ZMQ/Methods (Overrides)/Send Message.vi` | The message stored with `Write Message (sent)` takes `Data` from the input message, not the encoded string. Only the ZMQ and STM `Read Message.vi` read it back. |
| `RemoteControl.ZMQ/Methods (Overrides)/Close Connection.vi` | `zmq_setsockopt` (LINGER 0) and `zmq_close` run with a cleared error. `Socket Closed` and `Release Context` run only in the frame where a valid socket was closed. `error in` is merged back first. |
| `ZMQ.ContextManager/private/ContextManager.FGV.vi`, `Release Context` | If the counter is already 0, the frame passes through unchanged: no U32 wrap, no destroy. |

## Tests

`tests/unit tests/`, members of `tests/RemoteControl.ZMQ.Tests.lvclass`. That class is a child of `RemoteControl.ZMQ.lvclass` and lives outside `src/`, so neither the tests nor Caraya ship in the package. As a child it can call the protected overrides. `ZMQ.Handle Error.vi` is community scope with the test class as friend, for `Test Handle Error Passes Timeout.vi`. Run the suite from the project, not by opening the VI by path through COM. Each test uses its own port (55501 to 55506). `Test Helper - Echo Server.vi` runs a REP server in parallel with the client. It takes a reply delay, a first-reply-only option and an idle exit based on time since the last message.

| Test | Asserts |
| --- | --- |
| `Test Round Trip.vi` | No error; the data comes back unchanged; the server received exactly one request. |
| `Test Retry Resends Original Command.vi` | First reply delayed 300 ms, client timeout 100 ms. The client gets its data back; the server received 2 or more requests (a retry happened); none was `HELP`. |
| `Test Client Timeout Returns Error.vi` | No server, timeout 100 ms: error 123. |
| `Test Client Recovers After Timeout.vi` | Same client object: the first call returns 123, then the server starts and the second call succeeds. |
| `Test Server Idle is not an Error.vi` | A server with no client: no error, no messages. |
| `Test Server Reopen Same Port.vi` | Open, close and reopen a server on the same port without error. |
| `Test Handle Error Passes Timeout.vi` | Error 123 into `ZMQ.Handle Error.vi` comes out as 123. |

## Effect on callers

- `Instrument.LockinAPI:getStatus.vi` against an unresponsive lock-in now returns error 123 after about `7 x timeout`, instead of error 91.
- A slow reply is now retried with the original command and the caller's timeout, instead of being answered with the `HELP` reply.

Callers that show errors in a modal dialog still stop there: see levylabpitt/Transport#180.

## Later, not in this round

- **Re-sending runs a command again.** A retry now re-sends the real request, so a slow server can run a non-idempotent command (`startSweep`, `startDAQ`, set commands) more than once. Options: retry only commands marked idempotent, or have the server drop request IDs it has already seen.
- **`Send and Receive Message.vi` in levylabpitt/RemoteControl** turns a reply whose `Message.error` is set into `Command = "error"`, `Data = <error cluster>`, and leaves `error out` clean. ZMQ never sets `Message.error`, but other transports would hide server errors the same way.
- **The installed lock-in exe** has RemoteControl.ZMQ built in. Its server side uses `Close Connection.vi` and `ZMQ.Handle Error.vi` too, so rebuild it once this package is released.

Related: levylabpitt/Instrument-Framework#200, levylabpitt/Multichannel-Lockin#183, levylabpitt/Transport#180.
