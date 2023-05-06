### Adversary can DOS executeBatch function

SocketBatcher allows executing many messages in a single call using `executeBatch`. This function externally calls `execute` function in the socket contract and iterates thorughout the array individually.

Each message can only be executed once(provided it did not fail while doing that). This is achieved by this mapping.

```solidity
    mapping(bytes32 => bool) public messageExecuted;
```

Problem is anyone can cause the `executeBatch` to fail .

Checkout the below example

### Steps to reproduce

1. Caller calls `executeBatch` with 10 messages to be executed.

2. Attacker monitors the tx in mempool .

3. Attacker picks one of the message, calls the `execute` function directly . This will update the `messageExecuted` to true.

4. Now, when the original caller's call goes, the whole execution will revert as one of the message has already been executed. 

This falls under the catefgory of `Damage to users/protocol due to griefing` . Hence I assign it as a medium.

### Expected behavior

`executeBatch` should not fail.
### Actual behavior

`executeBatch` will fail using frontrunning
### Screenshots

```solidity
function executeBatch(
        address socketAddress_,
        ExecuteRequest[] calldata executeRequests_
    ) external {
        uint256 executeRequestslength = executeRequests_.length;
        for (uint256 index = 0; index < executeRequestslength; ) {
            ISocket(socketAddress_).execute(
                executeRequests_[index].packetId,
                executeRequests_[index].localPlug,
                executeRequests_[index].messageDetails,
                executeRequests_[index].signature
            );
            unchecked {
                ++index;
            }
        }
    }

```

```solidity
function execute(
        bytes32 packetId_,
        address localPlug_,
        ISocket.MessageDetails calldata messageDetails_,
        bytes memory signature_
    ) external override {
        if (messageExecuted[messageDetails_.msgId])
            revert MessageAlreadyExecuted();
        messageExecuted[messageDetails_.msgId] = true;

        uint256 remoteSlug = _decodeSlug(messageDetails_.msgId);

        PlugConfig storage plugConfig = _plugConfigs[localPlug_][remoteSlug];

        bytes32 packedMessage = hasher__.packMessage(
            remoteSlug,
            plugConfig.siblingPlug,
            chainSlug,
            localPlug_,
            messageDetails_.msgId,
            messageDetails_.msgGasLimit,
            messageDetails_.executionFee,
            messageDetails_.payload
        );

        (address executor, bool isValidExecutor) = executionManager__
            .isExecutor(packedMessage, signature_);
        if (!isValidExecutor) revert NotExecutor();

        _verify(
            packetId_,
            remoteSlug,
            packedMessage,
            plugConfig,
            messageDetails_.decapacitorProof
        );
        _execute(
            executor,
            messageDetails_.executionFee,
            localPlug_,
            remoteSlug,
            messageDetails_.msgGasLimit,
            messageDetails_.msgId,
            messageDetails_.payload
        );
    }
```
### Additional information

Use Try/Catch in `executeBatch` function.
