---
snip: 90
title: Censorship Resistance via Upper Bound Limits on L1 to L2 Messages
author: Denisa Diaconescu <denisa.diaconescu@nethermind.io>, Yudhishthra Sugumaran <yudhishthra.sugumaran@nethermind.io>
status: Draft
type: Standards Track
category: Core
created: 2024-06-26
---

## Summary

This SNIP proposes implementing upper bounds on messages from Layer 1 (L1) to Layer 2 (L2) to prevent sequencer censorship. The mechanism ensures that if a sequencer fails to process critical transactions within a defined limit, further batches from the sequencer will be refused until compliance is achieved. This approach leverages L1 as a censorship-resistant layer, ensuring transaction integrity, enhancing security, and providing users with reliable methods to enforce transaction inclusion and secure withdrawals.

## Abstract

The proposal introduces a framework to establish an upper limit on L1 to L2 message flows, creating a censorship-resistant environment within the Starknet ecosystem. By enforcing these limits, the protocol ensures that if sequencers fail to include essential transactions within the designated number of batches, the system will halt further batch processing until the transactions are addressed. This method enhances the security and reliability of L2 operations by providing users with mechanisms to enforce transaction compliance and secure their assets through L1, thereby maintaining the integrity of the network.

## Motivation

The increasing risk of sequencer censorship and failures in decentralized Layer 2 networks necessitates robust mechanisms to ensure reliable transaction processing and user fund security. This SNIP aims to leverage the immutable properties of Layer 1 to create a censorship-resistant framework. By setting upper bounds on L1 to L2 message flows, users are guaranteed that their critical transactions will be processed, thus preventing potential censorship by sequencers. This approach not only enhances the security and reliability of the network but also maintains user trust and network integrity by providing a dependable fallback mechanism.

## Implementation

```diff
// SPDX-License-Identifier: Apache-2.0.
pragma solidity ^0.6.12;

import "./IStarknetMessaging.sol";
import "starkware/solidity/libraries/NamedStorage.sol";

/**
  Implements sending messages to L2 by adding them to a pipe and consuming messages from L2 by
  removing them from a different pipe. A deriving contract can handle the former pipe and add items
  to the latter pipe while interacting with L2.
*/
contract StarknetMessaging is IStarknetMessaging {
+    // Upper bound for L1 to L2 messages
+    uint256 public upperBound;

+    // Mapping to store L1 messages and their deadlines
+    mapping(bytes32 => bool) public l1MessageIncluded;
+    mapping(bytes32 => uint256) public l1MessageDeadline;
+    bytes32[] public messageHashes;

+    // Blacklist for addresses
+    mapping(address => bool) public blacklist;

+    // Owner address
+    address owner;

    string constant L1L2_MESSAGE_MAP_TAG = "STARKNET_1.0_MSGING_L1TOL2_MAPPPING_V2";
    string constant L2L1_MESSAGE_MAP_TAG = "STARKNET_1.0_MSGING_L2TOL1_MAPPPING";
    string constant L1L2_MESSAGE_NONCE_TAG = "STARKNET_1.0_MSGING_L1TOL2_NONCE";
    string constant L1L2_MESSAGE_CANCELLATION_MAP_TAG = "STARKNET_1.0_MSGING_L1TOL2_CANCELLATION_MAPPPING";
    string constant L1L2_MESSAGE_CANCELLATION_DELAY_TAG = "STARKNET_1.0_MSGING_L1TOL2_CANCELLATION_DELAY";

    uint256 public constant MAX_L1_MSG_FEE = 1 ether;

+    modifier onlyOwner() {
+        require(msg.sender == owner, "Not the owner");
+        _;
+    }

+    constructor() public {
+        owner = msg.sender;
+    }

+    // Function to set upper bound
+    function setUpperBound(uint256 _upperBound) external onlyOwner {
+        upperBound = _upperBound;
+    }

+    // Function to submit L1 messages with tracking
+    function submitL1Message(bytes32 messageHash) external {
+        require(!l1MessageIncluded[messageHash], "Message already processed");
+        l1MessageIncluded[messageHash] = false;
+        l1MessageDeadline[messageHash] = block.number + upperBound;
+        messageHashes.push(messageHash);
+    }

+    // Function to enforce upper bound before processing new blocks
+    function processNewBatch() external {
+        require(checkAllMessagesProcessed(), "Unprocessed L1 messages exist within the bound");
+        // Logic to process the batch
+    }

+    // Helper function to check if all messages within the bound are processed
+    function checkAllMessagesProcessed() internal view returns (bool) {
+        for (uint256 i = 0; i < messageHashes.length; i++) {
+            bytes32 messageHash = messageHashes[i];
+            if (!l1MessageIncluded[messageHash] && block.number <= l1MessageDeadline[messageHash]) {
+                return false;
+            }
+        }
+        return true;
+    }

+    // Function to mark message as processed
+    function markMessageAsProcessed(bytes32 messageHash) external {
+        require(block.number <= l1MessageDeadline[messageHash], "Message deadline passed");
+        l1MessageIncluded[messageHash] = true;
+    }

+    // Function to dynamically adjust the upper bound
+    function adjustUpperBound(uint256 newBound) external onlyOwner {
+        upperBound = newBound;
+    }

+    // Function to add address to blacklist
+    function addToBlacklist(address user) external onlyOwner {
+        blacklist[user] = true;
+    }

+    // Function to remove address from blacklist
+    function removeFromBlacklist(address user) external onlyOwner {
+        blacklist[user] = false;
+    }

    function l1ToL2Messages(bytes32 msgHash) external view returns (uint256) {
        return l1ToL2Messages()[msgHash];
    }

    function l2ToL1Messages(bytes32 msgHash) external view returns (uint256) {
        return l2ToL1Messages()[msgHash];
    }

    function l1ToL2Messages() internal pure returns (mapping(bytes32 => uint256) storage) {
        return NamedStorage.bytes32ToUint256Mapping(L1L2_MESSAGE_MAP_TAG);
    }

    function l2ToL1Messages() internal pure returns (mapping(bytes32 => uint256) storage) {
        return NamedStorage.bytes32ToUint256Mapping(L2L1_MESSAGE_MAP_TAG);
    }

    function l1ToL2MessageNonce() public view returns (uint256) {
        return NamedStorage.getUintValue(L1L2_MESSAGE_NONCE_TAG);
    }

    function messageCancellationDelay() public view returns (uint256) {
        return NamedStorage.getUintValue(L1L2_MESSAGE_CANCELLATION_DELAY_TAG);
    }

+    // Additional functions for handling the new censorship resistance logic
+    function includeMessageInBatch(bytes32 messageHash) external onlyOwner {
+        require(block.number <= l1MessageDeadline[messageHash], "Message deadline passed");
+        require(!l1MessageIncluded[messageHash], "Message already included");
+        l1MessageIncluded[messageHash] = true;
+    }

+    function rejectNewBatch() external view returns (bool) {
+        return !checkAllMessagesProcessed();
+    }
}
```

## History

## Copyright

Copyright and related rights waived via [MIT](../LICENSE).
