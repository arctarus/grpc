# Mint Adapter — Performance Analysis

> Files analyzed:
> - `grpc_client/lib/grpc/client/adapters/mint.ex`
> - `grpc_client/lib/grpc/client/adapters/mint/connection_process/connection_process.ex`
> - `grpc_client/lib/grpc/client/adapters/mint/connection_process/state.ex`
> - `grpc_client/lib/grpc/client/adapters/mint/stream_response_process.ex`

---

## Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | 2 |
| 🟡 Moderate | 4 |
| 🔵 Minor    | 3 |

---

## 🔴 Critical Issues

---

### 1. Buffer not fully drained when a data frame contains multiple gRPC messages

**File:** `stream_response_process.ex` — `handle_call({:consume_response, {:data, data}})`

`GRPC.Message.get_message/2` is called exactly once per data chunk. If the buffer
contains more than one complete gRPC message (e.g. a single TCP/HTTP2 frame that carries
several small protobuf messages), only the first one is decoded and queued. The remaining
bytes stay in the buffer and will not be decoded until the **next** data event arrives —
which may be a long time away, or never if this was the last frame.

**Consequence:** consumers block on `:get_response` even though the data is already
sitting in the buffer; throughput drops proportionally to the number of messages packed
per frame.

**Current code:**
```elixir
case GRPC.Message.get_message(buffer <> data, state.compressor) do
  {{_, message}, rest} ->
    response = codec.decode(message, res_mod)
    new_responses = :queue.in({:ok, response}, responses)
    new_state = %{state | buffer: rest, responses: new_responses}
    {:reply, :ok, new_state, {:continue, :produce_response}}

  _ ->
    new_state = %{state | buffer: buffer <> data}
    {:reply, :ok, new_state, {:continue, :produce_response}}
end
```

**Suggested fix:** replace the single `case` with a loop (e.g. a private recursive
function or `Stream.unfold`) that keeps calling `get_message` until the buffer no longer
contains a complete message:

```elixir
defp drain_buffer(buffer, compressor, codec, res_mod, responses) do
  case GRPC.Message.get_message(buffer, compressor) do
    {{_, message}, rest} ->
      response = codec.decode(message, res_mod)
      drain_buffer(rest, compressor, codec, res_mod, :queue.in({:ok, response}, responses))

    _ ->
      {buffer, responses}
  end
end
```

Then call `drain_buffer(buffer <> data, ...)` in the handler and store both the
exhausted buffer and the fully-populated responses queue.

---

### 2. Synchronous `GenServer.call` from `ConnectionProcess` to `StreamResponseProcess` inside `handle_info`

**File:** `connection_process.ex` — `process_response/2` / `handle_info/2`

Every time the underlying socket delivers a batch of HTTP/2 responses,
`ConnectionProcess.handle_info/2` calls `StreamResponseProcess.consume/3` (and
`StreamResponseProcess.done/1`) using synchronous `GenServer.call` for **every
individual response** in the batch:

```elixir
{:ok, conn, responses} ->
  state = State.update_conn(state, conn)
  state = Enum.reduce(responses, state, &process_response/2)
  check_connection_status(state)
```

```elixir
defp process_response({:data, request_ref, new_data}, state) do
  :ok =
    state
    |> State.stream_response_pid(request_ref)
    |> StreamResponseProcess.consume(:data, new_data)  # <-- blocking call
  state
end
```

**Consequences:**
- The single `ConnectionProcess` (which owns the socket) is serially blocked waiting for
  each `StreamResponseProcess` to acknowledge every chunk. All open streams share this
  bottleneck.
- A slow or backed-up consumer on *one* stream stalls data delivery for **all other
  concurrent streams** on the same connection.
- Under high throughput (many small messages), the overhead of N×M round-trips across
  process mailboxes becomes a primary limiting factor.

**Applied fix:** `consume/3` and `done/1` were switched to `GenServer.cast`. The
corresponding `handle_call` clauses were renamed to `handle_cast` and their return
tuples updated from `{:reply, :ok, state, ...}` to `{:noreply, state, ...}`.

---

#### ⚠️ Trade-off: loss of implicit back-pressure

