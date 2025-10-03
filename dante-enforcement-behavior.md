# Dante Subscription Enforcement Behavior

## Overview
The dante enforcement services use a "check then act" pattern to maintain dante subscriptions without unnecessary network traffic. They continuously monitor state but only send dante protocol requests when actual configuration drift is detected.

## Service Types

### dante-subscription@.service & dante-disconnection@.service
- **Purpose**: Enforce specific dante subscriptions (connections/disconnections)
- **Mode**: Continuous monitoring with state-based enforcement
- **Script**: `/home/chris/bin/dante_device/src/dante_device/__main__.py --mode assert-subscription|assert-disconnection`

### dante-device-subscription@.service & dante-device-disconnection@.service  
- **Purpose**: Same as above but with different execution path
- **Script**: Direct python execution of dante_device module

## Enforcement Logic (subscription.py)

### 1. State Check First (Lines 116-123)
```python
coro = check_subscription_state_lightweight(...)
current_state_correct = self.run_async(coro)
```
- Reads current dante device subscriptions from network
- Compares actual state vs desired state
- **No dante protocol requests sent during state checking**

### 2. Conditional Enforcement (Lines 125-153)
```python
if current_state_correct:
    # State is correct - no action needed
    self.state["subscription_enforced"] = True
    self.notifier.ready()  # Tell systemd we're satisfied
else:
    # State drift detected - send correction
    coro = manage_subscription_async(..., action)
    enforce_result = self.run_async(coro)
```

**Key Point**: Dante protocol write requests are ONLY sent when `current_state_correct` is False

### 3. Timing Behavior (Lines 94-109)
- **Initial checks**: 1 second, then 3 seconds after service start
- **Continuous monitoring**: Every 3 seconds thereafter
- **Heartbeat**: Status updates every 10 seconds

## State Checking Methods

### check_subscription_state_lightweight() 
- Reads subscription list from cached device data
- Fast string comparison against existing subscriptions
- Falls back to full check if lightweight method fails
- **Read-only operation** - no dante configuration changes

### manage_subscription_async()
- **Write operation** - actually sends dante protocol requests
- Only called when state drift is detected
- Performs add_subscription() or remove_subscription() on dante devices

## Problem Scenarios

### Aggressive Retry Loops
When dante devices are missing/not ready:
- Services retry every 3 seconds indefinitely
- Each retry attempts device discovery and state checking
- Can overwhelm dante devices during startup/initialization
- May cause devices to reset to factory defaults as protection

### Missing Channel Names
If configured channel names don't exist on devices:
- State check always returns "incorrect" 
- Service continuously attempts to create non-existent subscriptions
- Generates constant dante protocol traffic
- Example: Trying to connect to `satellite-07:desktop-mic` when device only has `satellite-08:desktop-mic`

## Rate Limiting Behavior

### Current Intervals
- **State checks**: Every 3 seconds (continuous)
- **Startup delays**: 1 second, then 3 seconds for initial checks
- **Status updates**: Every 10 seconds (heartbeat)

### When Protocol Requests Are Sent
1. **Device discovery phase** - Queries dante devices for available channels
2. **Subscription enforcement** - Only when state drift detected
3. **Error retry** - When previous attempts failed

### Ideal Behavior
- Read current state frequently (low cost)
- Only write configuration when state has drifted (high cost)
- Back off retry intervals when devices are missing/not ready
- Respect dante device initialization timing

## Systemd Integration

### Service Lifecycle
- `Type=notify` - Service calls `notifier.ready()` when enforcement is active
- `Restart=no` - Services don't auto-restart on failure
- Long-running services that monitor continuously

### Conflict Management
- Services use `Conflicts=` to prevent opposing enforcement
- Example: Connection services conflict with disconnection services
- Prevents enforcement loops between different desired states