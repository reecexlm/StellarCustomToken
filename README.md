# StellarCustomToken
A step by step process to create a custom Token on Stellar using Soroban Smart Contracts
## Step 1: Project Setup

### 1.1 Create the Project Directory
```
cargo new my_custom_token --lib
cd my_custom_token
```
### 1.2: Update Cargo.toml
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
## Step 2: Implement Basic Functions
Now, we’ll implement the basic functions name, symbol, and decimals. These functions define the token’s basic identity information.
### 2.1 name Function
The name function returns the name of the token, which can be displayed in wallets or other interfaces.
#### 1.	Open src/lib.rs.
#### 2.	Add the following code to define the token name
:star: _env: &Env is the environment variable provided by Soroban but isn’t needed in this function.

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
#### 3. Write a Unit Test 
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

#### 5. Run The Test
```
cargo test
```
