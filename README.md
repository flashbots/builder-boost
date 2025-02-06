# Rollup Boost

This repo implements a proxy server for the Ethereum Engine API and is under active development. To read more about the design, check out the [design doc](https://github.com/ethereum-optimism/design-docs/pull/86).


## Usage

Run the server using the following command:

```
cargo run -- [OPTIONS]
```

### Command-line Options

- `--l2-jwt-token <TOKEN>`: JWT token for L2 authentication (required)
- `--l2-jwt-path <PATH>`: Path to the L2 JWT secret file (required if `--l2-jwt-token` is not provided)
- `--l2-url <URL>`: URL of the local L2 execution engine (required)
- `--builder-url <URL>`: URL of the builder execution engine (required)
- `--builder-jwt-token <TOKEN>`: JWT token for builder authentication (required)
- `--builder-jwt-path <PATH>`: Path to the builder JWT secret file (required if `--builder-jwt-token` is not provided)
- `--rpc-host <HOST>`: Host to run the server on (default: 0.0.0.0)
- `--rpc-port <PORT>`: Port to run the server on (default: 8081)
- `--tracing`: Enable tracing (default: false)
- `--log-level <LEVEL>`: Log level (default: info)
- `--metrics`: Enable metrics (default: false)
- `--boost-sync`: Enable syncing the builder with the proposer op-node (default: false)

### Environment Variables

You can also set the options using environment variables. See .env.example to use the default values.

### Example

```
cargo run --l2-jwt-token your_jwt_token --l2-url http://localhost:8545 --builder-jwt-token your_jwt_token --builder-url http://localhost:8546
```

## Core System Workflow

1. By default, `rollup-boost` forwards all JSON-RPC API calls from `proposer-op-node` to `proposer-op-geth`.
2. When `rollup-boost` receives an `engine_FCU` with attributes (initiating block building):
    - It relays the call to `proposer-op-geth` as usual.
    - If `builder-op-geth` is synced to the chain tip, the call is also multiplexed to it.
    - The FCU call returns a PayloadID, which should be identical for both `proposer-op-geth` and `builder-op-geth`.
3. When `rollup-boost` receives an `engine_getPayload`:
    - It first queries `proposer-op-geth` for a fallback block.
    - In parallel, it queries `builder-op-geth`.
4. Upon receiving the `builder-op-geth` block:
    - `rollup-boost` validates the block with `proposer-op-geth` using `engine_newPayload`.
    - This validation ensures the block will be valid for `proposer-op-geth`, preventing network stalls due to invalid blocks.
    - If the external block is valid, it is returned to the `proposer-op-node`.
5. As per its normal workflow, the `proposer-op-node` sends another `newPayload` request to the `rollup-boost` and another FCU(without) to update the state of its op-geth node.
    - In this case, the `rollup-boost` just relays the data and does not introspect anything.
    - Note that since we already tested `newPayload` on the proposer-op-geth in the previous step, this process should be cached. 

![Workflow Diagram](/assets/workflow.svg)

## License
The code in this project is free software under the [MIT License](/LICENSE).

---

Made with ☀️ by the ⚡🤖 collective.
