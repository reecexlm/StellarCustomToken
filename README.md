# StellarCustomToken tutorial
A step by step process to create a custom Token on Stellar using Soroban Smart Contracts
## Setup Tools
### install stellar-cli
```
brew install stellar-cli
```

### install rust
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
### Install the wasm32-unknown-unknown target.
```
rustup target add wasm32-unknown-unknown
```

### configure the stellar cli for testnet
```
stellar network add \
  --global testnet \
  --rpc-url https://soroban-testnet.stellar.org:443 \
  --network-passphrase "Test SDF Network ; September 2015"
```
### configure an identity that will be used to sign the transactions
```
stellar keys generate --global alice --network testnet
```
you can see the public keys using the following command
```
stellar keys address alice
```

### create the Project Directory
```
cargo new my_custom_token --lib
cd my_custom_token
```
## create custom token contract

### update Cargo.toml
```
[package]
name = "my_custom_token"  # The name of the crate (project). This will be used as the library name if used as a dependency
version = "0.1.0"         # version of this crate
edition = "2021"          # Rust edition.  

[dependencies]
soroban-sdk = { version = "21.7.6", features = ["alloc"] }

[dev-dependencies]
soroban-sdk = { version = "21.7.6", features = ["testutils"] }

[features]
testutils = ["soroban-sdk/testutils"]

[lib]
crate-type = ["cdylib"] # C-compatible dynamic library output required for wasm output in Soroban smart contracts.

[profile.release]
codegen-units = 1 # number of parallel codegen units
lto = true        # Enables link-time optimization
opt-level = "z"   # optimization level to "z" (reduce size of wasm)
```
## implement the contract functions and tests
We will start with some basic functions name, symbol, and decimals. These functions define the token’s basic identity information.
### name Function
The name function returns the name of the token, which can be displayed in wallets or other interfaces.
:star: Note: _env: &Env is the environment variable provided by Soroban but isn’t needed in this function.

```
#![no_std]  
use soroban_sdk::{contract, contractimpl, Env, String};

#[contract]
pub struct MyCustomToken;

#[contractimpl]
impl MyCustomToken {

    // Return token name
    pub fn name(env: &Env) -> String {
        String::from_str(&env, "MyCustomToken")
    }
}
```
### add a unit test 
Add the following test code below the impl MyCustomToken block in `src/lib.rs`:
```
#[cfg(test)]
mod test {
    use super::*;
    use soroban_sdk::{Env};
    use soroban_sdk::String as SorobanString;

    #[test]
    fn test_name() {
        let env = Env::default();
        let expected_name = SorobanString::from_str(&env, "MyCustomToken");
        assert_eq!(MyCustomToken::name(&env), expected_name);
    }
}
```

### run The test
output should look similar to the following
```
❯ cargo test
   Compiling my_custom_token v0.1.0 ([path to your project]/my_custom_token)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.03s
     Running unittests src/lib.rs (target/debug/deps/my_custom_token-5901d5dafea4014c)

running 1 test
test test::test_name ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### add Initialize, balance and mint Functions
Let's add a function initialize and mint functions. The initialize function is responsible for setting up the contract's initial state by storing an admin address. This function is designed to be called only once, typically when the contract is deployed or first used.  It allows the caller to specify an admin address, which will be stored in the contract’s storage. This admin is the only account authorized to perform certain privileged actions, such as minting new tokens using the mint contract function.  We also add a balance function which retrieves the token balance for a specified Address from persistent storage, returning 0 if no balance is found. This function is essential for tracking individual account balances within the contract.

#### add initialize function
add this function to `src/lib.rs`
```
    // Initialize the contract with an admin address
    pub fn initialize(env: &Env, admin: Address) {
        admin.require_auth();
        env.storage().instance().set(&symbol_short!("admin"), &admin);
    }
```

#### add balance function
```
    // Get balance for any address
    pub fn balance(env: &Env, id: Address) -> i128 {
        env.storage()
            .persistent()
            .get(&(symbol_short!("balance"), &id))
            .unwrap_or(0)
    }
```

#### add mint function
```
// Mint new tokens (only admin can call this)
    pub fn mint(env: &Env, to: Address, amount: i128) {
        let admin: Address = env.storage().instance().get(&symbol_short!("admin")).unwrap();
        admin.require_auth();
        let balance = Self::balance(env, to.clone());
        env.storage().persistent().set(&(symbol_short!("balance"), &to), &(balance + amount));
    }
