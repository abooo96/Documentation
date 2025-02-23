(opcodes)=
EVM Opcode Hooks
================

## Background
The EVM's instruction set is tightly integrated with the specifics of the EVM environment.
CVL, which was designed to be as general-purpose as possible, with strong decoupling between the environment of the source programs and the spec language, does not allow controlling all aspects of the EVM environment.
In particular, CVL allows direct access to an environment `env` object for accessing the following fields mapped to their corresponding EVM instructions:

| CVL `env` field   | EVM instruction   |
| ----------------- | ----------------- |
| `msg.value`       | `CALLVALUE`       |
| `msg.sender`      | `CALLER`          |
| `block.number`    | `NUMBER`          |
| `block.timestamp` | `TIMESTAMP`       |

The expressivity of specs may be limited since other instructions, such as `CHAINID` are not directly accessible from the `env` object.
However, it is undesirable to expand the CVL `env` type with too many fields. For instance, the `CHAINID` instruction would rarely return a different value between two Solidity calls to the same contract, even if they take two different environment variables.
Additionally, the scope of an `env` variable is a single function call, though it captures variables that are not necessarily scoped to a function call. `CHAINID` is not expected to change between two function calls.
In particular, the EVM instruction set is dynamic, and new instructions (opcodes) are added and removed occasionally.

Consequently, hooks help us define specialized behavior for every EVM instruction that accesses the virtual machine's internals in some non-trivial way.


## Hooking on opcodes
Currently, we support the following hook opcodes, named the same as their EVM instruction counterparts (with the exception of `CREATE1`, as seen below):
- `ADDRESS`
- `CALLER`
- `CALLVALUE`
- `ORIGIN`
- `BALANCE`
- `SELFBALANCE`
- `NUMBER`
- `TIMESTAMP`
- `GASPRICE`
- `GASLIMIT`
- `GAS`
- `COINBASE`
- `DIFFICULTY`
- `BASEFEE`
- `MSIZE`
- `CHAINID`
- `CODESIZE`
- `CODECOPY`
- `EXTCODESIZE`
- `EXTCODECOPY`
- `EXTCODEHASH`
- `BLOCKHASH`
- `CALL`
- `CALLCODE`
- `DELEGATECALL`
- `STATICCALL`
- `LOG0`
- `LOG1`
- `LOG2`
- `LOG3`
- `LOG4`
- `CREATE1` (for the `CREATE` opcode)
- `CREATE2`
- `REVERT`
- `SELFDESTRUCT`


The pattern for each hook follows the arguments that are accepted by the instructions they model. 
Each hook is followed by a command block `{ ... }` where the instrumentation spec code is provided.
Some hooks have output values, while others do not. Those that have an output value specify the type and name of a variable to bind to the output value after listing the arguments.
Most hooks are applied _after_ the appearance and execution of the instruction they model.
The only hooks that are applied _before_ are those for halting instructions such as `REVERT` and `SELFDESTRUCT`.

Below is the syntax for all opcode hook types, without command block braces:
```cvl
hook ADDRESS address v

hook BALANCE(address addr) uint v

hook ORIGIN address v

hook CALLER address v

hook CALLVALUE uint v

hook CODESIZE uint v

hook CODECOPY(uint destOffset, uint offset, uint length)

hook GASPRICE uint v

hook EXTCODESIZE(address addr) uint v

hook EXTCODECOPY(address b, uint destOffset, uint offset, uint length)

hook EXTCODEHASH(address a) bytes32 hash

hook BLOCKHASH(uint n) bytes32 hash

hook COINBASE address v

hook TIMESTAMP uint v

hook NUMBER uint v

hook DIFFICULTY uint v

hook GASLIMIT uint v

hook CHAINID uint v

hook SELFBALANCE uint v

hook BASEFEE uint v

hook MSIZE uint v

hook GAS uint v

hook LOG0(uint offset, uint length)

hook LOG1(uint offset, uint length, bytes32 t1)

hook LOG2(uint offset, uint length, bytes32 t1, bytes32 t2)

hook LOG3(uint offset, uint length, bytes32 t1, bytes32 t2, bytes32 t3)

hook LOG4(uint offset, uint length, bytes32 t1, bytes32 t2, bytes32 t3, bytes32 t4)

hook CREATE1(uint value, uint offset, uint length) address v

hook CREATE2(uint value, uint offset, uint length, bytes32 salt) address v 

hook CALL(uint g, address addr, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc

hook CALLCODE(uint g, address addr, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc

hook DELEGATECALL(uint g, address addr, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc

hook STATICCALL(uint g, address addr, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint rc

hook REVERT(uint offset, uint size)

hook SELFDESTRUCT(address a)
```

```{note}
The hooks are applied to _all_ contracts, not just the main contract under verification.
```

(rawhooks)=
## Hooking on raw reads and writes to storage
While it is already possible to hook on storage fields using storage patterns,
there are cases where we do not know the storage layout or prefer to catch all storage accesses.
For this use case one can use the `ALL_SLOAD` and `ALL_SSTORE` hooks to respectively hook on arbitrary storage loads or stores.

For example, if we wish to capture reads and writes to a storage field that is assumed to be of type address
in slot 0, we can write the following to have the `myAddress` ghost track the value at slot 0:
```cvl
ghost address myAddress;

hook ALL_SLOAD(uint loc) uint v {
    if (loc == 0) {
        require to_mathint(myAddress) == to_mathint(v);
    }
}


hook ALL_SSTORE(uint loc, uint v) {
    if (loc == 0) {
        havoc myAddress assuming to_mathint(myAddress@new) == to_mathint(myAddress);
    }
```

We can even have both a storage-pattern based hook and a raw storage hook.
In case they both apply on a storage access, the storage-pattern one will be applied first, 
and the raw storage hook will be applied second.
For example, if we add to the example from above:
```cvl
hook Sstore field0 /* made-up name of the field in slot 0 */ address newValue STORAGE {
    havoc myAddress assuming to_mathint(myAddress@new) == to_mathint(newValue)+5;
}
```
The value of `myAddress` will be the new value written to the field at slot 0 `newValue`, and not necessarily the address of `newValue+5`.
In any case both hooks are executed, so any other effects of the storage-pattern hook will still be visible.

```{note}
One optimization done by the Prover is automatic unpacking of packed storage variables.
As this can interfere with the raw reading of storage slots, it has to be disabled by specifying
`--prover_args "-enableStorageSplitting false"`
```

```{note}
Just like the usual opcode hooks, the raw storage hooks are applied on all contracts.
This means that a storage access on _any_ contract will trigger the hook.
Therefore, in a rule that models calls to multiple contracts,
if two contracts are accessing the same slot the same hook code will be called with the same `loc` value.
```

## Missing instructions
The standard stack-manipulating instructions `DUP*`, `SWAP*`, `PUSH*` and `POP` are not modeled.
`MLOAD` and `MSTORE` are also not modeled.

## Known inter-dependencies and common pitfalls
Hooks are instrumented for every appearance of the matching instruction in the bytecode, as generated by a high-level source compiler such as the Solidity compiler. 
The behavior of the bytecode may sometimes be surprising. 
For example, every Solidity method call to a non-payable function will contain an early call to `CALLVALUE` to check that it is 0. It means that every time a non-payable Solidity function is invoked, the `CALLVALUE` hook will be triggered.
