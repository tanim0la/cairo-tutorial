# Constructors in Cairo
A constructor is a one-time-call function executed during contract deployment to initialize state variables, perform contract setup tasks, make cross-contract interactions and so on.

In Cairo, constructors are defined using the `#[constructor]` attribute inside the `mod` block of a contract.

This article will cover how constructors work in Cairo, the rules for initializing contract state, and how constructor return values differ from Solidity’s.

## A Simple Cairo Constructor
Let’s take a simple Solidity contract that initializes a state variable `count` in its constructor:

```solidity
contract Counter {
    uint256 public count;

    constructor(uint256 _count) {
        count = _count;
    }
}
```

Here’s the equivalent Cairo version:

```rust
#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    fn get_count(self: @TContractState) -> felt252;
}

#[starknet::contract]
mod HelloStarknet {
    use starknet::storage::{StoragePointerReadAccess,StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        count: felt252
    }

    // ************ CONSTRUCTOR FUNCTION ************* //
    #[constructor]
    fn constructor(ref self: ContractState, _count: felt252) {
        self.count.write(_count);
    }

    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn get_count(self: @ContractState) -> felt252 {
            self.count.read()
        }
    }
}
```

The `#[constructor]` attribute in the above code marks the function as the contract’s constructor. The function must be named `constructor` and is executed once during deployment. It takes `ref self` to allow write access to the contract’s storage, along with any parameters needed for initialization, in this case, `_count`.

The constructor above simply initializes the `count` storage variable with the value passed as an argument.

Let’s test it out, create a new project with Scarb:

```bash
scarb new counter
```

Next, replace the generated code in the `src/lib.cairo` file with the `HelloStarknet` contract code from above.

To verify that `count` is correctly initialized, we will write a test that deploys the contract with a specific value and then asserts that the stored value matches what we passed.

Open the test file (`tests/test_contract.cairo`), then replace the generated code with the one below:

```rust
use counter::{IHelloStarknetDispatcher, IHelloStarknetDispatcherTrait};
use snforge_std::{ContractClassTrait, DeclareResultTrait, declare};
use starknet::ContractAddress;

fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();

    // CREATE ARGUMENT ARRAY FOR CONSTRUCTOR. WE'RE PASSING `5` AS THE INITIAL VALUE FOR `count`
    let mut args = ArrayTrait::new();
    args.append(5);

    // DEPLOY THE CONTRACT WITH THE PROVIDED CONSTRUCTOR ARGUMENTS
    let (contract_address, _) = contract.deploy(@args).unwrap();

    contract_address // Return address of deployed contract
}

#[test]
fn test_count() {
    let contract_address = deploy_contract("HelloStarknet");

    let dispatcher = IHelloStarknetDispatcher { contract_address };

    // CALL THE `get_count` FUNCTION TO READ THE CURRENT VALUE OF `count`
    let result = dispatcher.get_count();

    // ASSERT THAT THE INITIALIZED VALUE MATCHES WHAT WE PASSED DURING DEPLOYMENT
    assert!(result == 5, "failed {} != 5", result);
}
```

The key part of this test is how we pass the constructor argument **as an array of `felt252` values** during deployment. This part is easy to miss, but it is important; we append the value `5` to the `args` array before calling `deploy`, which is how we initialize the contract with `5`.

```rust
// CREATE ARGUMENT ARRAY FOR CONSTRUCTOR. WE'RE PASSING `5` AS THE INITIAL VALUE FOR `count`
let mut args = ArrayTrait::new();
args.append(5);

// DEPLOY THE CONTRACT WITH THE PROVIDED CONSTRUCTOR ARGUMENTS
let (contract_address, _) = contract.deploy(@args).unwrap();
```

The rest of the test confirms that this initialization worked as expected. We call `get_count`, which returns the current value of the `count` variable, and then assert that it's equal to `5`.

Unlike Solidity, where constructor arguments can be of any supported type, such as integers, addresses, strings, structs, or arrays, Cairo requires all constructor arguments to be passed as `felt252` values during deployment.

