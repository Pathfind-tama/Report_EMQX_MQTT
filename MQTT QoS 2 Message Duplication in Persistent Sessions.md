# CVE Report: EMQX MQTT QoS 2 Message Duplication in Persistent Sessions

## Vulnerability Description

EMQX broker violates the MQTT QoS 2 "exactly once" delivery guarantee in persistent sessions due to a non-atomic operation sequence when processing PUBLISH packets. The broker publishes messages to subscribers **before** the PacketId is committed to persistent storage, creating a time window where a crash or reconnection can cause message duplication.

## Code Evidence

**File**: `apps/emqx/src/emqx_persistent_session_ds.erl`  
**Lines**: 520-522

```erlang
undefined ->
    Results = emqx_broker:publish(Msg),           % Line 520: Publish first
    S = emqx_persistent_session_ds_state:put_awaiting_rel(PacketId, TS, S0),
    {ok, Results, ensure_state_commit_timer(Session#{s := S})};  % Line 522: Async commit
```

**The Problem**:

1. **Line 520**: Message is published to subscribers immediately
2. **Line 521**: PacketId is recorded in memory state  
3. **Line 522**: State commit is **asynchronous** via `ensure_state_commit_timer`

The state commit happens through a timer, not synchronously. If the broker crashes or the client reconnects before the timer fires, the PacketId is not persisted, allowing duplicate delivery.

## Trigger Conditions

**Prerequisites**:
1. EMQX broker with persistent sessions (DS storage)
2. MQTT client with `clean_session=false`
3. At least one subscriber

**Minimal Trigger**:

1. Client publishes QoS 2 message: `PUBLISH(QoS=2, PacketId=1)`
2. Broker executes line 520: message delivered to subscribers
3. Broker executes line 521-522: schedules async commit
4. **[CRITICAL WINDOW]** Before commit timer fires:
   - Broker crashes, OR
   - Client reconnects to different cluster node
5. PacketId is NOT persisted
6. Client retransmits: `PUBLISH(QoS=2, PacketId=1, DUP=1)`
7. Broker checks `awaiting_rel`: returns `undefined`
8. Broker publishes again → **Message duplicated**

**Why Hard to Reproduce**:
- Timing window depends on commit timer interval
- Requires crash or reconnection during window
- But cluster reconnections make it more likely

## Impact

**Business Impact**:
- Financial: Duplicate payments, double transactions
- IoT: Duplicate commands (unlock door twice, etc.)
- Event systems: Duplicate events corrupt state

**Technical Impact**:
- Violates MQTT 3.1.1/5.0 QoS 2 specification
- Persistent sessions don't prevent duplication
- Silent failure (no error logs)

## Root Cause

Asynchronous state commit via `ensure_state_commit_timer`:

```erlang
ensure_state_commit_timer(Session) ->
    % Schedules a timer to commit state later - NOT synchronous!
```

**Why This is Wrong**:
- Publish and commit are not atomic
- Timer may not fire before crash/reconnection
- Performance prioritized over correctness

## Vendor Response

**Status**: Vendor acknowledged but does not classify as security vulnerability

**Key Points**:
1. ✓ Confirmed DS storage has atomicity issues
2. ✓ Acknowledged QoS 1 has similar problems
3. ✓ Agreed proposed fix is "logically correct"
4. ✓ Mentioned DS already supports transactions
5. ✗ No fix timeline ("future enhancement plan")


## Recommended Fix

**Correct Implementation**:
```erlang
undefined ->
    % 1. Record PacketId first
    S1 = put_awaiting_rel(PacketId, TS, S0),
    % 2. Synchronously commit to storage
    ok = sync_commit_state(S1),
    % 3. Then publish message
    Results = emqx_broker:publish(Msg),
    {ok, Results, Session#{s := S1}};
```

**Benefits**:
- If crash after step 2, PacketId is persisted
- Retransmission rejected with PACKET_IDENTIFIER_IN_USE
- No duplication possible

**Trade-off**: Slightly slower but guarantees correctness

## Workarounds

1. **Application-Level Deduplication**: Track message IDs, reject duplicates
2. **Idempotent Operations**: Design operations to be safely repeatable
3. **Monitoring**: Alert on duplicate messages

## Affected Versions

- EMQX 5.x
- EMQX 6.x (verified on 6.1.0-sf)

## References

- MQTT 3.1.1 Specification: Section 4.3.3 (QoS 2: Exactly once delivery)
- EMQX Source: `apps/emqx/src/emqx_persistent_session_ds.erl`


