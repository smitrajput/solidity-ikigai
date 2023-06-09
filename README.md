# solidity-ikigai
Esoteric Solidity gas-optimisations + security learnings.

## Solidity Doc Notes

1. Size of string literal “smit” is 4 bytes. Size of hex string literal hex”face” is 2 bytes.
2. contract X is MostBaseLike, MostDerived { constructor() {} }.
    1. order of contracts needs to be ‘written’ in the way shown above to allow solidity to compile. It basically says MostDerived contract needs to override MostBaseLike contract
    2. constructors will be called in the following order: MostBaseLike → MostDerived → X
3. bytes, strings are right-padded, when converting them to 32-bytes (during encoding for e.g)
4. abi.encode/encoeWithSelector/encodeWithSignature: static types encoded in place | position of dynamic types (relative to this encoding) | data of dynamic types (first length | then actual data). A call to a function with the signature `f(uint256,uint32[],bytes10,bytes)` with values `(0x123, [0x456, 0x789], "1234567890", "Hello, world!")` is encoded as 

```js
positions 0x8be65246 // selector
0x00  0000000000000000000000000000000000000000000000000000000000000123 // in-place 0x123
0x20  0000000000000000000000000000000000000000000000000000000000000080 // position of start of [0x456, 0x789]
0x40  3132333435363738393000000000000000000000000000000000000000000000 // "1234567890"'s ASCII right-padded
0x60  00000000000000000000000000000000000000000000000000000000000000e0 // position of start of "Hello, world!"
0x80  0000000000000000000000000000000000000000000000000000000000000002 //  start of [0x456, 0x789]
0xa0  0000000000000000000000000000000000000000000000000000000000000456
0xc0  0000000000000000000000000000000000000000000000000000000000000789
0xe0  000000000000000000000000000000000000000000000000000000000000000d // start of "Hello, world!"
0x100  48656c6c6f2c20776f726c642100000000000000000000000000000000000000
```

5. abi.encodePacked():
    1. all types except arrays are packed unpadded (even strings, bytes are unpadded unless inside arrays)
    2. all dynamic types are packed ***without*** length
    3. all elements in array are padded (even strings, bytes)
6. calldata access in Yul: `x.length` for its length, `x.offset` for its byte-offset. They can be assigned to, though without any validation to be less than `calldatasize()`
7. storage access in Yul: `x.slot` for its slot, `x.offset` for its byte-offset. They cannot be assigned to.
8. Enable auto-gas-optimisation by compiler using `—via-ir` for compiling, which is a new code generation path for converting solidity code to opcodes. It only works if there is no inline assembly or if there is, then it is `memory-safe`. Now, inline assembly is memory-safe when all the memory that is accessed inside the assembly block is:
    1. scratch space (`0x00 - 0x3f`)
    2. allocated by solidity, for e.g. accessing memory within bounds of pre-created memory array
    3. allocated by you, for e.g. {don’t this and (d) contradict?}
        
        ```solidity
        // allocating 'length' more bytes
        assembly {
        	let pos := mload(0x40)
          mstore(0x40, add(pos, length))
        }
        ```
        
    4. allocated by the free memory pointer (`0x40`), without updating the free memory pointer, for e.g. 
        
        ```solidity
        assembly ("memory-safe") {
          let p := mload(0x40)
          returndatacopy(p, 0, returndatasize())
          revert(p, returndatasize())
        }
        ```
        
    
    ***Hence, the best way to create most gas-optimised code is by using both `—via-ir` and `memory-safe-assembly`.***
    

### Issues / Potential PRs

1. [Link](https://docs.soliditylang.org/en/v0.8.20/abi-spec.html#types): `int<M>`: two’s complement signed integer type of `M` bits, `0 < M <= 256`, `M % 8 == 0.` How come `256` for `int`?
2. [Link](https://docs.soliditylang.org/en/v0.8.20/abi-spec.html#non-standard-packed-mode): ‘Furthermore, structs as well as nested arrays are not supported.’ Then also, ‘The encoding of `string` or `bytes` does not apply padding at the end, unless it is part of an array or ***struct*** (then it is padded to a multiple of 32 bytes)’. struct? Choose a side, bitch.
3. [Link](https://docs.soliditylang.org/en/v0.8.20/assembly.html#things-to-avoid): what exactly are we avoiding here?

## PwningEth Blog Summaries

1. Aurora: delegatecall precompile depositETH contract (from malicious contract) that emits legit looking event but doesn’t really check if ETH was actually deposited. Because msg.value on malicious contract will persist as msg.value for precompile, and because it’s a precompile (special contract that extends EVM), smh its logic==state, even though malicious contract’s state should have been retained (coz that’s how delegatecall works) and aurora engine should have been able to distinguish this event from event generated on normal call (but it doesn’t).
2. Moonbeam: delegatecall retains msg.sender, and for precompile contracts logic==state. Make innocent callers call malicious contract and use this call to delegatecall to ‘asset’ precompile contracts, to give the attacker infinite approvals on these assets..
3. Polkadot Frontier EVM: ETH balances limit → 256 bits while Polka substrate balances limit → 128 bits. Bypass EVM implementation of transfer using (1 << 128) to 0 truncate and then add more balance to give attacker huge balance on transfer of wrapped assets.