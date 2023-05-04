
### Description

Trip and Untrip event can be falsely emitted.

### Steps to reproduce

1. Caller should be watcher with TRIP_ROLE and UNTRIP_ROLE respectively.
2. Call `tripGlobal` . 
3. Now , again call `tripGlobal` with new signature.
4. `SwitchboardTripped` event will be emitted again.

### Expected behavior

Emit only if state changes
### Actual behavior

State is same, but event emits. Leading to unexpected outcome in front end.
### Screenshots

```solidity
function tripGlobal(uint256 nonce_, bytes memory signature_) external {
        address watcher = SignatureVerifierLib.recoverSignerFromDigest(
            // it includes trip status at the end
            keccak256(abi.encode("TRIP", chainSlug, nonce_, true)),
            signature_
        );

        if (!_hasRole(TRIP_ROLE, watcher)) revert NoPermit(TRIP_ROLE);

        uint256 nonce = nextNonce[watcher]++;
        if (nonce_ != nonce) revert InvalidNonce();

        tripGlobalFuse = true;
        emit SwitchboardTripped(true);
    }
```


```solidity
 function untrip(uint256 nonce_, bytes memory signature_) external {
        address watcher = SignatureVerifierLib.recoverSignerFromDigest(
            // it includes trip status at the end
            keccak256(abi.encode("UNTRIP", chainSlug, nonce_, false)),
            signature_
        );

        if (!_hasRole(UNTRIP_ROLE, watcher)) revert NoPermit(UNTRIP_ROLE);
        uint256 nonce = nextNonce[watcher]++;
        if (nonce_ != nonce) revert InvalidNonce();

        tripGlobalFuse = false;
        emit SwitchboardTripped(false);
    }
```
### Additional information

Add a require statemnt in both the functions to ensure function can only be called only on state change.