The synchronous `GenServer.call` was also an accidental back-pressure mechanism. Because
Mint operates the socket in **`active: :once`** mode, it delivers exactly one TCP message
to the owning process and then goes passive. The socket is only re-armed when
`Mint.HTTP.stream/2` is called inside `handle_info`. With `GenServer.call`, the following
chain existed:

```
TCP frame → ConnectionProcess mailbox
  → handle_info calls Mint.HTTP.stream  (re-arms socket → next frame can arrive)
    → GenServer.call(StreamResponseProcess)  ← ConnectionProcess BLOCKS
      → StreamResponseProcess decodes protobuf  (slow consumer)
      → replies
    → ConnectionProcess unblocks, handle_info returns
  → only NOW is the next frame processed
```

While `ConnectionProcess` was blocked, new TCP frames would pile up in the OS socket
buffer. If the buffer filled, TCP flow control would signal the server to stop sending.
The slow consumer's back-pressure propagated all the way to the network.

With `GenServer.cast` this chain is broken:

```
TCP frame → ConnectionProcess mailbox
  → handle_info calls Mint.HTTP.stream  (re-arms socket immediately)
    → GenServer.cast(StreamResponseProcess)  ← returns immediately
  → handle_info returns
→ next TCP frame arrives and is processed at full network speed
→ (repeats indefinitely regardless of consumer speed)
```

`ConnectionProcess` now drains TCP frames as fast as the network delivers them.
Each `StreamResponseProcess` mailbox absorbs all data that the application has not yet
consumed. Because `ConnectionProcess` reads eagerly, Mint keeps issuing HTTP/2
`WINDOW_UPDATE` frames to the server, which continues sending without any slow-down
signal. **If a consumer is slow, the `StreamResponseProcess` mailbox — and therefore
memory — will grow unboundedly.**

#### Before / after comparison

| Scenario | `GenServer.call` (before) | `GenServer.cast` (after) |
|---|---|---|
| Head-of-line blocking | Yes — one slow stream stalls all others | No — each stream is independent |
| Back-pressure to TCP | Yes — slow consumer eventually triggers TCP flow control | No — `ConnectionProcess` reads eagerly at all times |
| Memory under a slow consumer | Bounded — OS TCP buffers fill; server slows down | Unbounded — `StreamResponseProcess` mailboxes grow proportionally |
| `handle_info` throughput | O(N) synchronous round-trips per batch | O(N) mailbox enqueues (non-blocking) |
| Message ordering within a stream | Guaranteed (synchronous) | Guaranteed (Erlang FIFO mailbox) |

#### Why the trade-off is acceptable for now

- Buffering was always going to happen somewhere. With `call` the data sat in OS TCP
  buffers; with `cast` it sits in `StreamResponseProcess` mailboxes. The difference is
  that OS buffering is bounded by kernel settings, while mailbox buffering is bounded
  only by available memory.
- The `GenServer.call(:get_response, :infinity)` on the consumer side (`build_stream/2`)
  already provides application-level back-pressure: the application can only request the
  next item after the previous one has been delivered.
- For most gRPC streaming use cases, consumers process data close to the rate it arrives
  and the mailbox never grows large in practice.

#### Future mitigation options

If unbounded mailbox growth becomes a concern in production, the following approaches can
be considered — in increasing order of complexity:

1. **HTTP/2 flow control at the application layer.** Stop issuing `WINDOW_UPDATE` grants
   to Mint until the `StreamResponseProcess` queue drops below a threshold. This requires
   exposing a back-pressure signal from `StreamResponseProcess` back to
   `ConnectionProcess` so it can pause `Mint.HTTP2.stream_request_body` or delay
   `WINDOW_UPDATE` grants.

2. **Credit-based cast protocol.** Give each `StreamResponseProcess` a fixed credit
   count. `ConnectionProcess` only casts to it while credits are positive; a processed
   message sends a credit back via a separate cast. This keeps the mailbox bounded without
   reintroducing per-chunk synchronous round-trips.

3. **Demand-driven pipeline (`GenStage`).** Replace the push model with a pull model
   where the consumer explicitly demands items. `GenStage` provides this out of the box
   and integrates naturally with `Flow` for parallel processing.

---

## 🟡 Moderate Issues

---

### 3. Binary buffer concatenation allocates a new binary on every data chunk

**File:** `stream_response_process.ex` — `handle_call({:consume_response, {:data, data}})`

