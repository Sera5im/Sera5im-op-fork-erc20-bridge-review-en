# L1 Flow Diff Review

## Baseline: Optimism

In standard `Optimism`, the ordinary-token `L1 -> L2` path uses a generic bridge core:

- `depositERC20(...)`
- `depositERC20To(...)`
- `_initiateERC20Deposit(...)`
- `_initiateBridgeERC20(...)`

In the ordinary-token branch, the bridge:

- takes custody of the `L1` token inside the bridge itself
- increases internal accounting:
  - `deposits[localToken][remoteToken] += amount`
- sends a message to the counterpart bridge
- lets the remote side finalize through mint-side or release-side logic depending on the token model

## Sky: L1 initiate path

In `Sky`, the corresponding path is built through:

- `bridgeERC20(...)`
- `bridgeERC20To(...)`
- `_initiateBridgeERC20(...)`

## Main differences

### 1. Naming and interface surface

`Sky` uses the more neutral `bridgeERC20` instead of `depositERC20`.

This shows that the target no longer preserves the `Optimism` naming model one-to-one and already uses its own wrapper surface.

### 2. Inline EOA check

Instead of `onlyEOA`, `Sky` uses an inline check:

```solidity
require(msg.sender.code.length == 0, "L1TokenBridge/sender-not-eoa");
```

By meaning, this is the same protective layer against contract callers for the self-bridge path, but implemented without the standard modifier layer.

### 3. Narrower initiate core

In `Optimism`, the generic initiate core accepts both `_from` and `_to` and branches between mintable and ordinary-token paths.

In `Sky`, the `L1` initiate path is narrower:

- the sender is fixed through `msg.sender`
- `_from` is not passed from the outside as a separate generic parameter
- the flow is tied to a specific `L1 -> L2` custody-and-message model

### 4. No internal deposits notebook

In `Optimism`, the ordinary-token path keeps internal `deposits` accounting.

In `Sky`, that accounting notebook is removed.

Instead, `Sky` uses:

```solidity
TokenLike(_localToken).transferFrom(msg.sender, escrow, _amount);
```

This means the backing:

- does not stay inside the bridge
- is not tracked through `deposits[local][remote]`
- is moved directly into a separate `Escrow`

### 5. Pair validation through explicit mapping

In `Sky`, the initiate path also requires:

```solidity
require(_remoteToken != address(0) && l1ToL2Token[_localToken] == _remoteToken, "L1TokenBridge/invalid-token");
```

So token-pair correctness depends on the explicit `l1ToL2Token` mapping rather than on the more generic pair model used by `Optimism`.

### 6. Open / close operational gate

`Sky` adds:

```solidity
require(isOpen == 1, "L1TokenBridge/closed");
```

This means new cross-chain initiation explicitly depends on a custom open / close state.

### 7. Finalization target on L2

The `Sky` `L1 -> L2` flow ultimately ends in mint-side finalization on `L2`, where the counterpart bridge simply mints the representation token to the recipient:

```solidity
TokenLike(_localToken).mint(_to, _amount);
```

So `Sky` makes the destination-side finalize path narrower and less generic than `Optimism StandardBridge`.

## Conclusion

The `Sky` `L1 -> L2` flow:

- removes generic `StandardBridge`-style custody accounting
- moves backing into a separate `Escrow`
- makes pair correctness mapping-driven
- fixes sender semantics more tightly inside the bridge
- finishes the remote-side path through a narrow mint finalize

This makes the flow more opinionated and more operationally constrained than the generic `Optimism` baseline.
