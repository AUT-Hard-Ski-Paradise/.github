# Communication Between the Central Control Unit and the Lift Operators.

The purpose of this document is to document all communication that takes place between the Central Control Unit (CCU) and the Lift Operators (LOs).

## Types of messages
**`CONTROL_MESSAGE` schema:**
``` json
{
    "opcode" : int,
    "description" : string
}
```
`opcode`:
- `0`: `POLL_AVAILABLE_OPERATORS`
- `1`: *reserved for smth important*
- `2`: `START_LIFT`
- `3`: `STOP_LIFT`
- `4`: `SLOW_LIFT`
- ...

**`LIFT_STATUS_REPORT` schema:**
``` json
{
    "type" : ENUM VAL,
    "name" : string,
    "state" : ENUM VAL,
    "throughput" : int,
    "queue_size" : int,
    "wind_speed" : string
}
```
`type`:
- `GONDOLA`
- `CHAIRLIFT`
- `T_BAR`

`state`:
- `STARTED`
- `STOPPED`
- `SLOW`

`throughput`: Number of people boarding per minute
`queue_size`: Number of people waiting in queue
`wind_speed`: Floating point number. unit: km/h

**`ERROR_REPORT` schema:**
``` json
{
    "message" : string,
    "type" : int,
    "severity" : int
}
```
`type`:
- 0 : motor failiure
- ...

`severity`:
- Warning
- Dangerous
- Fatal
## Keeping track of Lift Operators

A ski resort has a limited number of ski lifts. The number of lifts can be increased, however it is a slow and costly process, thus there is no need to engineer a dynamic way of adding or removing ski lifts from the system. A simple config file is sufficient.

The Central Control Unit has a JSON config file which is filled with the preconfigured Name and ID of all the available lifts in the ski resort.

*example for the JSON config:*
``` json
{
    [
        {
            "name" : "La Fée",
            "id" : 1,
            "type" : "CHAIRLIFT"
        },
        {
            "name" : "Pierre Grosse",
            "id" : 2,
            "type" : "GONDOLA"
        }
    ]
}
```

All the Lift operators have Name and ID from configurations. These configurations should be set on pod level in the kubernetes deployment.

### Initializing Lift Operators

When a Central Control Unit is created, it reads the JSON config and builds a list to keep track of the Lift Operators. At first, all the Operators should be considered `Offline`. Then the CCU consumes all remaining the messages in the `reports` queue then polls the available Lift Operators using a `POLL_AVAILABLE_OPERATORS` control message (opcode 0) using the `Broadcast` exchange.

**`POLL_AVAILABLE_OPERATORS` message:**
``` json
{
    "opcode" : 0,
    "description" : "Poll available Lift Operators."
}
```

If a lift operator receives a `POLL_AVAILABLE_OPERATORS` control message, or is first started, it sends a `LIFT_STATUS_REPORT` message to the CCU using the `Reports` exchange.

*Note that if a CCU instance is created after a Lift Operator, the `Reports` queue might have status reports and error reports from previously started LOs that could be outdated.*

To address this, the CCU should ignore any remaining status messages. This is to avoid listing Lift Operators as `Online` when they might have been shut down or disconnected since then. Only trust `LIFT_STATUS_REPORT` messages after the `POLL_AVAILABLE_OPERATORS` has been sent.

However, error reports should be cached, and handled when the corresponding Lift Operators status message is received and the CCU considers them `Online`.

**`LIFT_STATUS_REPORT` example:**
``` json
{
    "id" : 3,
    "type" : "GONDOLA",
    "name" : "Pierre Grosse",
    "state" : "STARTED",
    "throughput" : 32,
    "queue_size" : 241,
    "wind_speed" : "56"
}
```
When a `LIFT_STATUS_REPORT` is received and the LO of the corresponding id is listed as `Offline` on the CCU, it should be set to `Online` and updated with the status data.

### Heartbeat

The CCU periodically sends a `POLL_AVAILABLE_OPERATORS` message. If a LO fails to send a response until timeout, the CCU marks it as offline and notifies the Web Backend. Otherwise it updates the internal

