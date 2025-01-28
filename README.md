## 1. Goal
* The goal is to create a custom Polkadot SDK Pallet that acts as an NFT Marketplace. Our NFTs will represent kitties, which will be a digital pets that can be created, traded, and more.
* This Pallet could then be included into a Polkadot SDK blockchain and used to launch a Web3 application on the Polkadot Network.

## 2. Runtime
* At the heart of a blockchain is a state transition function (STF). This is the logic of the blockchain, and defines all the ways a blockchain is allowed to manipulate the blockchain state.
* In the polkadot-sdk we refer to this logic as the blockchain's `runtime`.

## 3. NFTs
Non-Fungible Tokens (NFTs) are a type of token which can be created and traded on a blockchain. As their name indicated, each NFT is totally unique, and therefore non-fungible with one another. NFTs can be used for many things, for example: Representing real world assets, Ownership Rights, Access Rights, Digital assets, Music, Images etc.

## 4. Macros in FRAME
* **FRAME** uses Macros to simplify the development of Pallets, while keeping all of the benefits of using Rust. You can identify most macros in one of two forms:
  * `#[macro_name]`: **Attribute macros***, which are applied on top of valid rust syntax.
  * `macro_name!(...)`: **Declarative macros**, which can define their own internal syntax.
* The entrypoint for all the `FRAME` macros looks like this:

  ```rust
    #[frame::pallet(dev_mode)]
    pub mod pallet {
        // -- snip --
    }
  ```

* We wrap all of our Pallet code inside of this entrypoint, which allows our macros to have context of all the details inside. Without this, there would be no way for the macros defined inside the entry point to to communicate information to one another.
The unfortunate limitation here is that wherever we want to use `FRAME` macros, we must basically do it in a single file and all enclosed by the `#[frame::pallet]` macro entrypoint.

## 5. Basic Pallet Structure

```rust
    use frame::prelude::*;
    pub use pallet::*;

    #[frame::pallet]
    pub mod pallet {
        use super::*;

        #[pallet::pallet]
        pub struct Pallet<T>(core::marker::PhantomData<T>);

        #[pallet::config]  // snip
        #[pallet::event]   // snip
        #[pallet::error]   // snip
        #[pallet::storage] // snip
        #[pallet::call]    // snip
    }
```

### 5.1 Pallet Struct (`#[pallet::pallet]`)
* The Pallet struct is the anchor on which we implement all logic and traits for our Pallet.

```rust
    #[pallet::pallet]
    pub struct Pallet<T>(core::marker::PhantomData<T>);

    // Function implementations
    impl<T: Config> Pallet<T> {
        // -- snip --
    }

    // Trait implementations
    impl<T: Config> Hooks for Pallet<T> {
        fn on_finalize() {
            // -- snip --
        }
    }

    ...

    // And we can access these functions inside trait implementations as follows:
    pallet_kitties::Pallet::<T>::on_finalize();
```

* In fact, many traits are automatically implemented on top of `Pallet` and are accessible thanks to the `#[pallet::pallet]` attribute macro. You can see all the different traits implemented on Pallet by looking at the **autogenerated Rust docs**.

### 5.2 Callable Functions (`#[pallet::call]`)
There are two types of functions exposed by Pallets:
* Internal Functions: Regular functions only callable from within the blockchain.
* Callable Functions: The way users interact with the blockchain is through transactions. Those transactions are processed, and then dispatched to callable functions within the blockchain.

#### 5.2.1 Pallet Call Macro
* `FRAME` allows you to create callable functions by introducing the `#[pallet::call]` macro on top of a normal function implementation code block.

```rust
    #[pallet::call]
    impl<T: Config> Pallet<T> {
        // All of the functions in this `impl` will be callable by users.
    }
```

* The macro enforces rules on how these functions should be defined. They are discussed in the following sections.
* **NOTE**: "Users submit an extrinsic to the blockchain, which is dispatched to a Pallet call."
* **NOTE**: An `extrinsic` is any message from the outside coming to the blockchain. A `transaction` is specifically a signed message coming from the outside.

#### 5.2.2 Origin
* The first parameter of every callable function must be `origin: OriginFor<T>`.
* It describes where the call is calling from, and allows us to perform simple access control logic based on that information.
* The most common origin is the signed origin, which is a regular transaction.
* Origin is a superset of the idea of `msg.sender` (from Smart contract development). No longer do we need to assume that every call to a callable function is coming from an external account. We could have pallets call one another, or other internal logic trigger a callable function.