This is because `felt252` is the fundamental type that CairoVM understands, and Starknet CLI serializes all constructor parameters as `felt252`. We cannot directly pass `ContractAddress` or other types during the deployment process. However, if we need to initialize other type(s), we can manually encode them as separate `felt252` values and decode them inside the constructor.

## Passing Primitive Types Other Than `felt252` to the Constructor
As mentioned earlier, we can manually encode non-`felt252` values (such as `ContractAddress`) into `felt252` during deployment, pass them as an array of `felt252`, and then decode them in the constructor to initialize the contract’s state. Let’s look at an example to see how this works in practice.

The Solidity version:

```solidity
contract SomeContract {
    uint256 public count;
    address public owner;
    bool public isActive;

    constructor(uint256 _count, address _owner, bool _isActive) {
        count = _count;
        owner = _owner;
        isActive = _isActive;
    }
}
```

Here’s the equivalent in Cairo:

```rust
use starknet::ContractAddress;

#[starknet::interface]
pub trait ISomeContract<TContractState> {
    fn get_count(self: @TContractState) -> u256;
    fn get_owner(self: @TContractState) -> ContractAddress;
    fn get_bool(self: @TContractState) -> bool;
}

#[starknet::contract]
mod SomeContract {
    use starknet::ContractAddress;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    // Define the contract's storage.
    #[storage]
    struct Storage {
        count: u256,
        owner: ContractAddress,
        is_active: bool,
    }

        // CONSTRUCTOR FUNCTION
        #[constructor]
        fn constructor(
                ref self: ContractState,
                _count_high: u128,
                _count_low: u128,
                _owner: ContractAddress,
                _is_active: bool
            ) {

            // (high * 2^128) + low
            let _count: u256 = (_count_high.into() * 2_u256.pow(128)) + _count_low.into();

            // INIT STATE VARS
            self.count.write(_count);
            self.owner.write(_owner);
            self.is_active.write(_is_active);
        }
}
```

Here the constructor accepts four arguments:

- `_count_high` and `_count_low`: Two 128-bit values that together represent a single 256-bit integer (u256). This separation is necessary because any value larger than the maximum representable `felt252` cannot be passed directly as a constructor argument in Cairo. By splitting the 256-bit value into two 128-bit halves, it becomes possible to transmit and reconstruct it safely within the constructor.
- `_owner`: The address of the contract owner.
- `_is_active`: A boolean flag indicating whether the contract should start in an active state.

Because Cairo does not natively support bit-shifting operators like `<<`, the code combines the two 128-bit segments into a single `u256` using arithmetic instead. The expression below:

```rust
// (high * 2^128) + low
(_count_high.into() * 2_u256.pow(128)) + _count_low.into()
```

is equivalent to shifting `_count_high` left by 128 bits and then adding `_count_low`. The result, `_count`, is stored as the contract’s initial counter value.

Finally, the constructor writes each of the provided arguments into their corresponding storage variables using the `.write()` method, thereby initializing the states on deployment.

### Encoding and Passing Constructor Arguments

The function below `deploy_contract` shows how values that are not of type `felt252` are encoded before being passed as constructor arguments:

```rust
fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();

    // CREATE ARGUMENT ARRAY FOR CONSTRUCTOR
    let mut args = ArrayTrait::new();

    // VALUES TO ENCODE
    // count = max value of u256
    let count = 115792089237316195423570985008687907853269984665640564039457584007913129639935;
    let owner:ContractAddress = 0xbeef.try_into().unwrap();
    let is_active = true;

    // ENCODE INTO FELT252, THEN APPEND TO `args` ARRAY
    args.append(count.high.into());    // count_high (u128) -> felt252
    args.append(count.low.into());     // count_low (u128) -> felt252
    args.append(owner.into());         // ContractAddress -> felt252
    args.append(is_active.into());     // bool -> felt252

    // DEPLOY THE CONTRACT WITH THE PROVIDED CONSTRUCTOR ARGUMENTS
    let (contract_address, _) = contract.deploy(@args).unwrap();
    contract_address
}
```