`buffer <> data` creates a brand-new binary every time a data chunk arrives. For
streaming calls with high message rates this triggers frequent large binary allocations
and places extra pressure on the garbage collector.

Additionally, when the buffer does not yet hold a complete message, the concatenation is
**evaluated twice** — once as the argument to `get_message` and once when storing the
new buffer value:

```elixir
_ ->
  new_state = %{state | buffer: buffer <> data}  # second concatenation
```

**Suggested fix:**
1. Bind the concatenated value once: `combined = buffer <> data`.
2. For a longer-term improvement, accumulate incoming chunks in an iolist and only
   convert to binary at the `get_message` call site, reducing copying:

```elixir
# accumulate as iolist
buffer_iolist = [buffer | [data]]
combined = IO.iodata_to_binary(buffer_iolist)
```

---

### 4. `decode_headers` called twice for the same headers on every `:headers` event

**File:** `stream_response_process.ex` — `handle_call({:consume_response, {:headers | :trailers, headers}})`

When headers arrive, both `update_compressor/2` and `get_headers_response/2` are called
independently, and each one calls `GRPC.Transport.HTTP2.decode_headers/1` on the same
raw header list:

```elixir
# First call — inside update_compressor/2
defp update_compressor({:headers, headers}, state) do
  decoded_trailers = GRPC.Transport.HTTP2.decode_headers(headers)
  ...
end

# Second call — inside get_headers_response/2
defp get_headers_response(headers, type) do
  decoded_trailers = GRPC.Transport.HTTP2.decode_headers(headers)
  ...
end
```

`decode_headers` iterates over the raw header list and builds a map; doing it twice per
event is pure waste.

**Suggested fix:** decode once at the call site and pass the result to both helpers:

```elixir
decoded = GRPC.Transport.HTTP2.decode_headers(headers)
state   = update_compressor_with_decoded({type, decoded}, state)
item    = get_headers_response_from_decoded(decoded, type)
```

---

### 5. `IO.iodata_length/1` called on the full body on every queue tick

**File:** `connection_process.ex` — `handle_continue(:process_request_stream_queue)`

Every time the request-stream queue is processed, the body size is recomputed from
scratch to compare it against the available window:

```elixir
cond do
  window_size == 0 -> {:noreply, state}
  IO.iodata_length(body) > window_size -> chunk_body_and_enqueue_rest(request, dequeued_state)
  true -> stream_body_and_reply(request, dequeued_state)
end
```

`IO.iodata_length/1` is O(n) in the number of iodata fragments. For large or deeply
nested iodata structures this is non-trivial, and it is repeated for every window-update
cycle.

**Suggested fix:** store the byte size alongside the body in the queue entry
`{request_ref, body, byte_size, from}` so the comparison is O(1). The size can be
computed once, before the entry is enqueued.

---

## 🔵 Minor Issues

---

### 6. `Process.flag(:trap_exit, true)` permanently mutates the calling process's exit-trapping behaviour

**File:** `mint.ex` — `connect/2`

```elixir
def connect(%{host: host, port: port} = channel, opts \\ []) do
  ...
  Process.flag(:trap_exit, true)   # <-- permanently mutates the CALLER
  channel
  |> mint_scheme()
  |> ConnectionProcess.start_link(host, port, opts)
  ...
end
```

#### Why the flag is legitimate here

`connect/2` calls `ConnectionProcess.start_link/4`, which links the new process to the
caller. That link stays alive for the lifetime of the connection. If `ConnectionProcess`
crashes at **any point** — whether during the HTTP/2 handshake, mid-stream, or on a
network drop — an EXIT signal with the crash reason is sent to every linked process,
including the caller.

Without `trap_exit`, that EXIT signal kills the caller outright (test process, controller,
GenServer, etc.). The flag converts those EXIT signals into harmless
`{:EXIT, pid, reason}` mailbox messages, keeping the caller alive.

Note that `catch :exit` only covers the **synchronous** path — when the `start_link` call
itself raises an exit exception (e.g. because `init/1` explicitly called `exit/1`). It
does **not** catch asynchronous EXIT signals delivered via the link mechanism after
`start_link` has already returned `{:ok, pid}`. The flag is therefore not redundant: it
covers a genuinely separate failure mode.

