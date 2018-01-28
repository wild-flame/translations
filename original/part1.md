
# Diving Into The Ethereum VM



Solidity offers many high-level language abstractions, but these features make it hard to understand what’s really going on when my program is running. Reading the Solidity documentation still left me confused over very basic things.

*What are the differences between string, bytes32, byte[], bytes?*

* Which one do I use, when?

* What’s happening when I cast a string to bytes? Can I cast to byte[]?

* How much do they cost?

*How are mappings stored by the EVM?*

* Why can’t I delete a mapping?

* Can I have mappings of mappings? (Yes, but how does that work?)

* Why is there storage mapping, but no memory mapping?

*How does a compiled contract look to the EVM?*

* How is a contract created?

* What is a constructor, really?

* What is the fallback function?

I think it’s a good investment to learn how a high-level language like Solidity runs on the Ethereum VM (EVM). For couple of reasons.

1. *Solidity is not the last word.* Better EVM languages are coming. (Pretty please?)

1. *The EVM is a database engine.* To understand how smart contracts work in any EVM language, you have to understand how data is organized, stored, and manipulated.

1. *Know-how to be a contributor.* The Ethereum toolchain is still very early. Knowing the EVM well would help you make awesome tools for yourself and others.

1. *Intellectual challenge.* EVM gives you a good excuse to play at the intersection of cryptography, data structure, and programming language design.

In a series of articles, I’d like to deconstruct simple Solidity contracts in order to understand how it works as EVM bytecode.

An outline of what I hope to learn and write about:

* The basics of EVM bytecode.

* How different types (mappings, arrays) are represented.

* What is going on when a new contract is created.

* What is going on when a method is called.

* How the ABI bridges different EVM languages.

My final goal is to be able to understand a compiled Solidity contract in its entirety. Let’s start by reading some basic EVM bytecode!

