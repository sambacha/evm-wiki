# evm-wiki

#### [access the wiki](https://github.com/sambacha/evm-wiki/wiki)

### Contents

- [General Machine Info](#general-machine-info)
  - [Jumps](#jumps)
  - [Exceptions](#exceptions)
  - [Fees](#fees)
  - [Execution Environment](#execution-environment)
  - [Execution](#execution)
- [Instructions](#instructions)
  - [Definitions](#definitions)
  - [Arithmetic Operations](#arithmetic-operations)
  - [Comparisons and Bitwise Logic](#comparisons-and-bitwise-logic)
  - [SHA3](#sha3)
  - [Environment Information](#environment-information)
  - [Block Information](#block-information)
  - [Stack, Memory, Storage, and Flow Operations](#stack-memory-storage-and-flow-operations)
  - [Stack Manipulation](#stack-manipulation)
  - [Logging](#logging)
  - [System Operations](#system-operations)

---

## General Machine Info

* word size: 256 bits
* volatile memory, denoted **µ**.
* persistent state/storage (on the blockchain), denoted **σ**.

All memory locations are initially zero. Program code is stored in a ROM, not easily accessible


### Jumps

A JUMP or JUMPI instruction sets the program counter and therefore starts execution off again from some newly-specified instruction.
These must jump to some explicit JUMPDEST instruction, that is not an argument literal of another instruction. 


### Exceptions

There is only one type of exception. If one occurs, return to the state prior to the current execution, but consume all gas. Exceptions may occur in any of the following circumstances:

* There is insufficient remaining gas to cover the cost of an instruction;
* An undefined instruction is to be executed;
* A jump to an invalid destination occurs.
* The stack size would exceed 1024 if the next instruction were to be executed


### Fees
Gas fees may be charged for the following reasons:

* Intrinsic fees charged for computation performed by each instruction
* gas may be deducted to pay for a subordinate contract call or creation
* gas may be paid to compensate for extra memory usage


### Execution Environment

The currently executing contract has access to a tuple **I**, with the following fields:

* **I_a** : the address of the account owning the executing code
* **I_o** : the address of the transaction originating this execution
* **I_p** : the gas price in the originating transaction
* **I_d** : a byte array of the input data
* **I_s** : sender address
* **I_v** : the value (in wei) passed to this account
* **I_b** : the byte array containing the machine code to be executed
* **I_H** : the current block header
* **I_e** : the current contract stack depth (number of CALL and CREATE instructions that have been executed)

The memory state **µ**, of the currently running contract, is itself subdivided thus:

* **µ_s** : the stack, holds up to 1024 words; initially empty
* **µ_m** : memory contents, initially implicitly 2^256 bytes of zeroes (word-addressed byte array, words stored big-endian);
* **µ_g** : remaining gas
* **µ_pc** : The program counter (a positive 256 bit integer)
* **µ_i** : the size of the active memory region in words


We also have a substate A, maintained during execution, which contains the following fields (initially empty or zero):

* **A_s** : The suicide set. Accounts to be discarded following transaction completion.
* **A_l** : Log series. archived and indexable "checkpoints" that help contract calls to be tracked.
* **A_r** : Refund balance, accrued by setting nonzero values in **σ** to zero, using the SSTORE instruction.


### Execution

At each instruction, gas fees are charged, state is update, and the program counter increments (or jumps).

---

## Instructions

### Definitions

* *μ* : the EVM's memory; by a slight overloading of terms, this shall mean the same thing as **µ_m**, above. If a value *v* is said to be saved to memory at address *α*, this means *μ*[*α*] <- *v*. If *v* is a word, it is saved in big-endian order in the address range [*α*, *α* + 31]. 
* *τ* : the EVM's execution stack; the same thing as **µ_s**, above. We have *τ*[0] the top stack element, and *τ*[k] the element k from the top. If an instruction pops m values off the stack and pushes n values onto it, this behaviour will be denoted -m+n in the table. The size of the stack after an operation is then (n - m). The result of operations is usually pushed to the stack.
* *ρ* : the persistent storage available to the executing account. This is equivalent to **σ**[**I_a**].
* **I_d** : a byte array of extra data that may have been included with the transaction as input to the currently running contract.
* **I_b** : the bytecode of the currently running contract.
* wei : the smallest denomination of the ether currency ether. 10^18 wei constitutes one ether. All ether values will be given in wei.

Behaviour will mostly be given approximately, and in English. See the yellow paper for precise definitions.

### Arithmetic Operations

| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0x00 | STOP | Halts Execution. | -0+0 | 0 |
| 0x01 | ADD | Sums two numbers. | -2+1 | 3 |
| 0x02 | MUL | Multiplies two numbers. | -2+1 | 5 |
| 0x03 | SUB | Difference: *τ*[0] - *τ*[1]. | -2+1 | 3 |
| 0x04 | DIV | Integer floor division resulting from *τ*[0] / *τ*[1]. Division by zero produces a zero result. | -2+1 | 5 |
| 0x05 | SDIV | Signed division. Fractional parts are truncated towards zero. | -2+1 | 5 |
| 0x06 | MOD | Modulo: *τ*[0] % *τ*[1]. Zero modulus produces a zero result. | -2+1| 5 |
| 0x07 | SMOD | Signed modulo. The sign is that of *τ*[0]. | -2+1 | 5 |
| 0x08 | ADDMOD | Modular addition: (*τ*[0] + *τ*[1]) % *τ*[2]. | -3+1 | 8 |
| 0x09 | MULMOD | Modular multiplication: (*τ*[0] * *τ*[1]) % *τ*[2]. | -3+1 | 8 |
| 0x0a | EXP | Exponentiation: *τ*[0] ^ *τ*[1]. | -2+1 | 10 * log(*τ*[1]) |
| 0x0b | SIGNEXTEND | Extend the sign bit of a smaller integer up to 32 bytes. Let **b** be the high bit of the *τ*[0]'th byte of *τ*[1]. Set all bits in the result more significant than **b** equal to **b**. The remaining bits are copied from *τ*[1]. | -2+1 | 5 |

### Comparisons and Bitwise Logic

In what follows, truth is represented by 1, falsehood by 0.

| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0x10 | LT | Unsigned less-than relation: *τ*[0] < *τ*[1] | -2+1 | 3 |
| 0x11 | GT | Unsigned greater-than relation: *τ*[0] > *τ*[1] | -2+1 | 3 |
| 0x12 | SLT | Signed less-than relation: *τ*[0] < *τ*[1] | -2+1 | 3 |
| 0x13 | SGT | Signed greater-than relation: *τ*[0] > *τ*[1] | -2+1 | 3 |
| 0x14 | EQ | equality: *τ*[0] = *τ*[1] | -2+1 | 3 |
| 0x15 | ISZERO | Logical negation. 1 if *τ*[0] is 0, 0 otherwise. | -1+1 | 3 |
| 0x16 | AND | Bitwise AND. | -2+1 | 3 |
| 0x17 | OR | Bitwise OR. | -2+1 | 3 |
| 0x18 | XOR | Bitwise XOR. | -2+1 | 3 |
| 0x19 | NOT | Bitwise NOT. | -1+1 | 3 |
| 0x1a | BYTE | Retrieve the *τ*[0]'th byte from *τ*[1]. Zero if *τ*[0] is greater than 31.  | -2+1 | 3 |


### SHA3

| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0x20 | SHA3 | Calculate the hash of the memory region from address *τ*[0] up to the address (*τ*[0] + *τ*[1] - 1). | -2+1 | 30 + 6 * (*τ*[1] / 32) |



### Environment Information

| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0x30 | ADDRESS | Get the address of the currently executing contract. | -0+1 | 2 |
| 0x31 | BALANCE | Get the balance of the account at address *τ*[0]. Zero if there is no such account. | -1+1 | 20 |
| 0x32 | ORIGIN | Get the externally-controlled account which originated the current execution. That is, the root of the contract call tree. | -0+1 | 2 |
| 0x33 | CALLER | Get the address of the account that's directly responsible for the current transaction. | -0+1 | 2 |
| 0x34 | CALLVALUE | Get the value of ether sent with this transaction. | -0+1 | 2 |
| 0x35 | CALLDATALOAD | Fetch the word *τ*[0] from the input data **I_d** of this transaction, and place it on the stack. Out-of-range values are zero. | -1+1 | 3 |
| 0x36 | CALLDATASIZE | The length of **I_d** in bytes. | -0+1 | 2 |
| 0x37 | CALLDATACOPY | Copy *τ*[2] bytes from **I_d**[*τ*[1]] into *μ*[*τ*[0]]. Out-of-range values are zero. | -3+0 | 3 + 3*(*τ*[2] / 32) |
| 0x38 | CODESIZE | Get the size of the currently-running code. | -0+1 | 2 |
| 0x39 | CODECOPY | Copy *τ*[2] bytes from **I_b**[*τ*[1]] into *μ*[*τ*[0]]. Out-of-range values are STOP. | -3+0 | 3 + 3*(*τ*[2] / 32) |
| 0x3a | GASPRICE | Get the transaction gas price | -0+1 | 2 |
| 0x3b | EXTCODESIZE | Get the size of the code in the account at the address *τ*[0]. | -1+1 | 20 |
| 0x3c | EXTCODECOPY | Let **I_c** be the contract code of the account at address *τ*[0]. Copy *τ*[3] bytes from **I_c**[*τ*[2]], into *μ*[*τ*[1]]; Out-of-range values are interpreted as STOP. | -4+0 | 20 + 3*(*τ*[3]/32) |
| 0x3d | RETURNDATASIZE | Get the size of output data from the previous call from the current environment. | -0+1 | 2 |
| 0x3e | RETURNDATACOPY | Copy output data (**O**) from previous call to memory. Copies *τ*[2] bytes from **O**[*τ*[1]], into *μ*[*τ*[0]]; Out-of-range values are interpreted as 0. | -3+0 | 3 + 3*(*τ*[2] / 32) |



### Block Information

| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0x40 | BLOCKHASH | Return the hash of block *τ*[1], as long as it was one of the 256 most recent blocks. Zero otherwise. | -1+1 | 20 |
| 0x41 | COINBASE | Return the block beneficiary address (where the mining reward is sent). | -0+1 | 2 |
| 0x42 | TIMESTAMP | Return this block's timestamp. | -0+1 | 2 |
| 0x43 | NUMBER | Return this block's number. | -0+1 | 2 |
| 0x44 | DIFFICULTY | Return this block's difficulty. | -0+1 | 2 |
| 0x45 | GASLIMIT | Return this block's gas limit. | -0+1 | 2 |



### Stack, Memory, Storage, and Flow Operations

Any of the following instructions which access *μ* may increase **μ_i**, the size of the active memory region in words. At any time, **μ_i** is the highest address that has yet been accessed, divided by 32, more or less.


| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0x50 | POP | Remove an item from the stack. | -1+0 | 2 |
| 0x51 | MLOAD | Load the word from *μ* at the address *τ*[0] onto the stack.  | -1+1 | 3 |
| 0x52 | MSTORE | Save the word *τ*[1] to memory in the 32 bytes starting at address *τ*[0]. | -2+0 | 3 |
| 0x53 | MSTORE8 | Save the lowest byte of *τ*[1] to memory at address *τ*[0] | -2+0 | 3 |
| 0x54 | SLOAD | Load the word at address *τ*[0] from storage onto the stack. | -1+1 | 50 |
| 0x55 | SSTORE | Save a word to storage: *ρ*[*τ*[0]] = *τ*[1]. | -2+0 | 20000 if target cell value is set to nonzero from zero, 5000 otherwise. A refund of 15000 is credited to **A_r** whenever a nonzero cell is set to zero.  |
| 0x56 | JUMP | Jump execution to the address on the stack. | -1+0 | 8 |
| 0x57 | JUMPI | Jump to *τ*[0] if and only if *τ*[1] is not zero.  | -2+0 | 10 | 
| 0x58 | PC | Push the address of the last executed instruction to the stack.  | -0+1 | 2 | 
| 0x59 | MSIZE | Push 32 * **μ_i** to the stack. This is the size of the active memory region in *bytes*. | -0+1 | 2 | 
| 0x5a | GAS | Push to the stack the remaining gas after executing this instruction. | -0+1 | 2 | 
| 0x5b | JUMPDEST | Purely a destination for jumps. Otherwise a NOP. | -0+0 | 1 | 


### Stack Manipulation

PUSH instructions can push anywhere from 1 to 32 bytes onto the stack as a single word, aligned to the right. These values follow the PUSH immediately in the bytecode. Any out of range values are interpreted as zero.
DUP and SWAP instructions have versions for n between 1 and 16. Although the stack operations seem to depend on the value of n chosen, the effect does not, and the gas cost is constant. That is, the appropriate memory is accessed directly in implementations, rather than through a series of stack and memory operations.

| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0x60 + (n-1) | PUSHn | Push the next n bytes from code to the stack. | -0+1 | 3 |
| 0x80 + (n-1) | DUPn | Push a copy of *τ*[n-1] to the stack. | -n+(n+1) | 3 |
| 0x90 + (n-1) | SWAPn | Swap *τ*[0] with *τ*[n]. | -(n+1)+(n+1) | 3 |


### Logging

Each LOGn instruction, **n** between 0 and 4, appends a log entry to the substate's log series. 
A log contains: 

* The address of the logging account;
* a topic series containing the **n** words on the stack following *τ*[1];
* a data blob copied from the memory region between *τ*[0] and (*τ*[0] + *τ*[1] - 1). So *τ*[1] is the number of bytes to store.

| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0xa0 + n | LOGn | Add a log record with n topics. | -(n+2)+0 | (8 * *τ*[1]) + ((n + 1) * 375) |


### System Operations
| **Value** | **Name** | **Behaviour** | **Stack** | **Gas Cost** |
|-----------|----------|---------------|-----------|--------------|
| 0xf0 | CREATE | Create a new contract. *τ*[0] is the Ethereum endowment of the new contract. Pass a *τ*[2]-byte-long data block starting at memory location *τ*[1] to the new contract as input. Push to the stack the address of the new contract, or else 0 if creation failed. | -3+1 | 32000 |
| 0xf1 | CALL | *τ*[0]: gas quantity to provide for execution. *τ*[1]: address of the contract to call. *τ*[2]: value of ether to send. Send along *τ*[4] bytes of memory starting at the address *τ*[3]. Target contract returns its result into the *τ*[6] byte memory region starting at *τ*[5]. Push a 0 on the stack if the call failed, or else 1. | -7+1 | 40 + 9000 (if some ether is sent) + 25000 (if the target account doesn't exist) + *τ*[0] |
| 0xf2 | CALLCODE | The same as CALL, except execute the code of the current contract instead of the code of the receiving contract. | -7+1 | As CALL. |
| 0xf3 | RETURN | Halt and return the *τ*[1] bytes of memory from *μ*[*τ*[0]]. | -2+0 | 0 |
| 0xf4 | DELEGATECALL | The same as CALLCODE, except the sender and value of the parent contract are propagated directly to the called contract. So the executing contract is acting simply as a relay between its parent and its child. | -6+1 | As CALL. |
| 0xfa | STATICCALL | The same as CALL, except the value of ether to be sent is 0, and the other arguments are all shifted down one position, and any state-changing operations made during the sub-exection will throw an exception. | -6+1 | ? |
| 0xfd | REVERT | Halt execution, reverting state changes, refunding remaining gas, and returning the bytes of memory as in RETURN. | -2+0 | 0 |
| 0xfe | INVALID | Designated invalid instruction. Immediately halts, consuming all remaining gas. | -0+0 | 0 |
| 0xff | SELFDESTRUCT | Halt execution, and add the account to the suicide list. Transfer the balance in this account to the address *τ*[0].  | -1+0 | 0 gas cost. Refund 24000 gas if the contract is not already in the suicide list.  |


#### license

CC0-1.0
