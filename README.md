# Sky OP Token Bridge Comparison Review

## Scope

This repository contains a comparison review of the custom `Sky OP Token Bridge` against the standard `Optimism Bedrock` bridge model.

The comparison is focused on:

- the `L1 -> L2` initiating flow
- the `L2 -> L1` initiating flow
- architectural deviations from `Optimism StandardBridge`
- new assumptions and new risk surface introduced by `Sky` customizations

## Baseline

The baseline used in this comparison is the `Optimism Bedrock` bridge model:

- `L1StandardBridge`
- `L2StandardBridge`
- `StandardBridge`

This baseline matters because it uses a more generic bridge core with:

- a shared `StandardBridge`
- internal `deposits[localToken][remoteToken]` accounting for ordinary-token custody
- generic mint / burn / release branching