#### 5.2.3 Dispatch Result
* Every callable function must return a DispatchResult, which is simply defined as:

```rust
    pub type DispatchResult = Result<(), sp_runtime::DispatchError>;
```

* So these functions can return `Ok(())` or some `Err(DispatchError)`. You can easily define new `DispatchError` variants using the included `#[pallet::error]`.

### 5.3 Pallet Config (`#[pallet::config]`)
* Each pallet includes a **trait** `Config` which is used to configure the pallet in the context of your larger runtime.

```rust
    #[pallet::config]
    pub trait Config: frame_system::Config {
        // -- snip --
    }
```

* Wherever you see `T` (from `<T: Config>`), you have access to our `trait Config` and the types and functions inside of it.

### 5.4 Pallet Events (`#[pallet::event]`)
* Events allow Pallets to express that something has happened, and allows off-chain systems like indexers or block explorers to track certain state transitions.
* The `#[pallet::event]` macro acts on an enum Event.

```rust
    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
        Created { owner: T::AccountId },
    }
```

* The macro `#[pallet::generate_deposit(pub(super) fn deposit_event)]` generates a helper function that handles event conversion and system integration.

```
Pallet Event                              // Your event
    ↓ (From conversion)
Pallet's Runtime Event                    // Runtime-level event
    ↓ (Into conversion)
frame_system::Config::RuntimeEvent        // System-level event
    ↓
Stored in block's event log
```

### 5.5 Storage (`#[pallet::storage]`)

#### 5.5.1 Storage Values
* The most basic storage type for a blockchain is a single `StorageValue`. A `StorageValue` is used to place a single object into the blockchain storage. A single object can be as simple as a single type like a `u32`, or more complex structures, or even vectors.
* `StorageValue` places a single entry into the merkle trie. So when you read data, you read all of it. When you write data, you write all of it. This is in contrast to a `StorageMap`.
* Here is how it is created in `FRAME`:

```rust
    #[pallet::storage]
    pub(super) type CountForKitties<T: Config> = StorageValue<Value = u32>;
```

* Our storage (`CountForKitties`) is a type alias for a new instance of `StorageValue`.
* `StorageValue` has a parameter `Value` where we can define the type we want to place in storage. In this case, it is a simple `u32`.
* Also notice `CountForKitties` is generic over `<T: Config>`. **All of our storage must be generic** over `<T: Config>` even if we are not using it directly. **Macros** use this generic parameter to fill in behind the scene details to make the `StorageValue` work.
* Visibility of the type is up to you and your needs, but you need to remember that blockchains are public databases. So `pub` in this case is only about Rust, and allowing other modules to access this storage and its APIs directly. You cannot make storage on a blockchain "private", and even if you make this storage without `pub`, there are low level ways to manipulate the storage in the database.
* `StorageValue` has a `QueryKind` parameter which defaults to `OptionQuery`. We can change this to `ValueQuery` to return the default values in case the value is not found to get rid of the `unwrap_or()` in the `get()` calls. If your type does not implement `Default`, you can't use `ValueQuery`. A common example of this is the `T::AccountId` type, which purposefully has no default value, and thus is not compatible out of the box with `ValueQuery`.
* `OnEmpty` type defines the behavior of the `QueryKind` type. When you call get, and the storage is empty, the `OnEmpty` configuration kicks in.You **CAN** modify `OnEmpty` to return a custom value, rather than Default.

#### 5.5.2 Storage Maps
* A `StorageMap` is a key-value store. Declaring a new StorageMap is very similar to a StorageValue:

```rust
    #[pallet::storage]
    pub(super) type Kitties<T: Config> = StorageMap<Key = [u8; 32], Value = ()>;
```

* A `StorageValue` stores a single value into a single key in the Merkle Trie. A `StorageMap` stores multiple values under different storage keys, all into different places in the Merkle Trie.
* `StorageValue` puts all of the data into a single object and stores that all into a single key in the Merkle Trie. In `StorageMap`, each value is stored in its own spot in the Merkle Trie, so you are able to read just one key / value on its own. This can be way more efficient for reading just a single item. However, trying to read multiple items from a `StorageMap` is extremely expensive.
* **Ensure**: `ensure!` is a macro which expands to the following:

