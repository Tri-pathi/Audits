### Use assembly to emit events

We can use assembly to emit events efficiently by utilizing `scratch space` and the `free memory pointer`. This will allow us to potentially avoid memory expansion costs.

Note: In order to do this optimization safely, we will need to cache and restore the free memory pointer

```solidity

 emit Disputed(msg.sender);



```

```solidity
    assembly {
        let eventSignature := 0x86e33e28 
        // Keccak256 hash of "Disputed(address)"
        let eventData := mload(0x40)      // Allocate memory for event data
        mstore(eventData, eventSignature) // Store event signature

        // Store the `msg.sender` value
        mstore(add(eventData, 0x20), caller())

        // Emit the event
        log1(eventData, 0x40, 0x0, 0x0)
        }
```
