# L2 Flow Diff Review

## Baseline: Optimism

In standard `Optimism`, the `L2 -> L1` path uses generic withdraw-side logic inside `StandardBridge`.

The source-side path can:

- burn the representation token
- send a message to the counterpart bridge

The remote-side path then:

- either releases the locked asset
- or finalizes according to generic bridge branch logic

This model remains part of the shared `StandardBridge` design and does not introduce a custom per-token withdraw cap in the reviewed bridge core.

## Sky: L2 initiate path

In `Sky`, the corresponding `L2 -> L1` path is built through:

- `bridgeERC20(...)`
- `bridgeERC20To(...)`
- `_initiateBridgeERC20(...)`

### 1. Same wrapper pattern, but custom withdraw core

At the entrypoint level, `Sky` keeps the same wrapper style:

- self-path
- explicit `_to` path

But the internal `L2` initiate core is already custom and clearly shaped around a burn-and-release model.

### 2. Open / close gate

In `Sky`, withdraw initiation requires:

```solidity
require(isOpen == 1, "L2TokenBridge/closed");
```

This makes withdrawal initiation operationally dependent on the bridge-open state.

### 3. Pair validation through mapping

In `Sky`, token correctness is checked as:

```solidity
require(_localToken != address(0) && l1ToL2Token[_remoteToken] == _localToken, "L2TokenBridge/invalid-token");
```

So token-pair correctness again depends on explicit `l1ToL2Token` mapping.

### 4. Per-token withdraw cap

`Sky` introduces an additional restriction:

```solidity
require(_amount <= maxWithdraws[_localToken], "L2TokenBridge/amount-too-large");
```

This is not a generic balance inside the bridge.

It is:

- the maximum allowed size of a single withdrawal transaction
- configured per token

This cap is absent in the baseline `Optimism StandardBridge` path used in this comparison.

### 5. Burn-side source semantics

After the checks, `Sky` burns the `L2` representation token:

```solidity
TokenLike(_localToken).burn(msg.sender, _amount);
```

This makes the `Sky` source-side logic explicit and narrow:

- the `L2` representation token must support bridge-burn semantics
- the source sender is again fixed through `msg.sender`

### 6. L1 release from Escrow

After the cross-chain message, the counterpart `L1` finalize path releases funds not from the bridge balance and not through internal `deposits` decrement, but from external custody:

```solidity
TokenLike(_localToken).transferFrom(escrow, _to, _amount);
```

So withdraw-side settlement fully depends on:

- `Escrow`
- the allowance / approval path
- correct custody configuration

## Conclusion

The `Sky` `L2 -> L1` flow:

- keeps the canonical burn -> message -> release shape
- but makes the model narrower and more custom
- adds `isOpen`
- adds `maxWithdraws`
- removes internal `deposits` accounting
- ties release-side backing to external `Escrow`

Compared with `Optimism`, this path is less generic and more operationally opinionated.