Removing it (as was done) breaks the two tests that use `ip: :loopback`, where Mint
connects successfully but `ConnectionProcess` then crashes asynchronously during the
HTTP/2 settings exchange, sending an EXIT signal that kills the (unprotected) test
process.

#### The real problem

The flag is set unconditionally and **never restored**, so `connect/2` permanently changes
the caller's exit-trapping behaviour as a side effect. Any process that calls
`Mint.connect/2` will silently start trapping all exits for the rest of its lifetime —
including exits unrelated to gRPC — which can mask crashes and is hard to debug.

#### Suggested fix

Save the previous flag value before setting it, and restore it after `start_link`
returns. This preserves the legitimate EXIT protection during the `start_link` call while
leaving the caller's exit-trapping state exactly as it was found:

```elixir
def connect(%{host: host, port: port} = channel, opts \\ []) do
  {config_opts, opts} = Keyword.pop(opts, :config_options, [])
  module_opts = Application.get_env(:grpc, __MODULE__, config_opts)
  opts = connect_opts(channel, opts) |> merge_opts(module_opts)

  previous_flag = Process.flag(:trap_exit, true)

  result =
    channel
    |> mint_scheme()
    |> ConnectionProcess.start_link(host, port, opts)

  Process.flag(:trap_exit, previous_flag)

  case result do
    {:ok, pid} ->
      {:ok, %{channel | adapter_payload: %{conn_pid: pid}}}

    error ->
      {:error, "Error while opening connection: #{inspect(error)}"}
  end
end
```

The `catch :exit` block can be retained as an extra safety net for any unexpected
synchronous exits, but the flag is the primary protection for the asynchronous case.

---

### 7. Dead code: `State.append_response_data/3` is never called

**File:** `connection_process/state.ex`

```elixir
def append_response_data(state, ref, new_data) do
  update_in(state.requests[ref].response[:data], fn data -> (data || "") <> new_data end)
end
```

This function also uses string concatenation (`(data || "") <> new_data`) in a hot path,
which would have been a performance issue had it been in use. A grep across the entire
codebase confirms it is not called anywhere — it is leftover from an earlier
implementation that accumulated response data inside the `ConnectionProcess` state rather
than routing it to `StreamResponseProcess`.

**Suggested fix:** delete the function.

---

### 8. `get_headers_and_trailers/1` and `check_for_error/1` traverse the keyword list multiple times

**File:** `mint.ex`

```elixir
defp get_headers_and_trailers(responses) do
  %{
    headers: Keyword.get(responses, :headers),    # first traversal
    trailers: Keyword.get(responses, :trailers)   # second traversal
  }
end

def check_for_error(responses) do
  error = Keyword.get(responses, :error)          # yet another traversal
  if error, do: {:error, error}, else: :ok
end
```

Both functions are called on the same `responses` keyword list in the unary/client-stream
receive path. Each `Keyword.get` is a linear scan.

**Suggested fix:** convert `responses` to a map once upstream, or perform a single-pass
reduction that extracts all needed keys simultaneously:

```elixir
defp extract_response_fields(responses) do
  Enum.reduce(responses, %{}, fn {k, v}, acc -> Map.put(acc, k, v) end)
end
```

---

## Quick-Reference Table

| # | Severity | File | Issue |
|---|----------|------|-------|
| 1 | 🔴 Critical | `stream_response_process.ex` | Buffer not drained for multi-message frames — **merged to master** |
| 2 | 🔴 Critical | `connection_process.ex` | Synchronous `GenServer.call` inside `handle_info` blocks all streams — **applied; see trade-off note** |
| 3 | 🟡 Moderate | `stream_response_process.ex` | Binary concatenation allocates a new copy per data chunk; computed twice in failure branch |
| 4 | 🟡 Moderate | `stream_response_process.ex` | `decode_headers` called twice for the same headers |
| 5 | 🟡 Moderate | `connection_process.ex` | `IO.iodata_length` recomputed on every queue tick |
| 6 | 🔵 Minor | `mint.ex` | `Process.flag(:trap_exit, true)` permanently mutates the calling process — should save and restore |
| 7 | 🔵 Minor | `state.ex` | Dead code: `append_response_data/3` never called |
| 8 | 🔵 Minor | `mint.ex` | Multiple linear scans over the same keyword list |