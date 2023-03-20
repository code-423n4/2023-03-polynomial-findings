G1. processDepositQueue() modifies the state variable ``queuedDepositHead`` in a for loop, which is a waste of gas. This variable should be modified ONCE only after the for loop completes.

```diff
+           uint256 cachedQueuedDepositHead = cachedQueuedDepositHead;

for (uint256 i = 0; i < idCount; i++) {
-            QueuedDeposit storage current = depositQueue[queuedDepositHead];
+            QueuedDeposit storage current = depositQueue[cachedQueuedDepositHead];


            if (current.requestedTime == 0 || block.timestamp < current.requestedTime + minDepositDelay) {
-                return;
+                break;
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
G2. There is no need to call ``getTokenPrice();`` inside the for loop to get the token price. This line can be moved before the for loop so that it will be called only once to save gas.

[https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L271](https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L271)

