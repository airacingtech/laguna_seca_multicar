# Multicar Rules of Engagement — Laguna Seca

| | |
|---|---|
| **Version** | 0.2.0 |
| **Status** | Draft |
| **Assumes** | IAC Passing Competition Rules v1.3.1 (Laguna Seca Edition) |

Engagement follows the **IAC Passing Competition Rules** — roles, rounds, right-of-way, safety
distances, and stops are defined there and are not restated here. This document defines only the
**shared state message** each car broadcasts so the others can track it.

The message is a **perception enhancement**: it puts the other car on the wire as a tracked object —
pose, velocity, and state — a shared, common-baseline signal each team can fuse with its own perception,
cross-check against it, or fall back to when onboard sensing of the rival is degraded or absent. It is
**stack-agnostic**: consume it exactly as you would any tracked object; nothing about your perception,
planning, or control stack is assumed.

## State message

```
std_msgs/Header header    # Standard ROS header; stamp drives relative timing
uint8   car_id            # Car identifier [ - ]
uint8   heartbeat         # Rolling counter; a stalled counter exposes a degraded link [ - ]

float64 lat               # Latitude, rear-axle centre (vehicle frame origin) [ dd.dd ]
float64 lon               # Longitude, rear-axle centre [ dd.dd ]
float32 alt               # Altitude (ellipsoid), rear-axle centre [ m ]
float32 heading           # Heading, GPS convention: North = 0, East = 90 [ deg ]
float32 v_north           # North velocity, ENU [ m/s ]
float32 v_east            # East velocity, ENU [ m/s ]
float32 v_up              # Up velocity, ENU [ m/s ]
uint32  gps_tow_ms        # GPS time-of-week [ ms ]
int8    ct_state          # Vehicle state, see constants [ enum ]

# Vehicle state constants (aligned with the Raptor ct_state enumeration)
int8 CT_STATE_UNKNOWN = 0
int8 CT_STATE_CAUTION = 8
int8 CT_STATE_NOMINAL = 9
int8 CT_STATE_SAFE_STOP = 10
int8 CT_STATE_EMERGENCY_SHUTDOWN = 12
```

## Using it

- Populate the pose and velocity fields from your own state estimate — the source (GNSS, INS, fusion, or
  otherwise) is up to each team. Publish at a common rate across teams, **10 Hz for now**, so every car's
  feed lines up.
- `gps_tow_ms` is GPS time-of-week: a shared clock across cars, so messages align regardless of each
  car's local time sync.
- The pose origin is the rear-axle centre, so the axle references the IAC right-of-way rules use follow
  directly from it.
- Treat any `ct_state` outside the set above as `CT_STATE_UNKNOWN`.
- Consume the message as a tracked object of the other car — everything else is the IAC ruleset.

## Emergency-stop coordination

The one behaviour driven by this message rather than the ruleset: on a transition into
`CT_STATE_EMERGENCY_SHUTDOWN`, latch the initiating `car_id` and `header.stamp` and come to a controlled
stop. Stay stopped until a newer message from that car reports a non-emergency state, or Race Control
clears it.[^race-control] Ignore repeats of the same stop event so a stale packet cannot re-trigger a
phantom stop.

[^race-control]: These rules assume an active human Race Control. Race Control must be prepared to halt
the field if the state feed fails, and to stop trailing cars when a leading car leaves the track or stops
unexpectedly. Teams may run their own perception, but cannot assume other entrants do — hence this shared
feed. Participation requires a longitudinal control system that respects the IAC separation rules.
