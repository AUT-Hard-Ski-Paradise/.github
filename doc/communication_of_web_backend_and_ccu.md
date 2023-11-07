# Communication Between the Central Control Unit and the Lift Operators.

The purpose of this document is to document all communication that takes place between the Central Control Unit (CCU) and the Web Backend.

*Note: this communication might need a rework if the solution for multiple types of messages in a single queue (specified [here](https://stackoverflow.com/questions/50288608/rabbitlistener-for-the-same-queue-in-multiple-classes)) does not work*

## Types of messages

The Web backend sends control messages for validation to the CCU
**`REQUEST_CONTROL_MESSAGE` schema:**
``` json
{
    "id" : int,
    "opcode" : int,
    "description" : string
}
```

**`REQUEST_STATE_MESSAGE` schema:**
``` json
{
    "session_id" : string
}
```

The CCU validates the control messages and sends a response to the Web BE
**`OPERATION_RESPONSE_MESSAGE` schema:**
``` json
{
    "opcode" : int,
    "description" : string,
    "status" : boolean,
    "reason" : string
}
```

If a lift fails the heartbeat, it will not be able to send an error report about it, so the CCU notifies the web backend
**`LIFT_OFFLINE_MESSAGE` schema:**
``` json
{
    "lift_id" : int,
    "reason" : string
}
```

**`STATE_RESPONSE_MESSAGE` schema:**
``` json
{
    "session_id" : string,
    "operators" : []
}