```rust
    ensure!(!Kitties::<T>::contains_key(my_key), Error::<T>::DuplicateKitty);
    //              |   |
    //              |   |  This gets expanded to the following:
    //              V   V
    if (Kitties::<T>::contains_key(my_key)) {
        return Err(Error::<T>::DuplicateKitty.into());
    }
```

* **NOTE**: In both `StorageValue` and `StorageMap`, the `insert` API cannot fail. If you try to insert a value into a key that already exists, it will overwrite the existing value.
* **NOTE**: To check if a value exists, you can use the `exists` API for `StorageValue` and `contains_key` for `StorageMap`.

#### 5.5.3 Traits Required for Storage
* When storing a custom struct in runtime storage like the `Kitty` struct, you need to implement the following traits on the struct:
  * `Encode`: The object must be encodable to bytes using `parity_scale_codec`.
  * `Decode`: The object must be decodable from bytes using `parity_scale_codec`.
  * `MaxEncodedLen`: When the object is encoded, it must have an upper limit to its size.
  * `TypeInfo`: The object must be able to generate metadata describing the object.
* We can use the `#[derive(...)]` macros to generate the implementation of these traits.

#### 5.5.4 Parity Scale Codec (`parity_scale_codec`)
`SCALE` defines how every object in the `polkadot-sdk` is represented in bytes. SCALE is:
* Simple to define.
* Not Rust-specific (but happens to work great in Rust).
  * Easy to derive codec logic: `#[derive(Encode, Decode)]`
  * Viable and useful for APIs like: `MaxEncodedLen` and `TypeInfo`
  * It does not use `Rust std`, and thus can compile to `Wasm no_std`.
* Consensus critical / bijective; one value will always encode to one blob and that blob will only decode to that value.
* Supports a copy-free decode for basic types on LE architectures (like Wasm).
* It is about as thin and lightweight as can be.

#### 5.5.5 Max Encoded Length (`MaxEncodedLen`)
* We track the maximum encoded length of an object: `MaxEncodedLen` and use that information to predict in the worst case scenario how much data will be used when we store it.
* For a `u8`, the `max_encoded_len()` is always: `1 byte`. For a `u64`, it is always: `8 bytes`. For a basic `enum`, it is also just `1 byte`, since an enum can represent up to 256 variants.
* For a `struct`, the `max_encoded_len()` will be the **sum** of the `max_encoded_len()` of all items in the struct.

#### 5.5.6 Type Info (`TypeInfo`)
* `TypeInfo` is a trait that is used to generate metadata about a type.
* This trait is key for off-chain interactions with your blockchain. It is used to generate metadata for all the objects and types in your blockchain. Metadata exposes all the details of your blockchain to the outside world, allowing us to dynamically construct APIs to interact with the blockchain.
* **Skip Type Params**: `TypeInfo` derive macro isn't very "smart". `TypeInfo` generates relevant metadata about the types used in your blockchain. However, part of our `Kitty` type is the generic parameter `T`, and it really does not make any sense to generate `TypeInfo` for `T`. To make `TypeInfo` work while we have `T`, we need to include the additional line:

```rust
    #[scale_info(skip_type_params(T))]
```

### 5.6 Pallet Errors (`#[pallet::error]`)
* **NOTE**: You cannot panic inside the runtime.
* All of our callable functions use the `DispatchResult` type. The `DispatchResult` type expects either `Ok(())` or `Err(DispatchError)`.

```rust
    // The `DispatchError` type has a few variants that you can easily construct / use.
    // For example, if you want to be a little lazy, you can simply return a `&'static str`:
    fn always_error() -> DispatchResult {
        return Err("this function always errors".into())
    }

    // But the better option is to return a custom Pallet Error:
    fn custom_error() -> DispatchResult {
        return Err(Error::<T>::CustomPalletError.into())
    }

    // Notice in both of these cases we had to call into() to convert our input type 
    // into the DispatchError type.
```

* To create `CustomPalletError`, simply add a new variants to the `enum Error<T>` type.

```rust
    #[pallet::error]
    pub enum Error<T> {
        /// This is a description for the error.
        ///
        /// This description can be shown to the user in UIs, so make it descriptive.
        CustomPalletError,
    }