```

#### update imports
you will also need to update your imports
```
use soroban_sdk::{contract, contractimpl, Env, String, Address, symbol_short};
```

#### add unit test for mint
```
#[test]
    fn test_mint() {
        let env = Env::default();
        let contract_id = env.register_contract(None, MyCustomToken);
        let client = MyCustomTokenClient::new(&env, &contract_id);

        // Create test addresses
        let admin = Address::generate(&env);
        let recipient = Address::generate(&env);

        // Set up authentication
        env.mock_all_auths();

        // Initialize the contract with admin
        client.initialize(&admin);

        // Verify no initial balance
        assert_eq!(client.balance(&recipient), 0);

        // Mint tokens
        let amount1: i128 = 1000;
        client.mint(&recipient, &amount1);

        // Verify balance after first mint
        assert_eq!(client.balance(&recipient), 1000);

        // Mint more tokens
        let amount2: i128 = 500;
        client.mint(&recipient, &amount2);

        // Verify final balance
        assert_eq!(client.balance(&recipient), 1500);
    }
```
    
---- [rest WIP] ----

### build the contract
```
RUSTFLAGS="-C target-feature=-reference-types" cargo build --target wasm32-unknown-unknown --release
```

### optimize the wasm
```
❯ stellar contract optimize --wasm target/wasm32-unknown-unknown/release/my_custom_token.wasm
Reading: target/wasm32-unknown-unknown/release/my_custom_token.wasm (4242 bytes)
Optimized: target/wasm32-unknown-unknown/release/my_custom_token.optimized.wasm (2021 bytes)
```

### deploy the contract
the contract id will be displayed at the end. You will use this id to execute contract functions.
```
❯ stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/my_custom_token.wasm \
  --source alice \
  --network testnet
ℹ️ Simulating install transaction…
🌎 Submitting install transaction…
ℹ️ Using wasm hash a6138aca80985c4cf164596d4b10bd35da5b6a8ff6f281589ef0b610e41385cd
ℹ️ Simulating deploy transaction…
🌎 Submitting deploy transaction…
ℹ️ Transaction hash is aeed7863fb5a420036b84520a6bb8424c0e1036045625d1bc236adb6ca50a8da
🔗 https://stellar.expert/explorer/testnet/tx/aeed7863fb5a420036b84520a6bb8424c0e1036045625d1bc236adb6ca50a8da
🔗 https://stellar.expert/explorer/testnet/contract/CAG72R52BC3ACZ25B6XHHY7FHBQXOYEP6MZHC4TWANW2T64OG5XR6WJ2
✅ Deployed!
CC3P7PQ34FJJXJCSXVJBG7EJMWPEDT2H44SQSQDTXKDRWSKWBBXQ2D5Y
```

## interact with the contract

### invoke the name function
```
❯ stellar contract invoke \
  --id CC3P7PQ34FJJXJCSXVJBG7EJMWPEDT2H44SQSQDTXKDRWSKWBBXQ2D5Y \
  --source alice \
  --network testnet \
  -- \
  name
ℹ️ Send skipped because simulation identified as read-only. Send by rerunning with `--send=yes`.
"MyCustomToken"
```

### add the custom token to freighter
select manage assets -> add asset -> add asset manually
enter the contract id that you deployed.
<img width="359" alt="image" src="https://github.com/user-attachments/assets/c76018cc-67fa-4b54-9661-683f8faea9e9">
<img width="359" alt="image" src="https://github.com/user-attachments/assets/abba8da9-93dc-4486-af90-0103a5ccf2c4">

### mint tokens to your freighter wallet
```
stellar contract invoke \
    --id CC3P7PQ34FJJXJCSXVJBG7EJMWPEDT2H44SQSQDTXKDRWSKWBBXQ2D5Y \
    --source alice \
    --network testnet \
    -- mint \
    --amount 1000000000 \
  --to GD5Y45RYKDO4WKQN5PZNPXU53LZT34G4D3WN26M3A7333SNXSYX4IH3T
```
<img width="359" alt="image" src="https://github.com/user-attachments/assets/c306b372-b7c9-45d3-a29f-dc6a2db3a259">