When dealing with multiple constructor arguments in Starknet, they must be passed as an `Array<felt252>`. In this example, the count value is split into two 128-bit halves; `count_high` and `count_low` before being appended to the array. As we said earlier, this is because a single `felt252` cannot safely hold a full 256-bit integer. By encoding each half as a separate `felt252`, the constructor can later reconstruct the original `u256` value inside the contract.

Here's what is happening with each type conversion:

- `u128` to `felt252` and `ContractAddress` to `felt252`: Uses `.into()` because this conversion is always safe and infallible
- `bool` to `felt252`: Uses `.into()` because booleans are represented as 0 or 1, which always fit in `felt252`

After conversion, we appended each encoded value to the `args` array in the exact order that matches the constructor's parameter sequence. This ordering is important as the deployment will fail or produce unexpected behavior if the arguments don't align or correspond with the constructor's expected parameter order.

The final step uses the `@` operator when passing the array to `contract.deploy(@args)`, which is Starknet's way of passing array data without transferring ownership.

## Passing Complex Type

Unlike Solidity, where you can pass structs and arrays seamlessly as constructor parameters, Cairo does not support passing complex types into constructors directly, such as:

- Custom structs
- Dynamic arrays
- Enums with data

However, if we want to pass these types to the constructor, the current working approach is:

1. Declare each value as its own constructor argument (flatten the complex types into primitives)
2. Reconstruct the complex type manually from those arguments, in the constructor.

Let’s consider this Solidity contract below that initializes a complex type (a `struct`) via the constructor:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.30;

contract Bank {
    struct Account {
        address wallet;
        uint64 balance;
    }

    Account userAccount;

    constructor(Account memory _userAccount) {
        userAccount = _userAccount;
    }
}
```

This is straightforward in Solidity; the `Account` struct is passed directly as an argument to the constructor, and Solidity handles the rest.

However, in Cairo, we don’t have that luxury. Complex types like structs can’t be passed directly to a constructor. Each field must be flattened into separate parameters, and the struct is then manually reconstructed inside the constructor before being written to storage.

In the example below, the `Account` struct has two fields; `wallet` and `balance`. Both are passed individually to the constructor as `_wallet` and `_balance`. The constructor then recreates the `Account` struct and stores it in the contract’s state:

```rust
#[starknet::contract]
mod Bank {
    use starknet::ContractAddress;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    // `Account` STRUCT
    #[derive(Drop, starknet::Store)]
    pub struct Account {
        pub wallet: ContractAddress,
        pub balance: u64,
    }

    #[storage]
    struct Storage {
        // Use the `Account` struct in storage
        pub user_account: Account,
    }

    // Constructor function
    #[constructor]
    // Each field is declared as its own constructor argument
    fn constructor(
            ref self: ContractState,
            _wallet: ContractAddress,
            _balance: u64
        ) {
        // Reconstruct the struct manually from those arguments
        let new_user_account = Account { wallet: _wallet, balance: _balance };

        // WRITE `new_user_account` STRUCT TO STORAGE
        self.user_account.write(new_user_account);
    }
}
```

Remember, the `starknet::Store` trait we added above the `Account` struct is what tells the Cairo compiler how to handle our struct in storage. Without it, the compiler won't know how to serialize/deserialize our struct for storage operations.

**Exercise:** Solve the `constructor` exercise in [the Cairo-Exercises](https://github.com/RareSkills/Cairo-Exercises) repo.

## **Return Values in Constructors**

In Solidity, a constructor never returns a value. During deployment, the EVM executes the constructor and treats its only “output” as the runtime bytecode to store on-chain.

Cairo on the other hand works differently. After deployment, it returns a tuple: `(ContractAddress, Span<felt252>)`.

- `ContractAddress`: the deployed contract’s address.
- `Span<felt252>`: a span of `felt252` values holding the constructor’s return data. Any type other than `felt252` is automatically converted before being placed here.

To demonstrate this, lets bootstrap a new scarb project:

```bash
scarb new rett
```

Then we add a constructor to our generated contract in `lib.cairo`, like so:

```rust