This table of [EVM Instruction Set](https://gist.github.com/hayeah/bd37a123c02fecffbe629bf98a8391df) would be a helpful reference.

## A Simple Contract

Our first contract has a constructor and a state variable:

    // c1.sol
    pragma solidity ^0.4.11;

    contract C {
        uint256 a;

        function C() {
          a = 1;
        }
    }

Compile this contract with solc:

    $ solc --bin --asm c1.sol

    ======= c1.sol:C =======
    EVM assembly:
        /* "c1.sol":26:94  contract C {... */
      mstore(0x40, 0x60)
        /* "c1.sol":59:92  function C() {... */
      jumpi(tag_1, iszero(callvalue))
      0x0
      dup1
      revert
    tag_1:
    tag_2:
        /* "c1.sol":84:85  1 */
      0x1
        /* "c1.sol":80:81  a */
      0x0
        /* "c1.sol":80:85  a = 1 */
      dup2
      swap1
      sstore
      pop
        /* "c1.sol":59:92  function C() {... */
    tag_3:
        /* "c1.sol":26:94  contract C {... */
    tag_4:
      dataSize(sub_0)
      dup1
      dataOffset(sub_0)
      0x0
      codecopy
      0x0
      return
    stop

    sub_0: assembly {
            /* "c1.sol":26:94  contract C {... */
          mstore(0x40, 0x60)
        tag_1:
          0x0
          dup1
          revert

    auxdata: 0xa165627a7a72305820af3193f6fd31031a0e0d2de1ad2c27352b1ce081b4f3c92b5650ca4dd542bb770029
    }

    Binary:
    60606040523415600e57600080fd5b5b60016000819055505b5b60368060266000396000f30060606040525b600080fd00a165627a7a72305820af3193f6fd31031a0e0d2de1ad2c27352b1ce081b4f3c92b5650ca4dd542bb770029

The number 6060604052... is bytecode that the EVM actually runs.

## In Baby Steps

Half of the compiled assembly is boilerplate that’s similar across most Solidity programs. We’ll look at those later. For now, let’s examine the unique part of our contract, the humble storage variable assignment:

    a = 1

This assignment is represented by the bytecode 6001600081905550. Let’s break it up into one instruction per line:

    60 01
    60 00
    81
    90
    55
    50

The EVM is basically a loop that execute each instruction from top to bottom. Let’s annotate the assembly code (indented under the label tag_2) with the corresponding bytecode to better see how they are associated:

    tag_2:
      // 60 01
      0x1
      // 60 00
      0x0
      // 81
      dup2
      // 90
      swap1
      // 55
      sstore
      // 50
      pop

Note that 0x1 in the assembly code is actually a shorthand for push(0x1). This instruction pushes the number 1 onto the stack.

It still hard to grok what’s going on just staring at it. Don’t worry though, it’s simple to simulate the EVM line by line.

## Simulating The EVM

The EVM is a stack machine. Instructions might use values on the stack as arguments, and push values onto the stack as results. Let’s consider the operation add.

Assume that there are two values on the stack:

    [1 2]

When the EVM sees add, it adds the top 2 items together, and pushes the answer back onto the stack, resulting in:

    [3]

In what follows, we’ll notate the stack with []:

    // The empty stack
    stack: []
    // Stack with three items. The top item is 3. The bottom item is 1.
    stack: [3 2 1]

And notate the contract storage with {}:

    // Nothing in storage.
    store: {}
    // The value 0x1 is stored at the position 0x0.
    store: { 0x0 => 0x1 }

Let’s now look at some real bytecode. We’ll simulate the bytecode sequence 6001600081905550 as EVM would, and print out the machine state after each instruction:

    // 60 01: pushes 1 onto stack
    0x1
      stack: [0x1]

    // 60 00: pushes 0 onto stack
    0x0
      stack: [0x0 0x1]

    // 81: duplicate the second item on the stack
    dup2
      stack: [0x1 0x0 0x1]

    // 90: swap the top two items
    swap1
      stack: [0x0 0x1 0x1]

    // 55: store the value 0x1 at position 0x0
    // This instruction consumes the top 2 items
    sstore
      stack: [0x1]
      store: { 0x0 => 0x1 }

    // 50: pop (throw away the top item)
    pop
      stack: []
      store: { 0x0 => 0x1 }

The end. The stack is empty, and there’s one item in storage.

What’s worth noting is that Solidity had decided to store the state variable uint256 a at the position 0x0. It's perfectly possible for other languages to choose to store the state variable elsewhere.

In pseudocode, what the EVM does for 6001600081905550 is essentially:

    // a = 1
    sstore(0x0, 0x1)

Looking carefully, you’d see that the dup2, swap1, pop are superfluous. The assembly code could be simpler:

    0x1
    0x0
    sstore

You could try to simulate the above 3 instructions, and satisfy yourself that they indeed result in the same machine state:

    stack: []
    store: { 0x0 => 0x1 }

## Two Storage Variables

Let’s add one extra storage variable of the same type:

    // c2.sol
    pragma solidity ^0.4.11;

    contract C {
        uint256 a;
        uint256 b;

        function C() {
          a = 1;
          b = 2;
        }
    }

Compile, focusing on tag_2:

    $ solc --bin --asm c2.sol

    // ... more stuff omitted

    tag_2:
        /* "c2.sol":99:100  1 */
      0x1
        /* "c2.sol":95:96  a */
      0x0
        /* "c2.sol":95:100  a = 1 */
      dup2
      swap1
      sstore
      pop
        /* "c2.sol":112:113  2 */
      0x2
        /* "c2.sol":108:109  b */
      0x1
        /* "c2.sol":108:113  b = 2 */
      dup2
      swap1
      sstore
      pop

The assembly in pseudocode:

    // a = 1
    sstore(0x0, 0x1)
    // b = 2
    sstore(0x1, 0x2)

What we learn here is that the two storage variables are positioned one after the other, with a in position 0x0 and b in position 0x1.

## Storage Packing

Each slot storage can store 32 bytes. It’d be wasteful to use all 32 bytes if a variable only needs 16 bytes. Solidity optimizes for storage efficiency by packing two smaller data types into one storage slot if possible.

Let’s change a and b so they are only 16 bytes each:

    pragma solidity ^0.4.11;

    contract C {
        uint128 a;
        uint128 b;

        function C() {
          a = 1;
          b = 2;
        }
    }

Compile the contract:

    $ solc --bin --asm c3.sol

The generated assembly is now more complex:

    tag_2:
      // a = 1
      0x1
      0x0
      dup1
      0x100
      exp
      dup2
      sload
      dup2
      0xffffffffffffffffffffffffffffffff
      mul
      not
      and
      swap1
      dup4
      0xffffffffffffffffffffffffffffffff
      and
      mul
      or
      swap1
      sstore
      pop

      // b = 2
      0x2
      0x0
      0x10
      0x100
      exp
      dup2
      sload
      dup2
      0xffffffffffffffffffffffffffffffff
      mul
      not
      and
      swap1
      dup4
      0xffffffffffffffffffffffffffffffff
      and
      mul
      or
      swap1
      sstore
      pop

The above assembly code packs these two variables together in one storage position (0x0), like this:

    [         b         ][         a         ]
    [16 bytes / 128 bits][16 bytes / 128 bits]

The reason to pack is because the most expensive operations by far are storage usage:

* sstore costs 20000 gas for first write to a new position.

* sstore costs 5000 gas for subsequent writes to an existing position.

* sload costs 500 gas.

* Most instructions costs 3~10 gases.

By using the same storage position, Solidity pays 5000 for the second store variable instead of 20000, saving us 15000 in gas.

## More Optimization

Instead of storing a and b with two separate sstore instructions, it should be possible to pack the two 128 bits numbers together in memory, then store them using just one sstore, saving an additional 5000 gas.

You can ask Solidity to make this optimization by turning on the optimize flag:

    $ solc --bin --asm --optimize c3.sol

Which produces assembly code that uses just one sload and one sstore:

    tag_2:
        /* "c3.sol":95:96  a */
      0x0
        /* "c3.sol":95:100  a = 1 */
      dup1
      sload
        /* "c3.sol":108:113  b = 2 */
      0x200000000000000000000000000000000
      not(sub(exp(0x2, 0x80), 0x1))
        /* "c3.sol":95:100  a = 1 */
      swap1
      swap2
      and
        /* "c3.sol":99:100  1 */
      0x1
        /* "c3.sol":95:100  a = 1 */
      or
      sub(exp(0x2, 0x80), 0x1)
        /* "c3.sol":108:113  b = 2 */
      and
      or
      swap1
      sstore

The bytecode is:

    600080547002000000000000000000000000000000006001608060020a03199091166001176001608060020a0316179055

And formatting the bytecode to one instruction per line:

    // push 0x0
    60 00
    // dup1
    80
    // sload
    54
    // push17 push the the next 17 bytes as a 32 bytes number
    70 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

    /* not(sub(exp(0x2, 0x80), 0x1)) */
    // push 0x1
    60 01
    // push 0x80 (32)
    60 80
    // push 0x80 (2)
    60 02
    // exp
    0a
    // sub
    03
    // not
    19

    // swap1
    90
    // swap2
    91
    // and
    16
    // push 0x1
    60 01
    // or
    17

    /* sub(exp(0x2, 0x80), 0x1) */
    // push 0x1
    60 01
    // push 0x80
    60 80
    // push 0x02
    60 02
    // exp
    0a
    // sub
    03

    // and
    16
    // or
    17
    // swap1
    90
    // sstore
    55

There are four magic values used in the assembly code:

* 0x1 (16 bytes), using lower 16 bytes

    // Represented as 0x01 in bytecode

    16:32 0x00000000000000000000000000000000
    00:16 0x00000000000000000000000000000001

* 0x2 (16 bytes), using higher 16bytes

    // Represented as 0x200000000000000000000000000000000 in bytecode

    16:32 0x00000000000000000000000000000002
    00:16 0x00000000000000000000000000000000

* not(sub(exp(0x2, 0x80), 0x1))

    // Bitmask for the upper 16 bytes

    16:32 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    00:16 0x00000000000000000000000000000000

* sub(exp(0x2, 0x80), 0x1)

    // Bitmask for the lower 16 bytes

    16:32 0x00000000000000000000000000000000 
    00:16 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF

The code does some bits-shuffling with these values to arrive at the desired result:

    16:32 0x00000000000000000000000000000002 
    00:16 0x00000000000000000000000000000001

Finally, this 32bytes value is stored at position 0x0.

### Gas Usage
> 60008054700**200000000000000000000000000000000**6001608060020a03199091166001176001608060020a0316179055

Notice that 0x200000000000000000000000000000000 is embedded in the bytecode. But the compiler could’ve also chosen to calculate the value with the instructions exp(0x2, 0x81), which results in shorter bytecode sequence.

But it turns out that 0x200000000000000000000000000000000 is a cheaper than exp(0x2, 0x81). Let's look at the gas fees involved:

* 4 gas paid for every zero byte of data or code for a transaction.

* 68 gas for every non-zero byte of data or code for a transaction.

Let’s compare how much either representation costs in gas.

* The bytecode 0x200000000000000000000000000000000. It has many zeroes, which are cheap.

(1 * 68) + (32 * 4) = 196.

* The bytecode 608160020a. Shorter, but no zeroes.

5 * 68 = 340.

The longer sequence with more zeroes is actually cheaper!

## Summary

An EVM compiler doesn’t exactly optimize for bytecode size or speed or memory efficiency. Instead, it optimizes for gas usage, which is an layer of indirection that incentivizes the sort of calculation that the Ethereum blockchain can do efficiently.

We’ve seen some quirky aspects of the EVM:

* EVM is a 256bit machine. It is most natural to manipulate data in chunks of 32 bytes.

* Persistent storage is quite expensive.

* The Solidity compiler makes interesting choices in order to minimize gas usage.

Gas costs are set somewhat arbitrarily, and could well change in the future. As costs change, compilers would make different choices.

In this article series about the EVM I write about:

* [Introduction to the EVM assembly code.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-6e8d5d2f3c30)

* [How fixed-length data types are represented.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-part-2-storage-layout-bc5349cb11b7)

* [How dynamic data types are represented.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-the-hidden-costs-of-arrays-28e119f04a9b)

* [How ABI Encodes External Method Calling.](https://medium.com/@hayeah/how-to-decipher-a-smart-contract-method-call-8ee980311603)

* What is going on when a new contract is created.

If you’d like to learn more about the EVM, subscribe to my weekly tutorial:

![](https://cdn-images-1.medium.com/max/2316/1*T8-ijp9Q6w_0hc1kMRXq7Q.jpeg)
