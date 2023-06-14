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
51. If there's a revert after selfdestruct() in the same txn, it can undo contract destruction, as the chain-state is reverted back.
52. type(myContractName).name, .creationCode, .runtimeCode
53. type(myInterfaceName).interfaceId
54. type(myInteger).max, .min => returns min and max value of int type of myInteger. For eg 0 and 255 for uint8 myInteger
55. non-booleans CANNOT be (implicitly) converted to boolean. For eg if(1) != if(true)
56. assert() => invariants, internal errors || require() => external/user interactions/dependency errors
57. Exceptions: error signatures: Error(string) => external/user interaction errors, Panic(uint256) => internal errors/assertions
58. Low-level-calls: call/staticcall/delegatecall RETURN TRUE even if destination contract DOES NOT exist
59. Panic Error Code: 0x01 - false arg in assert, 0x11 - over/underflow, 0x12 - div/mod by 0, 0x31: pop() empty array, 0x32: out-of-bounds
60. revert([String]) vs revert CustomError([arg1, ...])
61. catch blocks: catch Error(string reason), catch Panic(uint256 errorCode), catch (bytes lowLevelData), catch
62. function that are not implemented and meant to be overriden by its derived classes must be marked 'virtual'. Functions which ARE overriding these virtual functions need to be marked 'override'.
63. During multiple inheritance, specify contracts from left to right as 'most base-like' to 'most derived'
64. C3 linearization: to call a fn specified in multiple base classes, compiler starts checking inherited contracts from right to left as specified on the first line i.e. contract A is B, C, D {} -> first D is checked, then C, B
65. abstract: at least 1 fn unimplemented, interface: no fn implemented, no constructor,state vars,inheritance, all fns external, library: logic used by contracts using delegatecall
66. using A for B => fns in library A are now attached to type B. For eg. using SafeMath for uint256. Their scope is restricted to contract itself since 0.7, before it was inherited by child contracts
67. to call a fn exactly 1 level up inheritance heirarchy, super.fnName() is used
68. on overriding fns, mutability changes are allowed only to more stricter forms: public -> external, non-payable -> view/pure, view -> pure. payable can't be changed to anything
69. all fns in interface are virtual by default
70. public state variables can override external fns in base classes that have same name, parameters and return-type as getter fns of these public state variables. But public state vars themselves cannot be overriden.
71. modifiers can be overriden exactly the way fns can be.
72. constructor of base classes are called using C3 linearization
73. Libraries ARE STATELESS. They don't have state vars, can't receive ETH, can't be destroyed, can't inherit or be inherited. Can be called directly only for view/pure fns, need to delegatecall to change state. Calling contract can supply state vars tho.
74. structs and arrays always start with new storage slot (instead of getting packed inside previous slots even if space is available). And following items also always start with new storage slots
75. inherited state vars are stored in C3 linearized fashion: most-base to most-derived, and vars of diff contracts can be packed in 1 slot if space allows
76. While packing storage of state vars, consider BOTH sizes of vars AND read/write preferences of those vars. Coz packing together vars which are not required to be read/written together can result in increased gas costs, instead of decreased ones. As the only var to be read needs to be isolated from others using masking of other vars which requires additional gas costs
77. Dynamic array A -> stored at slot P. Then at P, current size of A is stored. A[0] is stored at keccak256(P), A[1] at keccak256(P)+1 and so on. These can be packed together too if possible
78. Mapping M -> slot P: P stores nothing. M[key:k] i.e. value corresponding to 1st key, is stored at keccak256(h(k).P). h is 32 padded if k is value type. [h is keccak256() if k is string/byte type. → highly doubt this as k can never be reference type]
79. bytes and string (S) storage at slot P: so P stores S.length*2 + S[0] if S[0] <= 31 bytes. And P stores only S.length*2 + 1 and S[0] is stored at keccack256(P) if S[0] >= 32 bytes. Now in P, if right-most digit (lowest bit) is 1 => S[0] storage starts at keccack256(P) and if right-most digit is 0 => S[0] storage starts at P {corresponding to S.length*2}.
80. 1st 4 reserved memory slots: 1st 2 as scratch space for hashing, 3rd as free memory pointer, 4th as zero slot: used as initial value for dynamic memory arrays
81. free memory pointer => location: 0x40, init value: 0x80, gets updated as memory gets used
82. memory CAN'T BE FREED manually. So NEVER assume default 0 values for memory locations.
83. Assembly: lang Yul, value types can directly be used as local vars, local vars that refer to memory/calldata evaluate to variable address NOT value hence effectively reference, storage vars defined by *.slot (location) and *.offset (position in that slot)
84. sol 0.6, 0.7, 0.8 breaking changes in Solidity 201 (Block 2): 28:00
85. mapping cannot be defined inside struct/array in memory (but allowed in storage) since Solidity 0.7
86. OZ's ERC777's hooks are 1 way to avoid approve/transferFrom
87. OZ's CREATE2 lib allows deploy(uint ethAmount, bytes32 salt, bytes bytecode) and computeAddress(bytes32 salt, bytecodehash) to deterministic contract addresses
88. OZ's Multicall lib to batch multiple txns into 1
89. OZ's String lib: toString(uint value): uint -> ASCII String Decimal, similarly for toHexSrting || UNCLEAR (probably 12 → hex’xyz’)
90. OZ's MerkleProof: to check if a leaf is part of a merkle tree or not
91. OZ's SignatureChecker lib allows creating signs for EOA (ECDSA) and for contracts (ERC-1271) for eg for smart wallets
92. OZ's EIP-712 lib allows signing and hashing of type structured data, instead of just binary blobs. Check if devs have specified chainID, contract address to prevent replay attacks
93. OZ's Escrow and ConditionalEscrow: conditional escrow provides condition on withdrawal, RefundEscrow: allows refund to multiple depositors, on top of ConditionalEscrow
94. OZ's ERC-165: to check if a contract supports a particular interface
95. OZ's SafeCast: for safe downcasting of types. For eg: uin256 -> uint128/uint64, simmy for int256
96. EnumerableMap: only exists for mapping (uint256 => address). Enumerable in O(n)
97. EnumberableSet: supported types: bytes/address/uint256. Enumerable in O(n)
98. BitMap: maps uint256 -> bool, wherein each bit in this uint256 var represents a bool
99. PaymentSplitter, TimelockController: for adding timelocks to a contract
100. Context: support for meta-txns, ERC2771Context: tx signer -> gas relay -> trusted forwarder (verifies txn) -> destination contract
101. MinimalForwarder: implements trusted forwarder in 100
102. Proxy: to implement proxy in proxy<>implementation contract pattern
103. ERC1967Proxy: 102, but upgradabale. Allows changing impl contract
104. TransparentUpgradableProxy: has Admin. All admin calls restricted to proxy contract. All non-admin calls delegated to impl contract. Hence solving selector clash problem between proxy and impl i.e. fn selector of proxy = impl, hence confused which one to call
105. ProxyAdmin: admin contract of 104
106. BeaconProxy: 102, but impl address coming from UpgradeableBeacon contract and it is stored at ERC1967 specified slot
107. UpgradeableBeacon: support BeaconProxy to point to impl contract
108. Clones: impl ERC1167 ie Minimal Proxy contracts: where all impl contracts are clones of a specific bytecode, and all calls are delegated to known fixed address
109. Initializable: impl contracts NEED to impl initialize() to initialise the state of proxy contract RIGHT after impl contract is created (in context of proxy’s state || also in context of its own state), and this initialize() must only be callable ONCE. So Initializable lib helps impl contracts in implementing this. impl contract's constructor cannot be used for this, as that can only change state of impl contract, not of proxy's
110. While both of these share the same interface for upgrades, in UUPS (**Universal Upgradeable Proxy Standard**) proxies the upgrade is handled by the implementation, and can eventually be removed. Transparent proxies, on the other hand, include the upgrade and admin logic in the proxy itself. This means `[TransparentUpgradeableProxy` 28](https://docs.openzeppelin.com/contracts/4.x/api/proxy#TransparentUpgradeableProxy)
 is more expensive to deploy than what is possible with UUPS proxies.
111. Dappsys' DSProxy: 102, but also allows CREATION of impl contract along with making calls to it
112. DSMath: SafeMath + fixed-point math + wad (18 decimals), ray (27 decimals) support (wmul, wdiv, rmul, rdiv, rpow)
113. DSAuth: enables authorisation by DSGuard: has access control list (ACL) of [srcAddr][fnSign][destAddr] => boolean
114. DSRoles: role based ACL, Root Users, Public Capabilities, Role Capabilities -> non-root, non-public capabs