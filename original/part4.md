
# How To Decipher A Smart Contract Method Call

Diving Into The Ethereum VM Part 4

![](https://cdn-images-1.medium.com/max/2000/1*pXqWf7UlmWdSn6xuFCrRSg.jpeg)

In previous articles of this series we’ve seen how Solidity represents complex data structures in the EVM storage. But data is useless if there’s no way to interact with it. The Smart Contract is the mediator between data and the outside world.

In this article we’ll see how Solidity and EVM makes it possible for external programs to call a contract’s methods and cause its state to change.

The “external program” is not limited to DApp/JavaScript. Any program that can communicate with an Ethereum node using HTTP RPC can interact with any contract deployed on the blockchain by creating transactions.

Creating a transaction is like making an HTTP request. A web server would accept your HTTP request and make changes to the database. A transaction would be accepted by the network, and the underlying blockchain extended to include the state changes.

Transactions are to Smart Contracts as HTTP requests are to web services.

If EVM assembly and Solidity data representation are unfamiliar, see previous articles of this series to learn more:

* [Introduction to the EVM assembly code.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-6e8d5d2f3c30)

* [How fixed-length data types are represented.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-part-2-storage-layout-bc5349cb11b7)

* [How dynamic data types are represented.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-the-hidden-costs-of-arrays-28e119f04a9b)

## Contract Transaction

Let’s look at a transaction that sets a state variable to 0x1. The contract we want to interact with has a setter and a getter for the variable a:

    pragma solidity ^0.4.11;

    contract C {
      uint256 a;

      function setA(uint256 _a) {
        a = _a;
      }

      function getA() returns(uint256) {
        return a;
      }
    }

This contract is deployed on the test network Rinkeby. Feel free to inspect it using Etherscan at the address [0x62650ae5…](https://rinkeby.etherscan.io/address/0x62650ae5c5777d1660cc17fcd4f48f6a66b9a4c2#code).

I’ve created a transaction that makes the call setA(1). Inspect this transaction at the address [0x7db471e5...](https://rinkeby.etherscan.io/tx/0x7db471e5792bbf38dc784a5b983ee6a7bbe3f1db85dd4daede9ee88ed88057a5).

![](https://cdn-images-1.medium.com/max/2106/1*IkWyWsud_E7DD6QfpPoKHQ.jpeg)

The transaction’s input data is:

    0xee919d500000000000000000000000000000000000000000000000000000000000000001

To the EVM, this is just 36 bytes of raw data. It is passed to the Smart Contract unprocessed as calldata. If the Smart Contact is a Solidity program, then it interprets these input bytes as a method call, and executes the appropriate assembly code for setA(1).

The input data can be broken down to two subparts:

    # The method selector (4 bytes)
    0xee919d5
    # The 1st argument (32 bytes)
    00000000000000000000000000000000000000000000000000000000000000001

The first four bytes is the method selector. The rest of the input data are method arguments in chunks of 32 bytes. In this case there is only 1 argument, the value0x1.

The method selector is the kecccak256 hash of the method signature. In this case the method signature is setA(uint256), which is the name of the method and the types of its arguments.

Let’s calculate the method selector in Python. First, hash the method signature:

    # Install pyethereum [https://github.com/ethereum/pyethereum/#installation](https://github.com/ethereum/pyethereum/#installation)
    > from ethereum.utils import sha3
    > sha3("setA(uint256)").hex()
    'ee919d50445cd9f463621849366a537968fe1ce096894b0d0c001528383d4769'

Then take the first 4 bytes of the hash:

    > sha3("setA(uint256)")[0:4].hex()
    'ee919d50'

## The Application Binary Interface (ABI)

As far as the EVM is concerned, the transaction’s input data (calldata) is just a sequence of bytes. The EVM doesn’t have builtin support for calling methods.

A smart contract can choose to simulate a method call by processing the input data in a structured way, as shown in the previous section.

If languages on the EVM all agree on how input data should be interpreted, then they can easily interoperate with each other. The [Contract Application Binary Interface](https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI#formal-specification-of-the-encoding) (The ABI) specifies a common encoding scheme.

We’ve seen how the ABI encodes a simple method call like setA(1). In later sections we’ll see how method calls with more complex arguments are encoded.

## Calling A Getter

If the method you are calling changes the state, then the entire network has to agree. This would requires a transaction, and costs you gas.

A getter method like getA() doesn’t change anything. Instead of asking the whole network to carry out the computation, we can send the method call to a local Ethereum node. An eth_call RPC request allows you to simulate a transaction locally. This is useful for read-only method or gas usage estimation.

An eth_call is like a cached HTTP GET request.

* It doesn’t change the global consensus state.

* The local blockchain (“cache”) may be slightly outdated.

Let’s make an eth_call to invoke the getA method, getting the state a in return. First, calculate the method selector:

    >>> sha3("getA()")[0:4].hex()
    'd46300fd'

Since there is no argument, the input data is just the method selector by itself. We can send an eth_call request to any Ethereum node. For this example, we will send the request to a public Ethereum node hosted by infura.io:

    $ curl -X POST \
    -H "Content-Type: application/json" \
    "[https://rinkeby.infura.io/YOUR_INFURA_TOKEN](https://rinkeby.infura.io/YOUR_INFURA_TOKEN)" \
    --data '
    {
      "jsonrpc": "2.0",
      "id": 1,
      "method": "eth_call",
      "params": [
        {
          "to": "0x62650ae5c5777d1660cc17fcd4f48f6a66b9a4c2",
          "data": "0xd46300fd"
        },
        "latest"
      ]
    }
    '

The EVM carries out the computation and returns raw bytes as the result:

    {
    "jsonrpc":"2.0",
    "id":1,
            "result":"0x0000000000000000000000000000000000000000000000000000000000000001"
    }

According to the ABI, the bytes should be interpreted as the value 0x1.

## Assembly For External Method Calling

Now let’s see how the compiled contract processes the raw input data to make a method call. Consider a contract that defines setA(uint256):

    pragma solidity ^0.4.11;

    contract C {
      uint256 a;

      // Note: `payable` makes the assembly a bit simpler
      function setA(uint256 _a) payable {
        a = _a;
      }
    }

Compile:

    solc --bin --asm --optimize call.sol

The assembly code for the methods being called is in the body of the contract, organized under sub_0:

    sub_0: assembly {
        mstore(0x40, 0x60)
        and(div(calldataload(0x0), 0x100000000000000000000000000000000000000000000000000000000), 0xffffffff)
        0xee919d50
        dup2
        eq
        tag_2
        jumpi
      tag_1:
        0x0
        dup1
        revert
      tag_2:
        tag_3
        calldataload(0x4)
        jump(tag_4)
      tag_3:
        stop
      tag_4:
          /* "call.sol":95:96  a */
        0x0
          /* "call.sol":95:101  a = _a */
        dup2
        swap1
        sstore
      tag_5:
        pop
        jump // out

    auxdata: 0xa165627a7a7230582016353b5ec133c89560dea787de20e25e96284d67a632e9df74dd981cc4db7a0a0029
    }

There are two pieces of boilerplate code that are irrelevant to this discussion, but FYI:

* mstore(0x40, 0x60) at the very top reserves the first 64 bytes in memory for sha3 hashing. This is always there whether the contract needs it or not.

* auxdata at the very bottom is used to verify that the published source code is the same as the deployed bytecode. This is optional, but baked into the compiler.

Let’s break the remaining assembly code to two parts for easier analysis:

1. Matching the selector and jumping to a method.

1. Loading the arguments, executing method, and returning from method.

First, the annotated assembly for matching the selector:

    // Load the first 4 bytes as method selector
    and(div(calldataload(0x0), 0x100000000000000000000000000000000000000000000000000000000), 0xffffffff)

    // if selector matches `0xee919d50`, goto setA
    0xee919d50
    dup2
    eq
    tag_2
    jumpi

    // No matching method. Fail & revert.
    tag_1:
      0x0
      dup1
      revert

    // Body of setA
    tag_2:
      ...

It’s straightforward except for the bit-shuffling at the beginning to load 4 bytes from call data. For clarity, the assembly logic in low-level pseudocode is like:

    methodSelector = calldata[0:4]

    if methodSelector == "0xee919d50":
      goto tag_2 // goto setA
    else:
      // No matching method. Fail & revert.
      revert

The annotated assembly for the actual method call:

    // setA
    tag_2:
      // Where to goto after method call
      tag_3

      // Load first argument (the value 0x1).
      calldataload(0x4)

      // Execute method.
      jump(tag_4)
    tag_4:
      // sstore(0x0, 0x1)
      0x0
      dup2
      swap1
      sstore
    tag_5:
      pop
      // end of program, will goto tag_3 and stop
      jump
    tag_3:
      // end of program
      stop

Before entering into the method body, the assembly does two things:

1. Saves the position to return to after method call.

1. Loads the arguments from call data onto the stack.

In low-level pseudocode:

    // Saves the position to return to after method call.
    @returnTo = tag_3

    tag_2: // setA
      // Loads the arguments from call data onto the stack.
      @arg1 = calldata[4:4+32]
    tag_4: // a = _a
      sstore(0x0, @arg1)
    tag_5 // return
      jump(@returnTo)
    tag_3:
      stop

Combining the two parts together:

    methodSelector = calldata[0:4]

    if methodSelector == "0xee919d50":
      goto tag_2 // goto setA
    else:
      // No matching method. Fail.
      revert

    @returnTo = tag_3
    tag_2: // setA(uint256 _a)
      @arg1 = calldata[4:36]
    tag_4: // a = _a
      sstore(0x0, @arg1)
    tag_5 // return
      jump(@returnTo)
    tag_3:
      stop
> *Fun trivia: The opcode for revert is fd. But you won't find specification for it in the Yellow Paper, or implementation in code. In fact, fd doesn't actually exist! It's an invalid op. When the EVM encounters an invalid op, it gives up and revert state as a side-effect.*

## Handling Multiple Methods

How does the Solidity compiler generate assembly for a contract that has multiple methods?

    pragma solidity ^0.4.11;

    contract C {
        uint256 a;
        uint256 b;

        function setA(uint256 _a) {
          a = _a;
        }

        function setB(uint256 _b) {
          b = _b;
        }
    }

Simple. Just more if-else branches one after another:

    // methodSelector = calldata[0:4]
    and(div(calldataload(0x0), 0x100000000000000000000000000000000000000000000000000000000), 0xffffffff)

    // if methodSelector == 0x9cdcf9b
    0x9cdcf9b
    dup2
    eq
    tag_2 // SetB
    jumpi

    // elsif methodSelector == 0xee919d50
    dup1
    0xee919d50
    eq
    tag_3 // SetA
    jumpi

In pseudocode:

    methodSelector = calldata[0:4]

    if methodSelector == "0x9cdcf9b":
      goto tag_2
    elsif methodSelector == "0xee919d50":
      goto tag_3
    else:
      // Cannot find a matching method. Fail.
      revert

## ABI Encoding For Complex Method Calls

![Don’t worry about the zeros. It’s FINE.](https://cdn-images-1.medium.com/max/2400/1*EtXrxIQWOtodr4kw3TyJFA.jpeg)*Don’t worry about the zeros. It’s FINE.*

For a method call, the first four bytes of the transaction input data is always the method selector. Then the method arguments follow in chunks of 32 bytes. The [ABI Encoding Specification](https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI) details how the more complex types of arguments are encoded, but it can be extremely painful to read.

Another strategy to learn the ABI encoding is to use the [pyethereum’s ABI encoding function](https://github.com/ethereum/pyethereum/blob/4e945e2a24554ec04eccb160cff689a82eed7e0d/ethereum/abi.py) to investigate how different types of data are encoded. We’ll start from simple cases, and build up to more complex types.

First, import the encode_abi function:

    from ethereum.abi import encode_abi

For a method that has three uint256 arguments (e.g. foo(uint256 a, uint256 b, uint256 c)), the encoded arguments are simply uint256 numbers one after another:

    # The first array lists the types of the arguments.
    # The second array lists the argument values.
    > encode_abi(["uint256", "uint256", "uint256"],[1, 2, 3]).hex()

    0000000000000000000000000000000000000000000000000000000000000001
    0000000000000000000000000000000000000000000000000000000000000002
    0000000000000000000000000000000000000000000000000000000000000003

Types smaller than 32 bytes are padded to 32 bytes:

    > encode_abi(["int8", "uint32", "uint64"],[1, 2, 3]).hex()

    0000000000000000000000000000000000000000000000000000000000000001
    0000000000000000000000000000000000000000000000000000000000000002
    0000000000000000000000000000000000000000000000000000000000000003

For fix-sized arrays, the elements are again 32 bytes chunks (zero padded if necessary), laid out one after another:

    > encode_abi(
       ["int8[3]", "int256[3]"],
       [[1, 2, 3], [4, 5, 6]]
    ).hex()

    // int8[3]. Zero-padded to 32 bytes.
    0000000000000000000000000000000000000000000000000000000000000001
    0000000000000000000000000000000000000000000000000000000000000002
    0000000000000000000000000000000000000000000000000000000000000003

    // int256[3].
    0000000000000000000000000000000000000000000000000000000000000004
    0000000000000000000000000000000000000000000000000000000000000005
    0000000000000000000000000000000000000000000000000000000000000006

## ABI Encoding for Dynamic Arrays

The ABI introduces an layer of indirection to encode dynamic arrays, following a scheme called [head-tail encoding](https://github.com/ethereum/pyethereum/blob/4e945e2a24554ec04eccb160cff689a82eed7e0d/ethereum/abi.py#L735-L741).

The idea is that the elements of the dynamic arrays are packed at the tail-end of the transaction’s calldata. The arguments (the “head” ) are references into the calldata where the array elements are.

If we call a method with 3 dynamic arrays, the arguments are encoded like this (comments and line breaks added for clarity):

    > encode_abi(
      ["uint256[]", "uint256[]", "uint256[]"],
      [[0xa1, 0xa2, 0xa3], [0xb1, 0xb2, 0xb3], [0xc1, 0xc2, 0xc3]]
    ).hex()

    /************* HEAD (32*3 bytes) *************/
    // arg1: look at position 0x60 for array data
    0000000000000000000000000000000000000000000000000000000000000060
    // arg2: look at position 0xe0 for array data
    00000000000000000000000000000000000000000000000000000000000000e0
    // arg3: look at position 0x160 for array data
    0000000000000000000000000000000000000000000000000000000000000160

    /************* TAIL (128**3 bytes) *************/
    // position 0x60. Data for arg1.
    // Length followed by elements.
    0000000000000000000000000000000000000000000000000000000000000003
    00000000000000000000000000000000000000000000000000000000000000a1
    00000000000000000000000000000000000000000000000000000000000000a2
    00000000000000000000000000000000000000000000000000000000000000a3

    // position 0xe0. Data for arg2.
    0000000000000000000000000000000000000000000000000000000000000003
    00000000000000000000000000000000000000000000000000000000000000b1
    00000000000000000000000000000000000000000000000000000000000000b2
    00000000000000000000000000000000000000000000000000000000000000b3

    // position 0x160. Data for arg3.
    0000000000000000000000000000000000000000000000000000000000000003
    00000000000000000000000000000000000000000000000000000000000000c1
    00000000000000000000000000000000000000000000000000000000000000c2
    00000000000000000000000000000000000000000000000000000000000000c3

So the head section has three 32 bytes arguments, pointing to locations in the tail section, which contains the actual data for the three dynamic arrays.

For example, the first argument is 0x60, pointing to the 96th (0x60) byte of the calldata. If you look at the 96th byte, it is the beginning of an array. The first 32 bytes is the length, followed by three elements.

It is possible to mix dynamic and static arguments. Here’s an example with (static, dynamic, static) arguments. The static arguments are encoded as is, whereas the data for the second dynamic array is placed in the tail section:

    > encode_abi(
      ["uint256", "uint256[]", "uint256"],
      [0xaaaa, [0xb1, 0xb2, 0xb3], 0xbbbb]
    ).hex()

    /************* HEAD (32*3 bytes) *************/
    // arg1: 0xaaaa
    000000000000000000000000000000000000000000000000000000000000aaaa
    // arg2: look at position 0x60 for array data
    0000000000000000000000000000000000000000000000000000000000000060
    // arg3: 0xbbbb
    000000000000000000000000000000000000000000000000000000000000bbbb

    /************* TAIL (128 bytes) *************/
    // position 0x60. Data for arg2.
    0000000000000000000000000000000000000000000000000000000000000003
    00000000000000000000000000000000000000000000000000000000000000b1
    00000000000000000000000000000000000000000000000000000000000000b2
    00000000000000000000000000000000000000000000000000000000000000b3

Lots of zeros, but it’s fine.

## Encoding Bytes

Strings and Byte Arrays are also head-tail encoded. The only difference is that the bytes are packed tightly in chunks of 32 bytes, like so:

    > encode_abi(
      ["string", "string", "string"],
      ["aaaa", "bbbb", "cccc"]
    ).hex()

    // arg1: look at position 0x60 for string data
    0000000000000000000000000000000000000000000000000000000000000060
    // arg2: look at position 0xa0 for string data
    00000000000000000000000000000000000000000000000000000000000000a0
    // arg3: look at position 0xe0 for string data
    00000000000000000000000000000000000000000000000000000000000000e0

    // 0x60 (96). Data for arg1
    0000000000000000000000000000000000000000000000000000000000000004
    6161616100000000000000000000000000000000000000000000000000000000

    // 0xa0 (160). Data for arg2
    0000000000000000000000000000000000000000000000000000000000000004
    6262626200000000000000000000000000000000000000000000000000000000

    // 0xe0 (224). Data for arg3
    0000000000000000000000000000000000000000000000000000000000000004
    6363636300000000000000000000000000000000000000000000000000000000

For each string/bytearray, the first 32 bytes encodes the length, followed by the bytes.

If the string is larger than 32 bytes, then multiple 32 bytes chunks are used:

    // encode 48 bytes of string data
    ethereum.abi.encode_abi(
      ["string"],
      ["a" * (32+16)]
    ).hex()

    
    0000000000000000000000000000000000000000000000000000000000000020

    // length of string is 0x30 (48)
    0000000000000000000000000000000000000000000000000000000000000030
    6161616161616161616161616161616161616161616161616161616161616161
    6161616161616161616161616161616100000000000000000000000000000000

## Nested Arrays

Nested arrays have one indirection per nesting.

    > encode_abi(
      ["uint256[][]"],
      [[[0xa1, 0xa2, 0xa3], [0xb1, 0xb2, 0xb3], [0xc1, 0xc2, 0xc3]]]
    ).hex()

    // arg1: The outter array is at position 0x20.
    0000000000000000000000000000000000000000000000000000000000000020

    // 0x20. Each element is the position of an inner array.
    0000000000000000000000000000000000000000000000000000000000000003
    0000000000000000000000000000000000000000000000000000000000000060
    00000000000000000000000000000000000000000000000000000000000000e0
    0000000000000000000000000000000000000000000000000000000000000160

    // array[0] at 0x60
    0000000000000000000000000000000000000000000000000000000000000003
    00000000000000000000000000000000000000000000000000000000000000a1
    00000000000000000000000000000000000000000000000000000000000000a2
    00000000000000000000000000000000000000000000000000000000000000a3

    // array[1] at 0xe0
    0000000000000000000000000000000000000000000000000000000000000003
    00000000000000000000000000000000000000000000000000000000000000b1
    00000000000000000000000000000000000000000000000000000000000000b2
    00000000000000000000000000000000000000000000000000000000000000b3

    // array[2] at 0x160
    0000000000000000000000000000000000000000000000000000000000000003
    00000000000000000000000000000000000000000000000000000000000000c1
    00000000000000000000000000000000000000000000000000000000000000c2
    00000000000000000000000000000000000000000000000000000000000000c3

Ya, lots of zeros.

## Gas Cost & ABI Encoding Design

Why does the ABI truncate the method selector to only 4 bytes? Could there be unlucky collisions for different methods if we don’t use the full 32 bytes of sha256? If the truncation is to save cost, why bother saving a mere 28 bytes in the method selector if it is wasting way more bytes with zero-padding?

These two design choices seem contradictory… until we consider the gas costs for a transaction.

* 21000 paid for every transaction.

* 4 paid for every zero byte of data or code for a transaction.

* 68 paid for every non-zero byte of data or code for a transaction.

Ah ha! Zeros are 17 times cheaper, so zero-padding isn’t as bad as it seems.

The method selector is a cryptographic hash, which is pseudorandom. A random string would tend to have mostly non-zero bytes, since each byte only has 0.3% (1/255) chance of being 0.

* 0x1 padded to 32 bytes costs 192 gas.

4*31 (zeroes bytes) + 68 (1 non-zero byte)

* sha256 is likely to have 32 non-zero bytes, which costs about 2176 gas

32 * 68

* sha256 truncated to 4 bytes would cost about 272 gas

32 * 4

The ABI demonstrates yet another example of quirky low-level design incentivized by the gas cost structure.

### Negative Integers…

Negative integers are usually represented using a scheme called [Two’s Complement](https://en.wikipedia.org/wiki/Two%27s_complement). The value -1 of the type int8 encoded would be all 1s1111 1111.

The ABI pads negative integers with 1s, so -1 would be padded to:

    ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff

Small negative numbers are mostly 1s, costing you quite a lot of gas.

¯\_(ツ)_/¯

## Conclusion

To interact with a Smart Contract, you send it raw bytes. It does some computation, possibly changing its own state, and then sends you raw bytes in return. Method calling does not actually exist. It is a collective illusion created by the ABI.

The ABI is specified like a low-level format, but in function it’s more like a serialization format for a cross-language RPC framework.

We could draw analogies between the architectural tiers of DApp and Web App:

* The blockchain is like the backing database.

* A contract is like a web service.

* A transaction is like a request.

* ABI is the data-interchange format, like [Protocol Buffer](https://en.wikipedia.org/wiki/Protocol_Buffers).

*If you enjoyed this article, you should follow me on Twitter [@hayeah.](https://twitter.com/hayeah)*

In this article series about the EVM I write about:

* [Introduction to the EVM assembly code.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-6e8d5d2f3c30)

* [How fixed-length data types are represented.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-part-2-storage-layout-bc5349cb11b7)

* [How dynamic data types are represented.](https://medium.com/@hayeah/diving-into-the-ethereum-vm-the-hidden-costs-of-arrays-28e119f04a9b)

* How ABI Encodes External Method Calling.

To learn more about the Solidity and EVM, subscribe to my weekly tutorial:

![](https://cdn-images-1.medium.com/max/2316/1*T8-ijp9Q6w_0hc1kMRXq7Q.jpeg)
