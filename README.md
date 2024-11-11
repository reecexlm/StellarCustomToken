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
soroban-sdk = { version = "21.7.6", default-features = false } # ensure compatibility with WebAssembly by disabling the std lib

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
_env: &Env is the environment variable provided by Soroban but isn’t needed in this function.

```
use soroban_sdk::{contractimpl, Env};

pub struct MyCustomToken;

#[contractimpl]
impl MyCustomToken {
    pub fn name(_env: &Env) -> String {
        "MyCustomToken".to_string()
    }
}
```
#### 3. Write a Unit Test 
Add the following test code below the impl MyCustomToken block:
```
#[cfg(test)]
mod tests {
    use super::*;
    use soroban_sdk::testutils::Env as TestEnv;

    #[test]
    fn test_name() {
        let env = TestEnv::default();
        assert_eq!(MyCustomToken::name(&env), "MyCustomToken");
    }
}
```

#### 5. Run The Test
```
cargo test
```