```

## 6. Generating Unique DNAs for Kitties

### 6.1 Randomness
Generating randomness on a blockchain is extremely difficult because any kind of randomness function must generate exactly the same randomness for all nodes. **NOTE** that Polkadot does provide access to a verifiable random function (VRF).

### 6.2 Uniqueness
There are different levels of uniqueness we can achieve using data from our blockchain.
* `frame_system::Pallet::<T>::parent_hash()`: The hash of the previous block. This will ensure uniqueness for every fork of the blockchain.
* `frame_system::Pallet::<T>::block_number()`: The number of the current block. This will obviously be unique for each block.
* `frame_system::Pallet::<T>::extrinsic_index()`: The number of the extrinsic in the block that is being executed. This will be unique for each extrinsic in a block.
* `CountForKitties::<T>::get()`: The number of kitties in our blockchain.

### 6.3 Hash
* `FRAME` provides access to the hash function: `frame::primitives::BlakeTwo256`

```rust
    // Collect our unique inputs into a single object.
    let unique_payload = (item1, item2, item3);
    // To use the `hash_of` API, we need to bring the `Hash` trait into scope.
    use frame::traits::Hash;
    // Hash that object to get a unique identifier.
    let hash: [u8; 32] = BlakeTwo256::hash_of(&unique_payload).into();
```

* The `hash_of` API comes from the `Hash` trait and takes any `encode`-able object, and returns a `H256`, which is a 256-bit hash. As you can see in the code above, it is easy to convert that to a `[u8; 32]` by just calling `.into()`, since these two types are equivalent.

## 7. Redundant Storage (Track Owned Kitties)
As a rule, you only want to store data in your blockchain which is necessary for consensus. We should evaluate what queries we will need to perform on the stored data and then decide that we may indeed need to have redundant storage in order to make the queries efficient.

### 7.1 Iteration
In general iteration should be avoided where possible, but if unavoidable it is critical that iteration be bounded in size. We literally cannot allow code on our blockchain which would do unbounded iteration, else that would stall our blockchain, which needs to produce a new block on a regular time interval.

* **Iteration in Maps**: When you iterate over a map, you need to make 2 considerations: That the map may not have a bounded upper limit and That each access to the map is very expensive to the blockchain (each is a unique read to the merkle trie). If you want to do iteration, probably you do **NOT** want to use a map.
* **Iteration in Vectors**: When you iterate over a vector, the only real consideration you need to have is how large that vector is. Accessing large files from the database is going to be slower than accessing small files. Once you access the vector, iterating over it and manipulating it is relatively **cheap** compared to any kind of storage map (**but not zero**, complexity about vector access still applies). If you want to do iteration, you definitely would prefer to use a vector.

### 7.2 Storage Optimisations
For vectors, it is not necessary to read data from the storage, append to it and write it back to storage. Instead we can use `FRAME's` Storage Abstractions and simply append to the vector like below:

```rust
    // Naive way
    // Get/read the vector from storage
    let mut owned_kitties: Vec<[u8; 32]> = KittiesOwned::<T>::get(owner);
    // Append to the vector
    owned_kitties.append(new_kitty);
    // Write the vector back to storage
    KittiesOwned::<T>::insert(owner, owned_kitties);

    // Better way
    KittiesOwned::<T>::append(owner, new_kitty);
```

### 7.3 Bounded Vectors
* The `BoundedVec` type is a zero-overhead abstraction over the `Vec` type allowing us to control the maximum number of item in the vector. To create a new BoundedVec with a maximum of 100 u8s, you can do the following:

```rust
    let my_bounded_vec = BoundedVec::<u8, ConstU32<100>>::new();
```

* The syntax here is very similar to creating a `Vec`, however we include a second generic parameter which tells us the bound. The easiest way to set this bound is using the `ConstU32<T>` type.
* The `BoundedVec` type has almost all the same APIs as a `Vec`. The main difference is the fact that a `BoundedVec` cannot always accept a new item. So rather than having `push`, `append`, `extend`, `insert`, and so on, you have `try_push`, `try_append`, `try_extend`, `try_insert`, etc. So converting the logic of a `Vec` to a `BoundedVec` can be as easy as:

```rust
    // Append to a normal vec.
    vec.append(item);
    // Try append to a bounded vec, handling the error.
    bounded_vec.try_append(item).map_err(|_| Error::<T>::TooManyOwned)?;
```

* Just like for `Vec`, our `BoundedVec` also has an optimized `try_append` API for trying to append a new item to the `BoundedVec` without having to read the whole vector in the runtime:

```rust
    // Append to a normal vec.
    KittiesOwned::<T>::append(item);
    // Try append to a bounded vec, handling the error.
    KittiesOwned::<T>::try_append(item).map_err(|_| Error::<T>::TooManyOwned)?;
```

