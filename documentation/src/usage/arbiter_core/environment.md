# Environment
The `Environment` owns a `revm` instance for processing EVM bytecode.
To make the `Environment` performant and flexible, it runs on its own system thread and receives all communication via `Instruction`s sent to it via a `Sender<Instruction>`.
The `Socket` is a struct owned by the `Environment` that manages all inward and outward communication with the `Environment`'s clients, such as the `Instruction` channel.

## Usage
To create an `Environment`, we use a builder pattern that allows you to pre-load an `Environment` with your own database.
We can do the following to create a default `Environment`:
```rust
use arbiter_core::environment::EnvironmentBuilder;

fn main() {
    let env = EnvironmentBuilder::new().build();
}
```
Note that the call to `.build()` will start the `Environment`'s thread and begin processing `Instruction`s.

If you have a database that has been forked from a live network, it has likely been serialized to disk.
In which case, you can do something like this:
```rust, ignore
use arbiter_core::environment::EnvironmentBuilder;
use arbiter_core::environment::fork::Fork;

fn main() {
    let path_to_fork = "path/to/fork";
    let fork = Fork::from_disk(path_to_fork).unwrap();
    let env = EnvironmentBuilder::new().with_db(fork).build();
}
```
This will create an `Environment` that has been forked from the database at the given path and is ready to receive `Instruction`s.

`Environment` supports more customization for the `gas_limit` and `contract_size_limit` of the `revm` instance. 
You can do the following:
```rust
use arbiter_core::environment::EnvironmentBuilder;

fn main() {
    let env = EnvironmentBuilder::new()
        .with_gas_limit(revm::primitives::U256::from(12_345_678))
        .with_contract_size_limit(111_111)
        .build();
}
```

## Instructions
`Instruction`s have been added to over time, but at the moment we allow for the following:
- `Instruction::AddAccount`: Add an account to the `Environment`'s world state. This is usually called by the `RevmMiddleware` when a new client is created.
- `Instruction::BlockUpdate`: Update the `Environment`'s block number and block timestamp. This can be handled by an external agent in a simulation, if desired.
- `Instruction::Cheatcode`: Execute one of the `Cheatcodes` on the `Environment`'s world state. 
The `Cheatcodes` include:
    - `Cheatcodes::Deal`: Used to set the raw ETH balance of a user. Useful when you need to pay gas fees in a transaction.
    - `Cheatcodes::Load`: Gets the value of a storage slot of an account. 
    - `Cheatcodes::Store`: Sets the value of a storage slot of an account.
    - `Cheatcodes::Access`: Gets the account at an address.
- `Instruction::Query`: Allows for querying the `Environment`'s world state and current configuration. Anything in the `EnvironmentData` enum is accessible via this instruction.
    - `EnvironmentData::BlockNumber`: Gets the current block number of the `Environment`.
    - `EnvironmentData::BlockTimestamp`: Gets the current block timestamp of the `Environment`.
    - `EnvironmentData::GasPrice`: Gets the current gas price of the `Environment`.
    - `EnvironmentData::Balance`: Gets the current ETH balance of an account.
    - `EnvironmentData::TransactionCount`: Gets the current nonce of an account.
- `Instruction::Stop`: Stops the `Environment`'s thread and echos out to any listeners to shut down their event streams. This can be used when handling errors or reverts, or just when you're done with the `Environment`.
- `Instruction::Transaction`: Executes a transaction on the `Environment`'s world state. This is usually called by the `RevmMiddleware` when a client sends a ETH-call or state-changing transaction.

The `RevmMiddleware` provides methods for sending the above instructions to an associated `Environment` so that you do not have to interact with the `Environment` directly!

## Events
The `Environment` also emits Ethereum events and errors/reverts to clients who are set to listen to them. 
To do so, we use a `tokio::sync::broadcast` channel and the `RevmMiddleware` manages subscriptions to these events.
As for errors or reverts, we are working on making the flow of handling these more graceful so that your own program or agents can decide how to handle them.