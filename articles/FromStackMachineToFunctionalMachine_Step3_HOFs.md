---
tags: taylor
---

# From Stack Machine to Functional Machine: Step 3 (HOFs)

###### tags: `Taylor`, `Ethereum`, `Solidity`, `Yul`, `eWasm`, `WebAssembly`

[ToC]

### Environment

For illustrating our journey, we will use the [Yul language](https://solidity.readthedocs.io/en/v0.6.4/yul.html). Yul compiles to both WebAssembly and EVM bytecode.


If you want to run the examples, it can be done with https://remix.ethereum.org:
- choose Yul as the compiled language, use the raw `calldata` input, check the return value using the debugger.
- use the Yul+ plugin to compile, deploy and interact (you will need to comment out the `mslice` helper function)

The full code source can also be found at https://gist.github.com/loredanacirstea/fc1abd6345a17519455188d2e345f372


### Prerequisites

Read the previous articles:
- [From Stack Machine to Functional Machine: Step 1 - Recursive Apply](https://medium.com/@loredana.cirstea/from-stack-machine-to-functional-machine-step-1-fd2f12a372e2).
- [From Stack Machine to Functional Machine: Step 2 - Currying](https://medium.com/@loredana.cirstea/from-stack-machine-to-functional-machine-step-2-currying-f26c7f8b7220).

## Higher-Order Functions (HOFs)

HOFs are functions that handle other functions as input and/or output arguments. HOFs make functional programming possible.


We are building Taylor: a function/graph-based language, for dType - a decentralized type system. You can find out more about dType and Taylor from our [Solidity Summit presentation](https://youtu.be/XMCgL99noYY). **With Taylor, types are recursively applied functions, based on a suit of "native" functions, implemented in a stack-based language, like Yul or WebAssembly.**

In order to create type definitions and casting functions, especially for various types of arrays, we needed HOFs such as `map`, `reduce`, `curry` (which transforms a function with multiple arguments into a curried function, at runtime).

**In this article, we are exploring how HOFs can be implemented in a stack-based language.** We will use Yul, but a similar approach can also be used for WebAssembly. We will build upon the code that we created in the previous steps, using recursive apply and currying.

### Map & Curry

A good use case example for currying functions is the `map` function, which receives a function and an array as input arguments.

Given an array of integers, if we want to apply `map` over the array and increase each element with `2`, we can use a curried `sum` function, to make our code reusable:

```
const sumCurried = a => b => a + b
const sumPartial = sumCurried(2)

const arr = [4, 7, 8, 2, 10]
const newarr = map(sumPartial, arr)

// newarr: [6, 9, 10, 4, 12]
```

Let's see how our `map` function looks in Yul:

```javascript
// map: function_signature, array
case 0xaaaaaaaa {
  let internal_fsig, fsig_size := getfSig(input_ptr)
  let array_length := mload(add(input_ptr, fsig_size))
  let values_ptr := add(add(input_ptr, fsig_size), 32)

  // build returned array - add length
  mstore(output_ptr, array_length)

  // internal_output_ptr is the memory pointer
  // at which the next array element is stored
  let internal_output_ptr := add(output_ptr, 32)
  result_length := 32

  for { let i:= 0 } lt(i, array_length) { i := add(i, 1) } {
    // Execute function on array element
    let interm_res_length := executeInternal(
      internal_fsig,
      values_ptr,
      32,
      internal_output_ptr,
      virtual_fns
    )

    // Move array values pointer & the resulting array pointer
    // to the next element
    values_ptr := add(values_ptr, 32)
    internal_output_ptr := add(internal_output_ptr, interm_res_length)

    // Update final result length
    result_length := add(result_length, interm_res_length)
  }
}
```

To simplify the pattern, we only expect arrays with `uint256` type elements. You will see how Taylor solves HOFs for any type in the second part of the article, but in the current case, using `uint256` makes the code simpler and easier to understand.

You can look at the full source code at the bottom of the article, to understand how it works.

To test this code, we will use the following `calldata`:
```
0xffffffffcccccccc0000000200000028000000c4bbbbbbbbeeeeeeee0000000000000000000000000000000000000000000000000000000000000002aaaaaaaa00000000000000000000000000000000000000000000000000000000000000050000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000700000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000a
```

Broken down, the `calldata` represents:

```
ffffffff - execute signature (main entry point)
cccccccc - recursive apply signature
00000002 - number of steps for recursive apply
00000028 - length in bytes for the first step
000000c4 - length in bytes for the second step
bbbbbbbb - first step: curry function signature
eeeeeeee - sum function signature
0000000000000000000000000000000000000000000000000000000000000002
    - partially applied argument for sum: 2
aaaaaaaa - second step: map signature
0000000000000000000000000000000000000000000000000000000000000005
    - array length
0000000000000000000000000000000000000000000000000000000000000004
0000000000000000000000000000000000000000000000000000000000000007
0000000000000000000000000000000000000000000000000000000000000008
0000000000000000000000000000000000000000000000000000000000000002
000000000000000000000000000000000000000000000000000000000000000a
    - array values
```

The first step, processed by the `recursiveApply` internal function is currying our `sum` function, transforming it from `const sum = (a, b) => a + b` to `const sumCurried = a => b => a + b`. Internally, we are doing this by storing the `sum` function signature in memory, along with the first `a` argument - read about it in the **Step 2 (currying)** article.

In the second step, `map` receives our `sumCurried` signature (the memory pointer where the partially applied function data exists) and our array and applies the curried function over each array element.

The final value is written at the given memory output pointer `output_ptr`.

If you call the contract with the above `calldata`, the result is:
```
0x000000000000000000000000000000000000000000000000000000000000000500000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000009000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000c
```

Broken down, the result looks like this:
```
0000000000000000000000000000000000000000000000000000000000000005
    - array length
0000000000000000000000000000000000000000000000000000000000000006
0000000000000000000000000000000000000000000000000000000000000009
000000000000000000000000000000000000000000000000000000000000000a
0000000000000000000000000000000000000000000000000000000000000004
000000000000000000000000000000000000000000000000000000000000000c
    - new array values
```

The above is equivalent to our example:
```
const sumCurried = a => b => a + b
const sumPartial = sumCurried(2)

const arr = [4, 7, 8, 2, 10]
const newarr = map(sumPartial, arr)

// newarr: [6, 9, 10, 4, 12]
```


### Reduce

The `reduce` function takes in a function of the form `(accumulator, currentValue) -> accumulator`, an array, and an initial value for the accumulator.

To simplify the pattern, we will only be using arrays of `uint256` type elements and the accumulator will also be of `uint256` type.

Given an array of integers, we want to apply `reduce` over the array and calculate the sum of all elements:

```
const sum = (a, b) => a + b

const arr = [4, 7, 8, 2, 10]
const result = reduce(sum, arr, 0)

// result: 31
```

Let's see how our `reduce` function looks in Yul:

```javascript
// reduce: function_signature, array, accumulator (initial value)
case 0x99999999 {
  let internal_fsig, fsig_size := getfSig(input_ptr)
  let new_ptr := add(input_ptr, fsig_size)

  // The accumulator is treated as a uint256 for simplicity
  let accumulator := mload(new_ptr)
  new_ptr := add(new_ptr, 32)

  let array_length := mload(new_ptr)
  let values_ptr := add(new_ptr, 32)

  for { let i:= 0 } lt(i, array_length) { i := add(i, 1) } {
    // Store arguments in a temporary memory pointer, for simplicity
    let temporary_input_ptr := 6000
    mstore(temporary_input_ptr, accumulator)
    mstore(add(temporary_input_ptr, 32), mload(values_ptr))

    // Output memory pointer is 0x00
    let interm_res_length := executeInternal(
      internal_fsig,
      temporary_input_ptr,
      64,
      0,
      virtual_fns
    )

    // Read result from the output pointer
    accumulator := mload(0)
    // Move array values pointer to the next element
    values_ptr := add(values_ptr, 32)
  }
  mstore(output_ptr, accumulator)
  result_length := 32
}
```

You can look at the full source code at the bottom of the article, to understand how it works.

To test this code, we will use the following `calldata`:
```
0xffffffffcccccccc00000001000000e899999999eeeeeeee000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000050000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000700000000000000000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000a
```

Broken down, the `calldata` represents:

```
ffffffff - execute signature (main entry point)
cccccccc - recursive apply signature
00000001 - number of steps for recursive apply
000000e8 - length in bytes for the first step
99999999 - first step: reduce function signature
eeeeeeee - sum signature (reduce function argument)
0000000000000000000000000000000000000000000000000000000000000000
    - reduce initial accumulator value
0000000000000000000000000000000000000000000000000000000000000005
    - array length
0000000000000000000000000000000000000000000000000000000000000004
0000000000000000000000000000000000000000000000000000000000000007
0000000000000000000000000000000000000000000000000000000000000008
0000000000000000000000000000000000000000000000000000000000000002
000000000000000000000000000000000000000000000000000000000000000a
    - array values
```

There is only one step that the `recursiveApply` internal function handles: the `reduce` function is called with the signature for the internal `sum` function, the initial accumulator value `0` and the array.

`reduce` then passes the `accumulator` value and an array element to `sum` and the result becomes the new `accumulator`.

The final value is written at the given memory output pointer `output_ptr`.


If you call the contract with the above `calldata`, the result is `31`:
```
0x000000000000000000000000000000000000000000000000000000000000001f
```


## Applications in Taylor

We use the `map`, `curry` and `reduce` functions in our Taylor on-chain graph interpreter smart contract, used for defining types. These functions are especially useful for defining array types and building casting functions, which iterate over array elements and cast them to a new type.


**Taylor uses a special, typed encoding and decoding format, where values are always preceded by their type. This allows Taylor to do some interesting things:**
- **have runtime type checking**
- **build generic functions, that can handle multiple types at runtime (Taylor only works with memory pointers)**

Our example from the first part of the article had some obvious limitations: we were using only `uint256` type values in the arrays and accumulator. Even if we use a general ABI encoding (e.g. Ethereum ABI encoding), it would be hard to impossible to create a generic `map` or `reduce` function that would work on any type (especially with varying type sizes - e.g. `uint8`).

But with Taylor, generic functions are possible and this opens the door towards **generic HOFs**, that do not need to be defined (and stored on-chain), separately, for each type.

Generic HOFs are the distinct feature of functional systems, that give them their intrinsic power.


### Taylor Array Casting Example

To try out the following example, you can interact with the Taylor smart contract deployed on Ropsten, at `0x7D4150f492f93e2eDD7FC0Fc62c9193b322f75e5`:
- create a new `.js` file in https://remix-alpha.ethereum.org
- copy the following code:

```
const address = '0x7D4150f492f93e2eDD7FC0Fc62c9193b322f75e5'

let provider = ethers.getDefaultProvider('ropsten');

const newUint8 = '0xffffffff11000001';
const newUint32Array3 = '0xffffffff44000000ee000002000000070000000f110000030000042200000411000004';
const castArrayInt32ToUint256 = '0xffffffff77777788ee0000020000000c000000202200000844000003110000204400000312000004000000020000000500000004';


const data = castArrayInt32ToUint256;

const transaction = {
    to: address,
    data,
}

provider.call(transaction).then(console.log);

// remix.exeCurrent()
```
- in the Remix console, execute `remix.exeCurrent()` or `remix.execute(filepath_to_your_js_file)`
- check out a more detailed explanation of the typed encoding format here: https://github.com/loredanacirstea/taylor/blob/master/SupportedTypes.md


The following is an example of a graph, that can be interpreted by Taylor, which casts an array to another array of the same length, but different array element types. Graphs in Taylor are more complex than in our example from the first part of the article - each graph step (function) can receive as input any of the initial inputs or variables produced by previous steps.

To be used in Taylor, we first store the graph, by calling the Taylor contract with this `calldata`:

```
0xfffffffe00000005777777880300000026ee000003000000070000000e000000161100000300000411000003000002220000043333331c0000001f3333331a000000030203003333332800000002040533333332000000020601
```

Broken down, the `calldata` represents:

```
fffffffe - store graph signature
00000005 - length in bytes for the type definition head
77777788 - this graph's signature
03       - steps count

00000026 - length in bytes for hardcoded graph inputs
ee000003 - tuple of 3 elements follows
00000007 - additive sums of lengths for each tuple element
0000000e
00000016
11000003 - type uint24 for slice_size - 02
000004   - slice_size value
11000003 - type uint24 for index - 03
000002   - index value
22000004 - type bytes4 for cast signature
3333331c - cast function signature

0000001f - length in bytes for graph steps
3333331a - selectraw signature
00000003 - how many inputs selectraw will receive
02       - index for finding slice_size in all graph-local variables
03       - index for finding index
00       - index for to_type
33333328 - curry signature
00000002
04       - index for cast signature
05       - index for selectraw signature
33333332 - map signature
00000002
06       - index for curry signature
01       - index for the array to cast from
```

To summarize: we have a graph with three steps (functions):
- `selectraw` - we need this to select the `cast_to` type for an array element from the new array type. E.g. selecting `11000020` (`uint256`) from `22000008 44000003 11000020`, where `4400000311000020` means `uint256[3]`.
- `curry` - curries the `cast` function & partially applies it to the `selectraw` result: `11000020` (`cast_to` type).
- `map` - maps over the `cast_from` array elements and applies the partially applied `cast` function on each of them. Aggregates the results in an array.

We have stored our graph. Now, we can execute it.

Given the array `[2, 5, 4]` or type `int32[3]`, we want to cast it to `uint256[3]`. Given that all the array values are valid unsigned integers that fit into a `uint256`, it should be possible to achieve this.

The `calldata` for doing this using our array casting graph is:

```
0xffffffff77777788ee0000020000000c000000202200000844000003110000204400000312000004000000020000000500000004
```

Broken down, the `calldata` represents:

```
ffffffff - execute function signature, the main entry point
77777788 - our array casting graph signature
ee000002 - tuple of 2 elements following
0000000c - additive sum of lengths in bytes for each tuple element
00000020
22000008 - type bytes8 for to_array
44000003 - to_array signature: uint256[3]
11000020
44000003 - actual from_array starts here, type int32[3]
12000004
00000002
00000005
00000004
```

If you call the Taylor contract with the above `calldata`, the result is:
```
0xee000001000000684400000311000020000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000050000000000000000000000000000000000000000000000000000000000000004
```
Broken down, the result represents:
```
ee000001 - all inputs & outputs are wrapped in a tupple
00000068
44000003 - final array uint256[3]
11000020
0000000000000000000000000000000000000000000000000000000000000002
0000000000000000000000000000000000000000000000000000000000000005
0000000000000000000000000000000000000000000000000000000000000004
   - array values: [2, 5, 7], left-padded to fit uint256
```


## Full Code

```javascript
object "ContractB" {
  code {
    datacopy(0, dataoffset("Runtime"), datasize("Runtime"))
    return(0, datasize("Runtime"))
  }
  object "Runtime" {
    code {
      let _calldata := 2048
      let _output_pointer := 0

      // This is where we keep our virtual functions
      // generated at runtime as partial function applications
      let _virtual_fns := 1024

      calldatacopy(_calldata, 0, calldatasize())

      let fn_sig := mslice(_calldata, 4)

      switch fn_sig

      // execute function
      case 0xffffffff {
        let internal_fn_sig := mslice(add(_calldata, 4), 4)
        let input_pointer := add(_calldata, 8)
        let input_size := sub(calldatasize(), 4)

        let result_length := executeNative(
          internal_fn_sig,
          input_pointer,
          input_size,
          _output_pointer,
          _virtual_fns
        )
        return (_output_pointer, result_length)
      }
      // other cases/function signatures
      default {
        mslicestore(_output_pointer, 0xeee1, 2)
        revert(_output_pointer, 2)
      }

      function executeNative(
        fsig,
        input_ptr,
        input_size,
        output_ptr,
        virtual_fns
      ) -> result_length {
        switch fsig

        // sum: a + b
        case 0xeeeeeeee {
          let a := mload(input_ptr)
          let b := mload(add(input_ptr, 32))
          mstore(output_ptr, add(a, b))
          result_length := 32
        }

        // recursiveApply, with piping outputs as input to the next function
        case 0xcccccccc {
          // number of execution steps
          let count := mslice(input_ptr, 4)

          // offsets/size in bytes for each step
          let offsets_start := add(input_ptr, 4)
          let input_inner := add(offsets_start, mul(count, 4))

          let temporary_ptr := 0x80
          let last_output_size := 0

          for { let i := 0 } lt(i, count) { i := add(i, 1) } {
            let step_length := mslice(add(offsets_start, mul(i, 4)), 4)

            // current step function signature
            let func_sig := mslice(input_inner, 4)

            // add current input after previous return value
            mmultistore(
              add(temporary_ptr, last_output_size),
              add(input_inner, 4),  // without the function signature
              sub(step_length, 4)
            )

            result_length := executeInternal(
              func_sig,
              temporary_ptr,
              add(last_output_size, sub(step_length, 4)),
              output_ptr,
              virtual_fns
            )

            // move termporary input after previous data
            temporary_ptr := add(temporary_ptr, step_length)

            // store output as new input for the next step
            mmultistore(temporary_ptr, output_ptr, result_length)
            last_output_size := result_length

            // move input pointer to the next step
            input_inner := add(input_inner, step_length)
          }
        }

        // curry: fsig, partial application argument
        case 0xbbbbbbbb {
          // first 32 bytes is the next free memory pointer
          let fpointer := mload(virtual_fns)
          if eq(fpointer, 0) {
            fpointer := add(virtual_fns, 32)
          }

          let internal_fsig := mslice(input_ptr, 4)
          let arg := mload(add(input_ptr, 4))

          // virtual function marker
          mslicestore(fpointer, 0xfefe, 2)

          // add input size (so we know how much to read)
          mstore(add(fpointer, 2), input_size)

          // store the actual data - partial application argument
          mmultistore(add(fpointer, 34), input_ptr, input_size)

          // update the free memory pointer for our curried functions references
          mstore(virtual_fns, add(fpointer, 38))

          // return the virtual function pointer
          mstore(output_ptr, fpointer)
          result_length := 32
        }

        // map: function_signature, array
        case 0xaaaaaaaa {
          let internal_fsig, fsig_size := getfSig(input_ptr)
          let array_length := mload(add(input_ptr, fsig_size))
          let values_ptr := add(add(input_ptr, fsig_size), 32)

          // build returned array - add length
          mstore(output_ptr, array_length)

          // internal_output_ptr is the memory pointer
          // at which the next array element is stored
          let internal_output_ptr := add(output_ptr, 32)
          result_length := 32

          for { let i:= 0 } lt(i, array_length) { i := add(i, 1) } {
            // Execute function on array element
            let interm_res_length := executeInternal(
              internal_fsig,
              values_ptr,
              32,
              internal_output_ptr,
              virtual_fns
            )

            // Move array values pointer & the resulting array pointer
            // to the next element
            values_ptr := add(values_ptr, 32)
            internal_output_ptr := add(internal_output_ptr, interm_res_length)

            // Update final result length
            result_length := add(result_length, interm_res_length)
          }
        }

        // reduce: function_signature, array, accumulator (initial value)
        case 0x99999999 {
          let internal_fsig, fsig_size := getfSig(input_ptr)
          let new_ptr := add(input_ptr, fsig_size)

          // The accumulator is treated as a uint256 for simplicity
          let accumulator := mload(new_ptr)
          new_ptr := add(new_ptr, 32)

          let array_length := mload(new_ptr)
          let values_ptr := add(new_ptr, 32)

          for { let i:= 0 } lt(i, array_length) { i := add(i, 1) } {
            // Store arguments in a temporary memory pointer, for simplicity
            let temporary_input_ptr := 6000
            mstore(temporary_input_ptr, accumulator)
            mstore(add(temporary_input_ptr, 32), mload(values_ptr))

            // Output memory pointer is 0x00
            let interm_res_length := executeInternal(
              internal_fsig,
              temporary_input_ptr,
              64,
              0,
              virtual_fns
            )

            // Read result from the output pointer
            accumulator := mload(0)
            // Move array values pointer to the next element
            values_ptr := add(values_ptr, 32)
          }
          mstore(output_ptr, accumulator)
          result_length := 32
        }

        // other cases/function signatures
        default {
          // revert with error code
          mslicestore(output_ptr, 0xeee2, 2)
          revert(output_ptr, 2)
        }
      }

      function executeInternal(
        fsig,
        input_ptr,
        input_size,
        output_ptr,
          virtual_fns
      ) -> result_length {
        let fsig_size := getfSigSize(fsig)

        switch fsig_size
        case 4 {
          result_length := executeNative(
            fsig,
            input_ptr,
            input_size,
            output_ptr,
            virtual_fns
          )
        }
        case 32 {
          result_length := executeCurriedFunction(
            fsig,
            input_ptr,
            input_size,
            output_ptr,
            virtual_fns
          )
        }
        default {
          // revert with error code
          mslicestore(output_ptr, 0xeee3, 2)
          revert(output_ptr, 2)
        }
      }

      function executeCurriedFunction(
        fpointer,
        input_ptr,
        input_size,
        output_ptr,
        virtual_fns
      ) -> result_length {
        // first 32 bytes are the input size
        let new_input_size := mload(add(fpointer, 2))

        // exclude input size from input ptr
        let new_input_ptr := add(fpointer, 34)

        // get curried function signature
        let fsig, fsig_size := getfSig(new_input_ptr)

        // exclude signature
        new_input_ptr := add(new_input_ptr, fsig_size)
        new_input_size := sub(new_input_size, fsig_size)

        // store the inputs for the curried function after the curried function arguments
        // effectively composing the input for the actual function that we need to run
        mmultistore(add(new_input_ptr, new_input_size), input_ptr, input_size)
        new_input_size := add(new_input_size, input_size)

        result_length := executeInternal(
          fsig,
          new_input_ptr,
          new_input_size,
          output_ptr,
          virtual_fns
        )
      }

      function getfSigSize(fsig) -> fsig_size {
        fsig_size := 4

        if lt(fsig, 10000000) {
          // check if the curried function marker exists
          // fsig is a memory pointer
          if eq(mslice(fsig, 2), 0xfefe) {
            fsig_size := 32
          }
        }
      }

      function getfSig(input_ptr) -> fsig, fsig_size {
        let fpointer := mload(input_ptr)
        fsig_size := getfSigSize(fpointer)

        switch fsig_size
        case 4 {
          fsig := mslice(input_ptr, 4)
        }
        case 32 {
          fsig := fpointer
        }
        default {
          mslicestore(0, 0xeee0, 2)
          revert(0, 2)
        }
      }

      // loads a slice of bytes in memory, where the slice <= 32 butes
      function mslice(position, length) -> result {
        result := div(
          mload(position),
          exp(2, sub(256, mul(length, 8)))
        )
      }

      // stores a slice of bytes in memory, for tight packing
      // (always shifts to left)
      function mslicestore(_ptr, val, length) {
        let slot := 32
        mstore(_ptr, shl(mul(sub(slot, length), 8), val))
      }

      // stores any number of bytes in memory
      function mmultistore(_ptr_target, _ptr_source, sizeBytes) {
        let slot := 32
        let size := div(sizeBytes, slot)

        for { let i := 0 } lt(i, size)  { i := add(i, 1) } {
          mstore(
            add(_ptr_target, mul(i, slot)),
            mload(add(_ptr_source, mul(i, slot)))
          )
        }

        let current_length :=  mul(size, slot)
        let remaining := sub(sizeBytes, current_length)
        if gt(remaining, 0) {
          mslicestore(
            add(_ptr_target, current_length),
            mslice(add(_ptr_source, current_length), remaining),
            remaining
          )
        }
      }
    }
  }
}
```
