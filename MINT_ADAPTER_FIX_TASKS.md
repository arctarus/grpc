# Mint Adapter — Fix Task List

> Work through each issue in order. Every section follows the same three-step
> pattern: **implement → test → document**. Mark each checkbox as you go.
>
> Related analysis: [`MINT_ADAPTER_PERFORMANCE_ANALYSIS.md`](./MINT_ADAPTER_PERFORMANCE_ANALYSIS.md)

---

## Table of Contents

| # | Severity | Issue | Status |
|---|----------|-------|--------|
| [1](#issue-1-buffer-not-fully-drained-on-multi-message-frames) | 🔴 Critical | Buffer not fully drained on multi-message frames | ✅ Merged |
| [3](#issue-3-binary-buffer-concatenation-allocates-a-new-copy-per-chunk) | 🟡 Moderate | Binary buffer concatenation allocates a new copy per chunk | ✅ Done |
| [4](#issue-4-decode_headers-called-twice-for-the-same-headers) | 🟡 Moderate | `decode_headers` called twice for the same headers | ✅ Done |
| [5](#issue-5-ioiodata_length-recomputed-on-every-queue-tick) | 🟡 Moderate | `IO.iodata_length` recomputed on every queue tick | ⬜ Open |
| [6](#issue-6-processflagtrap_exit-true-mutates-the-calling-process) | 🔵 Minor | `Process.flag(:trap_exit, true)` permanently mutates the calling process — should save and restore | ✅ Done |
| [7](#issue-7-dead-code-stateappend_response_data3-is-never-called) | 🔵 Minor | Dead code: `State.append_response_data/3` is never called | ✅ Done |
| [8](#issue-8-multiple-linear-scans-over-the-same-keyword-list) | 🔵 Minor | Multiple linear scans over the same keyword list | ⬜ Open |

---

## Definition of Done

Before closing any issue:

- [ ] All checkboxes in the issue's section are ticked.
- [ ] `mix test` passes with no failures and no new warnings.
- [ ] The changed module's `@moduledoc` / `@doc` reflects the new behaviour.
- [ ] A brief entry is added to `CHANGELOG.md` under an `## Unreleased` heading.

---

## Issue 1 — Buffer not fully drained on multi-message frames

**Severity:** 🔴 Critical  
**Status:** ✅ Merged to master  
**File:** `grpc_client/lib/grpc/client/adapters/mint/stream_response_process.ex`

`GRPC.Message.get_message/2` is called once per incoming data chunk. If a single
HTTP/2 data frame carries multiple complete gRPC messages, only the first one is
decoded; the rest sit in the buffer until the next frame arrives, stalling consumers.

All implementation, test, and documentation tasks were completed and merged to master.
See the `CHANGELOG.md` and the `# Buffer draining` section in `StreamResponseProcess`
`@moduledoc` for details.

---

## Suggested Implementation Order

While each issue can be worked on independently on its own branch, the
following order minimises merge conflicts and logical dependencies:

```
Issue 7  →  Issue 6  →  Issue 3  →  Issue 1  →  Issue 4  →  Issue 5  →  Issue 8
(delete)    (save/       (fix         (buffer     (decode     (queue      (map
             restore     concat)      draining)   once)       size)       scan)
             flag)
```

- **Issues 7 and 6** have no inter-dependencies — start here to reduce noise
  in later diffs.
- **Issue 3** should land before **Issue 1** because the buffer-drain loop
  builds on the single-binding pattern.
- **Issue 4** is independent of 1 and 3 but touches the same handler, so
  sequencing it after keeps diffs small.
- **Issue 2** (`GenServer.call` → `cast`) has been applied but carries a
  back-pressure trade-off documented in `MINT_ADAPTER_PERFORMANCE_ANALYSIS.md`.
  It is excluded from this order until a proper back-pressure strategy is decided.

## Issue 3 — Binary buffer concatenation allocates a new copy per chunk

**Severity:** 🟡 Moderate  
**File:** `grpc_client/lib/grpc/client/adapters/mint/stream_response_process.ex`

`buffer <> data` creates a brand-new binary on every incoming data chunk. In the
partial-message branch this concatenation is also performed twice — once as the
argument to `GRPC.Message.get_message` and once when storing the updated buffer.

---

### 3.1 Implementation

- [x] At the very top of `handle_cast({:consume_response, {:data, data}}, ...)` 
  (after Issue 2 lands) or the equivalent `handle_call`, bind the result once:
  `combined = buffer <> data`.

- [x] Pass `combined` to `drain_buffer/5` (after Issue 1 lands) or to
  `GRPC.Message.get_message/2` directly, and use `combined` as the initial
  buffer value in the no-match branch — eliminating the second concatenation.

- [ ] As a follow-up improvement, consider accumulating incoming chunks as an
  iolist in the `buffer` field and calling `IO.iodata_to_binary/1` only at the
  `get_message` boundary. Gate this behind a separate sub-task (deferred):

  - [ ] Change `buffer` initial value in `init/1` from `<<>>` to `[]`.
  - [ ] In the data handler, prepend new data: `combined = [buffer | [data]]`
    and pass `IO.iodata_to_binary(combined)` to `get_message`.
  - [ ] Store the remaining binary (`rest`) directly back into `buffer` since
    `get_message` already returns a proper binary tail.

---

### 3.2 Tests

> Target file: `grpc_client/test/grpc/adapters/mint/stream_response_process_test.exs`

- [x] **Regression** — run the full `"handle_call/3 - data"` describe block and
  confirm all existing tests still pass.

- [x] **New test** — `"buffer holds the concatenation of prior buffer and new
  data when combined is still incomplete"`: set `state.buffer` to a non-empty
  binary (a 5-byte gRPC framing prefix), send a 2-byte chunk that still does
  not complete the message, and assert that
  `new_state.buffer == prior_buffer <> new_chunk` and responses queue is empty
  (i.e. data is not lost and no duplication occurs).

- [ ] If the iolist optimisation sub-task is implemented, add a test verifying
  that `state.buffer` starts as `[]`, correctly accumulates partial data, and
  yields the right decoded message once a completing chunk arrives.

---

### 3.3 Documentation

- [x] Add a one-line inline comment next to the `combined = buffer <> data`
  binding explaining that the concatenation is performed once to avoid
  allocating duplicate binaries.

- [ ] If the iolist optimisation is done, document the `buffer` field in the
  `@moduledoc` (or in an `@typep` for the state map) noting that it is an
  iolist during accumulation and a binary after a partial message is retained.

- [ ] Add a `CHANGELOG.md` entry:
  ```
  ### Changed
  - `StreamResponseProcess`: eliminated duplicate binary concatenation in the
    data handler; buffer concatenation is now computed once per incoming chunk.
  ```

---

## Issue 4 — `decode_headers` called twice for the same headers

**Severity:** 🟡 Moderate  
**File:** `grpc_client/lib/grpc/client/adapters/mint/stream_response_process.ex`

Both `update_compressor/2` and `get_headers_response/2` independently call
`GRPC.Transport.HTTP2.decode_headers/1` on the identical raw header list, doing
redundant work on every `:headers` or `:trailers` event.

---

### 4.1 Implementation

- [x] In the two `handle_cast` (or `handle_call`) clauses that match
  `{type, headers}` when `type in @header_types`:
  - Decode the headers once at the top of the clause:
    `decoded = GRPC.Transport.HTTP2.decode_headers(headers)`
  - Pass `decoded` into both `update_compressor` and `get_headers_response`.

- [x] Update the signature of `update_compressor/2` to accept pre-decoded
  headers: `update_compressor({type, decoded_headers}, state)` — or introduce
  a new private `update_compressor_decoded/2` to keep the change minimal.

- [x] Update the signature of `get_headers_response/2` similarly so it receives
  the already-decoded map and skips the `decode_headers` call internally.

- [x] Remove the `GRPC.Transport.HTTP2.decode_headers/1` call from inside both
  `update_compressor/2` and `get_headers_response/2`.

---

### 4.2 Tests

> Target file: `grpc_client/test/grpc/adapters/mint/stream_response_process_test.exs`

- [x] **Regression** — run the full `"handle_call/3 - headers/trailers"` describe
  block and confirm all tests still pass, especially:
  - `"put error in responses when incoming headers has error status"` (all
    parameter combinations).
  - `"add compressor to state when incoming headers match available compressor"`.
  - `"don't update compressor when unsupported compressor is returned"`.

- [x] **New test** — `"compressor is set and headers are enqueued correctly from
  a single headers event"`: asserts both that the compressor is resolved
  correctly (`GRPC.Compressor.Gzip`) and that the enqueued response contains
  the decoded headers map — all from one `handle_call` invocation, proving
  the shared decoded value is used by both paths.

---

### 4.3 Documentation

- [x] Add an inline comment above the `decoded = ...` binding in each clause
  explaining that the result is shared between compressor resolution and
  response construction.

- [x] Update the `@doc false` (or inline comment) for `update_compressor/2` and
  `get_headers_response/2` to reflect their new signatures.

- [ ] Add a `CHANGELOG.md` entry:
  ```
  ### Changed
  - `StreamResponseProcess`: HTTP/2 headers are now decoded once per event
    instead of twice, removing a redundant map construction on every
    headers/trailers frame.
  ```

---

## Issue 5 — `IO.iodata_length` recomputed on every queue tick

**Severity:** 🟡 Moderate  
**File:** `grpc_client/lib/grpc/client/adapters/mint/connection_process/connection_process.ex`

`IO.iodata_length(body)` is called inside `handle_continue(:process_request_stream_queue)`
on every window-update cycle. For large or deeply nested iodata this is O(n) work
repeated every time the queue is ticked.

---

### 5.1 Implementation

- [ ] Change the queue entry tuple from `{request_ref, body, from}` to
  `{request_ref, body, byte_size, from}` where `byte_size` is the pre-computed
  byte length of `body`.

- [ ] In `handle_call({:request, method, path, headers, body, opts}, ...)`,
  compute `byte_size = IO.iodata_length(body)` before enqueueing and store it
  in the tuple: `:queue.in({request_ref, body, byte_size, nil}, ...)`.

- [ ] In `handle_cast({:stream_body, request_ref, body}, ...)` (i.e. the
  non-`:eof` streaming path), compute `byte_size = IO.iodata_length(body)` and
  store it in the enqueued tuple: `:queue.in({request_ref, body, byte_size, from}, ...)`.

- [ ] In `handle_continue(:process_request_stream_queue)`, destructure the
  four-element tuple and replace `IO.iodata_length(body) > window_size` with
  the pre-stored `byte_size > window_size`.

- [ ] In `chunk_body_and_enqueue_rest/2`, after splitting `body` into `head`
  and `tail`, compute `tail_size = byte_size(tail)` (tail is always a binary
  at this point) and re-enqueue as `{request_ref, tail, tail_size, from}`.

- [ ] Update `finish_all_pending_requests/1` to destructure the four-element
  tuple wherever it pattern-matches on queue entries.

---

### 5.2 Tests

> Target file: `grpc_client/test/grpc/adapters/mint/connection_process_test.exs`

- [ ] **Regression** — run the full `"handle_continue/2 - :process_stream_queue"`
  describe block and confirm all existing tests still pass.

- [ ] **New test** — `"stores byte size alongside body in queue entry"`: after
  calling `handle_call` with a non-stream body, inspect the queue in the
  returned state and assert the third element of the head entry equals the
  expected byte length.

- [ ] **New test** — `"re-enqueued chunk tail carries the correct byte size"`:
  set the window size to a value smaller than the body, trigger
  `handle_continue`, and assert the re-enqueued tail entry has an updated
  `byte_size` matching `byte_size(tail)`.

---

### 5.3 Documentation

- [ ] Add an inline comment in `handle_continue` next to the `byte_size`
  extraction explaining that the size is pre-computed to avoid O(n) traversal
  on every queue tick.

- [ ] Update the `@moduledoc` for `ConnectionProcess` to mention that queue
  entries carry a pre-computed body size for efficient window-size comparison.

- [ ] Add a `CHANGELOG.md` entry:
  ```
  ### Changed
  - `ConnectionProcess`: request body byte size is now computed once at
    enqueue time instead of being recomputed on every HTTP/2 window-update
    tick, reducing O(n) work during high-throughput streaming.
  ```

---

## Issue 6 — `Process.flag(:trap_exit, true)` permanently mutates the calling process's exit-trapping behaviour

**Severity:** 🔵 Minor  
**File:** `grpc_client/lib/grpc/client/adapters/mint.ex`

`Process.flag(:trap_exit, true)` is called inside `connect/2` but **never restored**.
The flag is legitimate — `connect/2` calls `ConnectionProcess.start_link/4`, which links
the new process to the caller, and without `trap_exit` any future crash of that linked
process would kill the caller via an asynchronous EXIT signal (the `catch :exit` block
only covers synchronous exits raised during `start_link` itself). However, leaving the
flag permanently `true` silently changes the caller's exit-trapping behaviour for the
rest of its lifetime, affecting all exits unrelated to gRPC.

The fix is to save the previous flag value before setting it and restore it after
`start_link` returns, so the caller's state is left exactly as it was found.

---

### 6.1 Implementation

- [x] In `connect/2`, capture the previous flag value before setting it:
  ```elixir
  previous_flag = Process.flag(:trap_exit, true)
  ```

- [x] After `ConnectionProcess.start_link/4` returns (but before the `case`
  result is processed), restore the flag:
  ```elixir
  Process.flag(:trap_exit, previous_flag)
  ```

- [x] Retain the existing `catch :exit, reason ->` clause as a safety net for
  any unexpected synchronous exits from `start_link`.

- [x] Confirm the two previously failing tests now pass:
  - `"connects insecurely (custom options)"`
  - `"accepts config_options for application specific configuration"`

---

### 6.2 Tests

> Target file: `grpc_client/test/grpc/adapters/mint_test.exs`

- [x] **New test** — `"connect/2 does not permanently alter the trap_exit flag
  of the calling process"`:
  1. Ensure the caller starts with `trap_exit: false`:
     `Process.flag(:trap_exit, false)`.
  2. Call `Mint.connect(channel, [])`.
  3. Assert the flag was restored:
     `assert Process.info(self(), :trap_exit) == {:trap_exit, false}`.

- [x] **Regression** — run the full `MintTest` suite and confirm all tests pass,
  including the error-path test `"connects insecurely (custom options)"` which
  previously crashed the test process with an unhandled EXIT signal.

---

### 6.3 Documentation

- [x] Add an inline comment in `connect/2` explaining why the flag is set and
  why it is restored:
  ```elixir
  # trap_exit must be true while start_link is in progress so that any
  # asynchronous EXIT from the linked ConnectionProcess does not kill the
  # caller. We restore the previous value immediately after start_link
  # returns to avoid permanently altering the caller's exit-trapping state.
  previous_flag = Process.flag(:trap_exit, true)
  ```

- [ ] Add a `CHANGELOG.md` entry:
  ```
  ### Fixed
  - `Mint.connect/2` now saves and restores the calling process's
    `trap_exit` flag instead of setting it permanently to `true`, preventing
    silent changes to the caller's exit-handling behaviour.
  ```

---

## Issue 7 — Dead code: `State.append_response_data/3` is never called

**Severity:** 🔵 Minor  
**File:** `grpc_client/lib/grpc/client/adapters/mint/connection_process/state.ex`

`append_response_data/3` was part of an older implementation that accumulated
response data directly in the `ConnectionProcess` state. It is now unreachable —
no call site exists anywhere in the codebase.

---

### 7.1 Implementation

- [x] Delete the `append_response_data/3` function from `state.ex`.

- [x] Run a project-wide grep for `append_response_data` to confirm no remaining
  call sites exist before deleting.

---

### 7.2 Tests

- [x] Confirm that no existing test references `State.append_response_data/3`
  (a grep before deleting is sufficient; no new tests are needed for a
  deletion).

- [x] Run the full test suite (`mix test`) and confirm it is green.

---

### 7.3 Documentation

- [x] No public API change — no `@doc` or `@moduledoc` update is required.

- [x] Add a `CHANGELOG.md` entry:
  ```
  ### Removed
  - `ConnectionProcess.State.append_response_data/3`: removed dead code left
    over from a previous implementation; response data is routed exclusively
    through `StreamResponseProcess`.
  ```

---

## Issue 8 — Multiple linear scans over the same keyword list

**Severity:** 🔵 Minor  
**File:** `grpc_client/lib/grpc/client/adapters/mint.ex`

In the unary / client-stream receive path, `check_for_error/1`,
`Keyword.fetch!/2`, and `get_headers_and_trailers/1` each perform independent
`O(n)` scans over the same `responses` keyword list.

---

### 8.1 Implementation

- [ ] In `do_receive_data/3` (the `:client_stream` / `:unary` clause), after
  collecting `responses = Enum.to_list(stream)`, convert it to a map once:
  `response_map = Map.new(responses)`.

- [ ] Rewrite `check_for_error/1` to accept a map and use `Map.get/3` instead
  of `Keyword.get/3`.

- [ ] Rewrite `get_headers_and_trailers/1` to accept a map and destructure
  `:headers` and `:trailers` in a single pattern match or two `Map.get` calls
  on the already-built map.

- [ ] Update `Keyword.fetch!(responses, :ok)` to `Map.fetch!(response_map, :ok)`.

- [ ] Update all call sites of `check_for_error/1` and
  `get_headers_and_trailers/1` to pass `response_map` instead of `responses`.

---

### 8.2 Tests

> Target file: `grpc_client/test/grpc/adapters/mint_test.exs`

- [ ] **Regression** — run all existing `connect/2`, `send_request`, and
  `receive_data` tests and confirm they still pass.

- [ ] **New test** — `"check_for_error/1 returns :ok when no error key is
  present in response map"`: call `Mint.check_for_error(%{ok: some_value})`
  and assert `:ok`.

- [ ] **New test** — `"check_for_error/1 returns {:error, reason} when error key
  is present in response map"`: call
  `Mint.check_for_error(%{error: "boom"})` and assert `{:error, "boom"}`.

- [ ] **New test** — `"get_headers_and_trailers/1 correctly extracts both keys
  from response map"`: call the function with a map containing both `:headers`
  and `:trailers` keys and assert the returned struct matches.

---

### 8.3 Documentation

- [ ] Add an inline comment in `do_receive_data/3` above the `Map.new(responses)`
  call explaining the single-pass conversion.

- [ ] Update the `@doc` (or inline spec comment) for `check_for_error/1` and
  `get_headers_and_trailers/1` to reflect that they now accept a map.

- [ ] Add a `CHANGELOG.md` entry:
  ```
  ### Changed
  - `Mint` adapter receive path: the response keyword list is now converted to
    a map once, replacing multiple independent linear scans with O(1) lookups.
  ```

---

