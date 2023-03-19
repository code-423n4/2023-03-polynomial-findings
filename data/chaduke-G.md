G1. processDepositQueue() modifies the state variable ``queuedDepositHead`` in a for loop, which is a waste of gas. This variable should be modified ONCE only after the for loop completes.

```diff
+           uint256 cachedQueuedDepositHead = cachedQueuedDepositHead;

for (uint256 i = 0; i < idCount; i++) {
-            QueuedDeposit storage current = depositQueue[queuedDepositHead];
+            QueuedDeposit storage current = depositQueue[cachedQueuedDepositHead];


            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
                return;
            }

            uint256 tokensToMint = current.depositedAmount.divWadDown(tokenPrice);

            current.mintedTokens = tokensToMint;
            totalQueuedDeposits -= current.depositedAmount;
            totalFunds += current.depositedAmount;
            VAULT_TOKEN.mint(current.user, tokensToMint);

            emit ProcessDeposit(current.id, current.user, current.depositedAmount, tokensToMint, current.requestedTime);

            current.depositedAmount = 0;
-            queuedDepositHead++;
+            cachedQueuedDepositHead++;
        }

+       queuedDepositHead = cachedQueuedDepositHead;
```