#[starknet::interface]
pub trait IHelloStarknet<TContractState> {
    fn increase_balance(ref self: TContractState, amount: felt252);
    fn get_balance(self: @TContractState) -> felt252;
}

#[starknet::contract]
mod HelloStarknet {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        balance: felt252,
    }

    // ************************ NEWLY ADDED - START ***********************//
    #[constructor]
    fn constructor(ref self: ContractState) -> felt252 {
        33
    }
    // ************************ NEWLY ADDED - END ************************//


    #[abi(embed_v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn increase_balance(ref self: ContractState, amount: felt252) {
            assert(amount != 0, 'Amount cannot be 0');
            self.balance.write(self.balance.read() + amount);
        }

        fn get_balance(self: @ContractState) -> felt252 {
            self.balance.read()
        }
    }
}
```

To keep things simple, our constructor will just return the value `33`.

To show that the constructor actually returns a value, let’s navigate to the test file `test_contract.cairo` and replace the generated code with this (*simplified the code for readability*):

```rust
use rett::{IHelloStarknetDispatcher, IHelloStarknetDispatcherTrait};
use snforge_std::{ContractClassTrait, DeclareResultTrait, declare};
use starknet::ContractAddress;

fn deploy_contract(name: ByteArray) -> ContractAddress {
    let contract = declare(name).unwrap().contract_class();
    let (contract_address, _) = contract.deploy(@ArrayTrait::new()).unwrap();
    contract_address
}

#[test]
fn test_increase_balance() {
    let contract_address = deploy_contract("HelloStarknet");
}
```

We will make changes to the highlighted part in this screenshot (*ignore the fire in my edito*r):

![a screenshot of the function to deploy a contract with the updated parts highlighted](https://r2media.rareskills.io/CairoConstructor/image1.png)

**The updated test code**

What changed are:

1. Return type: `deploy_contract` function now returns a tuple `(ContractAddress, felt252)` instead of just `ContractAddress`.
2. Constructor output capture: Introduced `ret_vals` of type `Span<felt252>` to hold the constructor’s return values.
3. Tuple return: We return the contract address alongside the first element of `ret_vals`, since the constructor only returns a single value.

Finally, the test asserts that the constructor’s return value is `33`, confirming that the value is correctly passed back during deployment.

```rust
use rett::{IHelloStarknetDispatcher, IHelloStarknetDispatcherTrait};
use snforge_std::{ContractClassTrait, DeclareResultTrait, declare};
use starknet::ContractAddress;

// Change return type to a tuple so we can capture the constructor’s return value.
fn deploy_contract(name: ByteArray) -> (ContractAddress, felt252) {
    let contract = declare(name).unwrap().contract_class();

    // Capture both the contract address and the constructor’s return values (as a Span<felt252>).
    let (contract_address, ret_vals) = contract.deploy(@ArrayTrait::new()).unwrap();

    // Return the address plus the first element in ret_vals (we expect only one value).
    (contract_address, *ret_vals.at(0))
}

#[test]
fn test_increase_balance() {
    let (_, ret_val) = deploy_contract("HelloStarknet");

        // Verify that the constructor actually returned 33 as expected.
    assert(ret_val == 33, 'Invalid return value.');
}
```

To confirm, run `scarb test`, the test should pass. In a later article we will see how to deploy a contract directly from another.

## Is There a Payable-Like Constructor?
While STRK behaves like an ERC-20 token, it also serves as Starknet’s native fee token. However, Starknet does not have a true “native token” in the same sense as Ethereum’s ETH. As a result, Cairo does not support “payable” constructors. If we want to enforce that a contract has a certain STRK balance at deployment, we can transfer STRK to the predicted address, then assert in the constructor that the balance of the contract is at least the desired amount.