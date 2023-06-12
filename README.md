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

# Secureum Bootcamp
## Till Solidity 201

1. Read eth white paper
2. P2P network on TCP Port: 30303 on protocol named DevP2P
3. Decentralization: combination of Hardware, Software, Wetware (people, orgs)
4. ECDSA used to implement public key cryptography in eth. Elliptic curve with finite fields
5. private key -> 1 point of its elliptic curve -> public key -> keccak256 hash -> last 20 bytes -> address
6. Patricia tree => prefix tree: key prefixes keep increasing from root leaves determining path to data
7. eth uses Hexagonal Merkle Patricia Tree => each node has 16 children
8. EVM is Quasi Turing Complete as computations are Gas bounded
9. EVM stack has max 1024 elements each with max size 256 bits (word)
10. EVM ordering is Big-endian (most significant byte on lowest address and ...)
11. ORIGIN vs CALLER opcode??
12. 'constant' is more gas-efficient than 'immutable' coz immutable reserves a 256 bit storage slot always.
13. functions existing outside of contract are called free-functions. They're file-level. They implicitly are 'internal', and are included in all contracts that call them, similar to internal library functions
14. indexed params in events are bit more gaseous than non-indexed params
15. min:1, max: 256 constants can be specified in an enum
16. constructors are optional
17. receive() => eth send, no calldata, cannot return data ||| fallback() => no match, no receive(), can receive/return data, payable for ETH
18. functions/contracts/state variables are visible before declaration
19. Solidity doesn't support fixed point math. Need to use => PRBMath, DSMath etc
20. address payable can be converted to address and vice versa. Also ==, != etc operators can be used with 'address'. Also address.balance, address.code, address.codehash exist
21. address.transfer() REVERTS on destination contract consuming more than 2300 gas. address.send() DOES NOT REVERT, but returns 0,1. send() is lower level alt of transfer
22. staticcall is delegatecall which can only read state of caller contract and NOT MODIFY it. Return value check is important for these calls.
23. 'contract' type can be converted to and from 'address' type. 'contract' type has no operators, but can only be used to access its member functions, (public/external) state variables
24. bytes and byte[] are dynamic byte arrays. bytes >>> byte[] as byte[] wastes 31 bytes of memory for every array element due to padding rules
25. string => supports ascii + escape characters
26. unicode => supports any utf-8 sequence => needs to be prefixed with 'unicode' keyword
27. hex => supports hex digits => prefix with 'hex' keyword
28. function types => variable to which a function can be assigned, be used like other vars
29. 4 storage places in EVM: stack, memory, storage, calldata
30. calldata: non-modifiable non-persistent, used mainly for storing arguments of external fn calls but can be used for other vars too
31. Assignment semantics: memory-memory and storate-storage => reference, storage-memory and others => copy
32. array members: .length, .push, .push(x), .pop || .push only increases size and returns reference to new element while .push(x) appends 'x' and returns nothing
33. string == bytes except it doesn't allow length/index access
34. memory arrays can ONLY BE FIXED SIZED.
35. Array Literals: comma separated list of elements in [ ]. For eg: [45, 89, 3]. Fixed one cannot be assigned to Dynamic one
36. array.push() is constant gas, array.pop() depends on size of element being removed
37. Array Slices only supported for 'calldata arrays'. X[start:end] => X[start] to X[end-1]
38. struct types CANNOT contain members of the same struct type.
39. mapping[key -> value]: key can ONLY be value type (not reference type), value can be ANY type. key is not really stored, its keccak256 hash is looked up. mappings cannot be (returned from functions and sent as arguments), also true for arrays/structs containing mappings.
40. delete essentially assigns default value to the var. delete arr[2] actually assigns 0 to 3rd element while keeping other 4 elements AND array length intact.
41. mappings inside structs won't be auto-set to default values when struct is delete'd
42. //exception to implicit conversions (done by compiler): int128 -> uint256, uint256 -> uint128 [2nd one doesn't make sense to me]
43. explicit conversions: most types eg: uint16 -> uint8 => higher order bits (left-side) are cut off || uint8 -> uint16 => padded on left. EXACT OPPOSITE for fixed-size BYTES: bytes16 -> bytes8 => lower order bits (right-side) are cut off || bytes8 -> bytes16 => padded on right
44. Literals (implicit) Conversions: decimals/hex -> integers works. (hex/string/hex string) -> fixed byte arrays IFF hex digits, #chars (for string) exactly fit
45. msg.sig: function signature in hex
46. abi.(encode, decode, encodeWithSelector, encodeWithSignature, encodePacked)
47. abi.encodePacked encodes the args WITHOUT padding => ambiguous
48. Math/Crypto fns: addmod(), mulmod(), keccak256(bytes memory), sha256(b m), ripemd160(b m), ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s). ecrecover() returns the address that signed the signature from sig details
49. ecrecover() is vulnerable to sig malleability attack i.e. 2 valid sigs possible coz of 2 possible values of s in v, r, s, 1 each in lower order and higher order. Openzeppelin's ECDSA fixes this by specifying v=27/28, s=lower
50. ETH sent via selfdestruct to a contract DOES NOT TRIGGER its receive()