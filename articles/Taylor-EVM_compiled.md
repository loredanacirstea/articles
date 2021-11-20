---
tags: taylor
---

# Taylor - A Performant Language for the Ethereum Virtual Machine (EVM)

![Taylor-EVM](https://raw.githubusercontent.com/loredanacirstea/articles/master/articles/assets/taylor-eth-medium.png)


Taylor has started as an interpreted functional language for the EVM, being born out of our [Pipeline](https://github.com/pipeos-one/pipeline) project. Pipeline is a textual and visual language for flow-based programming, where function inputs and outputs can be connected to create complex atomic operations. These atomic operations are interpreted by our Pipeline graph interpreter smart contract or the graphs can be deployed as standalone, smart contracts, optimized for smaller transaction gas costs.

With Taylor, we created a Turing complete language, that can be interpreted by the EVM, with operations that can run both on the stack or on memory frames. This allows Taylor to do recursive operations that are not possible now on Solidity or [Yul](https://docs.soliditylang.org/en/latest/yul.html) (the language that Solidity is based on)
(https://github.com/ethereum/solidity/issues/9622).

For example, Taylor can interpret this recursive implementation of the Fibonacci series with a higher number of iterations (bounded by gas limit instead of stack limit), because, unlike Solidity or Yul, it can leave the stack empty between function calls:

```lisp
(def! fibonacci (fn* (n) (if (or (eq n 1) (eq n 2)) 1 (add(fibonacci (sub n 1)) (fibonacci (sub n 2)) ) )))
```

While building the Taylor interpreter and other EVM interpreters, neither Solidity, nor Yul was flexible enough to build them efficiently. We created a macro language exactly for this purpose - [mASM](https://github.com/loredanacirstea/masm). mASM produces EVM assembly. 

Today we are announcing the next step for Taylor: as a compiled language.

`taylor-evm` is an intermediate language that gives developers access to all EVM instructions and to several out-of-the-box compositions. It compiles to EVM bytecode, which can be deployed as a smart contract. It supersedes mASM.

Its canonical form looks like `(add 34 (add 3 4))`. But, it also has an indentation-based form with optional parenthesis:

```python
add
  34
  add
    3
    4
```

or

```lisp
add
  34
  (add 3 4)
```

You can directly manipulate the stack:
```lisp
(list 
  0x02
  0x04
  (dup 2)
  (swap 1)
  (add 5))
```

You can have access to change control flow:

```lisp
(if (eq 3 4) (add 0x03 0x04) (add 0x05 0x06))

switch 0x4444
    (case 0x1000 (add 0x1 0x1))
    (case 0x4444 (add 0x2 0x2))
    (case 0x2000 (add 0x3 0x3))
    (default 0x55)
```

You can use loops. The following is the efficient, non-recursive way to implement the Fibonacci function (fib):

```lisp
(list
  0x00
  0x01
  (loop 1 8
    (list
      (add (dup 4) (dup 4))
      (swap 3)
      (swap 4)
      (pop)
    )
  )
  (mem-store 0x00)
  (return 0x00 0x20)
)
```


Hence, the Fibonacci function can be written as a smart contract like this:

```lisp
define fib ["uint256"] ["uint256"] 
  if (eq 0x0 (dup 1))
    (pass 0x0)
    if (eq 0x1 (dup 1))
      (pass 0x01)
      list
        add
          (fib (sub (dup 4) 1 ))
          (fib (sub (dup 3) 2 ))
        (swap 1)
        (pop)


to-deploy 
  list
    (include "fib")
    (fib (calldata-load 0x0))
    (mem-store 0x00)
    (return 0x00 0x20)
```

Or, with indentation-only, like this:

```lisp
if 
  eq
    0x1
    dup
      1
  pass
    0x01
  list
    add
      fib
        sub
          dup
            4
          1
      fib
        sub
          dup
            3
          2
    swap
      1
    pop
```

The generated assembly code (asm) is:


```assembly
0x4b
0x0d
0x00
codecopy
0x4b
0x00
return
stop
after_do_fib_14
0x00
calldataload
fib
jump

after_do_fib_14:
0x00
mstore
0x20
0x00
return

fib:
dup1
0x00
eq
ifsource_9
jumpi

elsesource_9:
dup1
0x01
eq
ifsource_8
jumpi

elsesource_8:
after_do_fib_13
0x02
dup3
sub
fib
jump

after_do_fib_13:
after_do_fib_12
0x01
dup4
sub
fib
jump

after_do_fib_12:
add
swap1
pop
endif_8
jump

ifsource_8:
fib_end
jump

endif_8:
endif_9
jump

ifsource_9:
fib_end
jump

endif_9:

fib_end:
swap1 
jump


```

## Comparison

The above Fibonacci function contract is equivalent to the following contract in Yul:

```solidity
object "TestFib" {
  code {
    datacopy(0, dataoffset("Runtime"), datasize("Runtime"))
    return(0, datasize("Runtime"))
  }
  object "Runtime" {
    code {
      let input := calldataload(0x0)
      let res := fib(input)
      mstore(0x0, res)
      return(0x0, 0x20)
      function fib(nth) -> result {
        switch nth
        case 0 { result := 0 }
        case 1 { result := 1 }
        default {
          result := add(fib(sub(nth, 2)), fib(sub(nth, 1)))
}}}}}
```

At this stage, these are the stats from our own Cometh EVM debugger when calling `fib(16)`, `fib(18)`, `fib(19)`, `fib(30)`:

|fib(x)|taylor x=16 | yul 16| taylor 18 | yul 18 | taylor 19 | yul 19 | taylor 30 |
|---|-----------------|---------|--------------|---------|---|---|---|
|gas cost| 318,044 | 338,138 | 832,815 | 885,425 | 1,347,586 | 1,432,852 | >30,000,000 |
|execution steps | 85,745 | 95,559 | 224,527 | 250,223 | 363,309 | 404,927 |  |
|function calls | 65,536 | 65,536 | 262,144 |262,144 | 524,288 | 524,288 | 1,073,741,824 |
|max recursions| 16 | 16 | 18 | 18 | 19 | 19 | 30 |
|max stack occupied| 49 | 67 | 55 | 75 | 58 | 84? | 91 |
|result | 987 | 987 | 2,584 | 2,584 | 4,181 | 4,181 | 832,040 |


The performance of the compiled Taylor relative to Yul is a consistent 94% on gas and 89.7% on execution steps with a more important margin on stack usage.


## Compiled vs. Interpreted

The compiled version of Taylor fulfills the usecases that Solidity and Yul fulfill - optimizing for transaction execution in the EVM.

The interpreted version of Taylor optimizes for flexibility, upgradability, and off-chain processing, verified by the EVM.


| dimension | interpreted on EVM | compiled to EVM | advantage |
|-----------|--------------------|-----------------|-----------|
| gas used for running  | more | less |  compiled (better than yul, Solidity) |
| performance | less | more | compiled (better than yul) |
| gas used for deployment  | less (none) | more |  interpreted |
| upgradeability | more | less | interpreted |
| debugging | more | less | interpreted |
| max recursion | more than 2000 | 18 | interpreted (better than yul, Solidity) |
| stack hygiene | robust | in development | interpreted |
| memory management | robust | in development | interpreted |
| simulators | JS, WASM, Python | JS | interpreted |
| ease of learning | easier | harder | interpreted |
| replayability | no difference | no difference | none |
| compatibility with Solidity | support exists | support exists | none |
| use with dApps development | no difference | no difference | none |
| type checking | available | available | none |
| DB support | dType | dType | none |
| underlying tech | masm (asm macros) | asm | none |




As far as we know Solidity's internals, the compiled code is generated by Yul: it is based on Yul. If we managed to outperform Yul, it is the only comparison we need: Solidity will be performing equal to or less than Yul.
In order to achieve maximum performance, we had to use EVM assembly language (ASM) directly and for this purpose, we had to develop our own macro language: mASM. And the interpreted Taylor is based on masm while the compiled Taylor is built (using itself) straight from ASM.
Taylor is presently used as a language interpreted by the EVM, compiled for the EVM, and interpreted by JavaScript. Therefore a developer could use the same language to develop web3 dapps.
However, amongst the 3 flavors of Taylor, the compiled one (that we demoed today) is the most under-developed: work on stack and memory management is underway with the intent to outperform our present implementation.

## Video Demo

https://youtu.be/vpLpFxRJKDE

This technology was created by and for volunteers at The Laurel Project. You may join this tech effort.






