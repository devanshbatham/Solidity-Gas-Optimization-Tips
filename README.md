# Solidity Gas Optimization Tips
Solidity Gas Optimization Tips taken from various sources (sources listed in the #credits section below). Make pull requests to contribute gas alpha. 


# Gas Optimizations in Solidity

## 1- Use of Bit shift operators

Check if arithmethic operations can be achieved using bitwise operators, if yes, implement same operations using bitwise operators and compare the gas consumed. Usually bitwise logic will be cheaper (ofc we cannot use bitwise everywhere, there some limitations)

**Note:** Bit shift `<<` and `>>` operators are not among the arithmetic ones, and thus don’t revert on overflow.

**Code example:** 


```diff
-    uint256 alpha = someVar / 256;
-    uint256 beta = someVar % 256;
+    uint256 alpha = someVar >> 8;
+    uint256 beta = someVar & 0xff;


-    uint 256 alpha = delta / 2;
-    uint 256 beta = delta / 4; 
-    uint 256 gamma = delta * 8;
+    uint 256 alpha = delta >> 1;
+    uint 256 beta = delta >> 2; 
+    uint 256 gamma = delta << 3;

```

## 2- Public vs External (External is cheaper)

The `public` visiblilty modifier is equivalent to `external` plus `internal`. In other words, both `public` and `external` can be called from outside your contract (like MetaMask), but of these two, only `public` can be called from other functions inside your contract.

Because `public` grants more access than `external` (and is costlier than the latter), the general best practice is to prefer `external`. Then you can consider switching to `public` if you fully understand the security and design implications.

**Code Example:**

```solidity
pragma solidity 0.8.10;

contract Test {
 
    string message = "Hello World";
    
    // Execution cost: 24527
    function test() public view returns (string memory){
         return message;
    }

    //Execution cost: 24505
    function test2() external view returns  (string memory){
         return message;
    }
}
```

## 3- There is no need to initialize variables with default values

If the variable is not set/initialized, it is assumed to have a default value (0, false, 0x0, etc., depending on the data type). If you explicitly initialize it with its default value, you are just wasting gas.


```solidity
uint256 foo = 0;//bad, expensive
uint256 bar;//good, cheap
```


## 4- Use short strings in revert, require checks

It is considered a best practice to append the error reason string with `require` statement. But these strings take up space in the deployed bytecode. Each reason string needs at least 32 bytes, so make sure your string complies with 32 bytes, otherwise it will become more expensive.


**Code Example:**


```solidity
require(balance >= amount, "Insufficient balance");//good
require(balance >= amount, "Too bad, it appears, ser you are broke, bye bye"; //bad
```

## 5- Avoid redundant checks
We should avoid unnecessary/redundant checks to save some extra gas. 

**Code Example:**

```solidity
Bad Code:
require(balance>0, "Insufficient balance");
if (balance>0){ // this check is redundant
    . . . //some action
}

Good Code: 
require(balance>0, "Insufficient balance");
. . . //some action
```



## 6- Use nested if and, avoid multiple check combinations

Using nested is cheaper than using && multiple check combinations. There are more advantages, such as easier to read code and better coverage reports. 



**Code Example:**




```solidity
pragma solidity 0.8.10;

contract NestedIfTest {

    //Execution cost: 22334 gas
  function funcBad(uint256 input) public pure returns (string memory) { 
       if (input<10 && input>0 && input!=6){ 
           return "If condition passed";
       } 

   }

    //Execution cost: 22294 gas
    function funcGood(uint256 input) public pure returns (string memory) { 
    if (input<10) { 
        if (input>0){
            if (input!=6){
                return "If condition passed";
            }
        }
    }
}
}
```

## 7- Use of Multiple require(s)
Using mutiple require statements is cheaper than using `&&` multiple check combinations. There are more advantages, such as easier to read code and better coverage reports. 



```solidity
pragma solidity 0.8.10;

contract MultipleRequire {

    // Execution cost: 21723 gas
  function bad(uint a) public pure returns(uint256) {
        require(a>5 && a>10 && a>15 && a>20);
        return a;
    }

    // Execution cost: 21677 gas
    function good(uint a) public pure returns(uint256) {
        require(a>5);
        require(a>10);
        require(a>15);
        require(a>20);
        return a;
    }
}
```

## 8- Internal functions are cheaper to call
Set approriate function visibillities, as `internal` functions are cheaper to call than `public` functions.There is no need to mark a function as `public` if it is only meant to be called internally.

```diff
//Following function will only be called interally
- function _func() public {...}
+ function _func() internal {...}
```

## 9- Use libraries to save some bytecode

When you call a `public` function of the library, the bytecode of the function will not become part of your contract, so you can put complex logic in the library while keeping the contract scale small. 


The call to the library is made through a delegate call, which means that the library can access the same data and the same permissions as the contract. This means that it is not worth doing for simple tasks.

## 10- Replace state variable reads and writes within loops with local variable reads and writes.


Reading and writing local variables is cheap, whereas reading and writing state variables that are stored in contract storage is expensive.

```solidity
function badCode() external {                
  for(uint256 i; i < myArray.length; i++) { // state reads
    myCounter++; // state reads and writes
  }        
}


function goodCode() external {
    uint256 length = myArray.length; // one state read
    uint256 local_mycounter = myCounter; // one state read
    for(uint256 i; i < length; i++) { // local reads
        local_mycounter++; // local reads and writes  
    }
    myCounter = local_mycounter; // one state write
}
```


## 11- Packing Structs

A common gas optimization is “packing structs” or “packing storage slots”. This is the action of using smaller types like uint128 and uint96 next to each other in contract storage. When values are read or written in contract storage a full 256 bits are read or written. So if you can pack multiple variables within one 256 bit storage slot then you are cutting the cost to read or write those storage variables in half or more.


```solidity
// Unoptimized
struct MyStruct {
  uint256 myTime;
  address myAddress;
}

//Optimized
struct MyStruct {
  uint96 myTime;
  address myAddress;
}
```

In the above a `myTime` and `myAddress` state variables take up `256` bits so both values can be read or written in a single state read or write.

## 12- Solidity Gas Optimizer
Make sure Solidity’s optimizer is enabled. It reduces gas costs. If you want to gas optimize for contract deployment (costs less to deploy a contract) then set the Solidity optimizer at a low number. If you want to optimize for run-time gas costs (when functions are called on a contract) then set the optimizer to a high number.

## 13- Save on data types
It is better to use `uint256` and `bytes32` than using uint8 for example. While it seems like `uint8` will consume less gas than uint256 it is not true, since the Ethereum virtual Machine(EVM) will still occupy `256` bits, fill 8 bits with the uint variable and fill the extra bites with zeros. 


**Code Example:**
```solidity
pragma solidity ^0.8.1;

contract SaveGas {
    
    uint8 resulta = 0;
    uint resultb = 0;
    
    // Execution cost: 73446 gas
    function UseUint() external returns (uint) {
        uint selectedRange = 50;
        for (uint i=0; i < selectedRange; i++) {
            resultb += 1;
        }
        return resultb;
    }
    
    // Execution cost: 84175 gas 
    function UseUInt8() external returns (uint8){
        uint8 selectedRange = 50;
        for (uint8 i=0; i < selectedRange; i++) {
            resulta += 1;
        }
        return resulta;
    }
}
```

## 14- Short circuiting
Short-circuiting is a strategy we can make use of when an operation makes use of either ``||`` or ``&&``. This pattern works by ordering the lower-cost operation first so that the higher-cost operation may be skipped (short-circuited) if the first operation evaluates to true.


```solidity
// f(x) is low cost
// g(y) is expensive

// Ordering should go as follows
f(x) || g(y)
f(x) && g(y)
```

## 15- Avoiding/Removing unnecessary libraries
Libraries are often only imported for a small number of uses, meaning that they can contain a significant amount of code that is redundant to your contract. If you can safely and effectively implement the functionality imported from a library within your contract, it is optimal to do so.


## 16- Make fewer external calls
External calls are expensive, therefore, fewer external gas == fewer gas cost.


## 17- `unchecked { ++i;}` is cheaper than `i++;` & `i=i+1;`


**Code Example:**

```solidity

// Unoptimized Code

for(uint i=0; i<n; i++){
    . . . //some stuff here
}

//Optimized Code

for(uint i; i<n;){
    . . . //some stuff here
    unchecked { ++i; }
}
```

## 18- Avoid repeated computations
Arithmetic computations cost gas, it is recommended to avoid repeated computations. 

**Code Example:**

```solidity

// Unoptimized code: 
for (uint i=0;i<length;i++) {
tokens[i] += limit * price;
}


// Optimized code:
uint local = limit * price;
for (uint i=0;i<length;i++) {
tokens[i] += local;
}
```

## 19- Avoid dead code
There is no point of leaving dead lines in code. Those lines are never going to execute, and but they will take place in bytecode, it is better to get rid of em.


**Code Example:**

```solidity
function deadCode(uint x) public pure {
  if(x <1) {
    if(x> 2) {
      return x;
    }
  }
}
```

## 20- Delete your variables to get a gas refund.

Since there's no garbage collection, you have to throw away unused data yourself. 


**Code Example:**

```solidity

//Using delete keyword
delete myVariable;

//Or assigning the value 0 if integer
myInt = 0;
```

## 21- Single line swaps
One line to swap two variables without writing a function or temporary variable that needs more gas.


**Code Example:**

```solidity
(hello, world) = (world, hello);
```

## 22- Store data on memory not storage.

Choosing the perfect Data location is essential. You must know these:

- **storage** - variable is stored on the blockchain. It's a persistent state variable. Costs Gas to define it and change it.

- **memory** - temporary variable declared inside a function. No gas for declaring. But costs gas for changing memory variable (less than storage)

- **calldata** - like memory but non-modifiable and only available as an argument of external functions

Also, it is important to note that, If not specified data location, then it storage by default.

## 23- Use `calldata` instead of `memory` for function parameters


```solidity
contract C {
    function add(uint[] memory arr) external returns (uint sum) {
        uint length = arr.length;
        for (uint i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
    }
}
```

In the above example, the dynamic array `arr` has the storage location `memory`. When the function gets called externally, the array values are kept in `calldata` and copied to memory during ABI decoding (using the opcode `calldataload` and `mstore`). And during the for loop, `arr[i]` accesses the value in memory using a `mload`. However, for the above example this is inefficient. Consider the following snippet instead:


```solidity
contract C {
    function add(uint[] calldata arr) external returns (uint sum) {
        uint length = arr.length;
        for (uint i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
    }
}
```

## 24- Some state variables can be set to immutable

```solidity

// Unoptimized code: 
contract C {
    /// The owner is set during contruction time, and never changed afterwards.
    address public owner = msg.sender;
}

// Optimized code: 


contract C {
    /// The owner is set during contruction time, and never changed afterwards.
    address public immutable owner = msg.sender;
}
```

## 25- Declaring constructor as payable

You can cut out 10 opcodes in the creation-time EVM bytecode if you declare a constructor payable. The following opcodes are cut out:

- `CALLVALUE`
- `DUP1`
- `ISZERO`
- `PUSH2`
- `JUMPI`
- `PUSH1`
- `DUP1`
- `REVERT`
- `JUMPDEST`
- `POP`

In Solidity, this chunk of assembly would mean the following:


```solidity
if(msg.value != 0) revert();
```

## 26- Upgrade to at least `0.8.4`

Using newer compiler versions and the optimizer gives gas optimizations and additional safety checks for free.

## 27- Use `!= 0` instead of `> 0` for unsigned integer comparison

When dealing with unsigned integer types, comparisons with `!= 0` are cheaper then with ``> 0``.

## 28- Limit Modifiers

The code of modifiers is inlined inside the modified function, thus adding up size and costing gas. Limit the modifiers. Internal functions are not inlined, but called as separate functions. They are slightly more expensive at run time, but save a lot of redundant bytecode in deployment, if used more than once.


## 29- Packing booleans

In Solidity, Boolean variables are stored as uint8 (unsigned integer of 8 bits). However, only 1 bit would be enough to store them. If you need up to 32 Booleans together, you can just follow the Packing Variables pattern. If you need more, you will use more slots than actually needed.

Pack Booleans in a single uint256 variable. To this purpose, create functions that pack and unpack the Booleans into and from a single variable. The cost of running these functions is cheaper than the cost of extra Storage.

## 30- Mappings are cheaper than arrays

Solidity provides only two data types to represents list of data: arrays and maps. Mappings are cheaper, while arrays are packable and iterable.

In order to save gas, it is recommended to use mappings to manage lists of data, unless there is a need to iterate or it is possible to pack data types. This is useful both for Storage and Memory. You can manage an ordered list with a mapping using an integer index as a key.


## 31- Fixed size variables are cheaper than variable size.

Whenever it is possible to set an upper bound on the size of an array, use a fixed size array instead of a dynamic one.


## Credits

- https://eip2535diamonds.substack.com/p/smart-contract-gas-optimization-with
- https://blog.birost.com/a?ID=00950-e4e25dc9-573d-4332-8262-41131961734f
- https://marduc812.com/2021/04/08/how-to-save-gas-in-your-ethereum-smart-contracts/
- https://betterprogramming.pub/how-to-write-smart-contracts-that-optimize-gas-spent-on-ethereum-30b5e9c5db85
- https://mudit.blog/solidity-gas-optimization-tips/
- http://www.cs.toronto.edu/~fanl/papers/gas-brain21.pdf
- https://gist.github.com/hrkrshnn/ee8fabd532058307229d65dcd5836ddc
