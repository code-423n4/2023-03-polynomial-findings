* `Exchange.sol`: `SUSD` state variable is never used and can be removed

* `LiquidityPool.sol`: Queued deposits always have `requestedTime` set so the `current.requestedTime == 0` check
in `processDeposits` is redundant

* `solmate` does not protect ERC20 transfers to the zero address and it's left up to users
of the library to handle such a case. Currently in all Polynomial ERC20 contracts (`PowerPerp`,
`VaultToken`, and `LiquidityToken`) it's possible to transfer to the zero address

* `LiquidityPool.sol`: If the fee recipient is not set then all LP operations such as
deposits and withdrawals will fail. Consider making fee transfers optional depending on
whether a fee recipient and percentage is set

* `LiquidityPool.sol`: The deposit queue processing in `processDeposits` reads the price of the
liquidity token only once at the start of the loop so this means that some queued deposits
may not incur price slippage depending on how `processDeposits` is executed. Imagine 4
queued deposits that are ready to be processed: `processDeposits(4)` means all deposits
will be processed with the same price whereas calling `processDeposits(1)` 4 times will
force all but the first deposit to get worse execution.