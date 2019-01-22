# Pedersen hash standardization effort

This desciption is based on the original `saping_crypto` implementation by Zcash.

## Building blocks

#### Group hash iteration

``` rust
pub fn group_hash<E: JubjubEngine>(
    tag: &[u8],
    personalization: &[u8],
    params: &E::Params
) -> Option<edwards::Point<E, PrimeOrder>>
```

Given the tag and personalization return point on the Twisted Edwards curve (usually called `JubJub` when working with zkSNARKs) with a prime order.

Algorithm for hashing into Twisted Edwards curve:

- Build a hasher with (Blake specific) no key and no IV ```let mut h = Blake2s::with_params(32, &[], &[], personalization);```
- Update hasher state with a constant ```h.update(constants::GH_FIRST_BLOCK);```
- Update a hasher state with a tag ```h.update(tag);```
- Finalize and get 32 bytes of result
- Interpret 32 bytes are Little Endian representation of some big integer
- Use top bit of this integer as a sign (parity) bit for X coordinate of the point on Twisted Edwards curve
- Use the rest of the integer as a field element of some prime field `Fr` of integers modulo `r`, where `r` is a size of the large prime group for `BLS12` curve or the main and only group on `BN` curve
- If this integer is out of the field - return None
- Otherwise use this integer (it's in the field) as an `y` coordinate of the point on the Twisted Edwards curve
- Find a corresponding `x` coordinate using the sign (parity) bit. If point is not on the curve
- Multiply this point by cofactor. If result in zero - return None
- Return multiplication result

There is no any kind of iterative process (`nonce`) in this procedure, it's moved to the higher level.

### Group hash

``` rust
fn find_group_hash<E: JubjubEngine>(
            m: &[u8],
            personalization: &[u8; 8],
            params: &E::Params
        ) -> edwards::Point<E, PrimeOrder>
```

Personalizetion is 8 bytes (hardcoded)

This process is iterative

- Take message `m` as a vector
- Make `nonce` in a form of unsigned integer of 8 bits starting with zero
- Append this nonce to the `m` and run the `group_hash` iteration from above using such concatenated message as a `tag`
- If `group_hash` returs a point - use this point
- Else increment `nonce`, form new `tag`
- Nonce can not repeat (obviously)

There is a fine point in this procedure:

- Hash output result is 256 bits
- One bit can be 0 or 1 and is just a sign (parity) bit
- Rest of the bits MUST form a representation of the field element in `Fp`. For `BLS12-381` used in Zcash a bitsize of prime making `Fp` is 255 bits, so MOST of the hashes will be in the field.
- For `BN256` curve used in Ethereum a bitsize of prime making `Fp` is 254 bits, so hashing will require twice the number of iterations to hit the result the is in `Fp`. Remember, there is no modular redution when byte array is interpreted as an integer inside the field!
- Range of nonces is 0-255, so in principle we'd have enough tries to hit the point in case of `BN256`, but for other curves this procedure may require a larger set of nonces and we may try to standardize it too in this document. E. g. make nonce an unsigned integer of 32 bits.

### Make Pedersen hash generators

This is actually the most trivial procedure of all

``` rust

for m in 0..5 {
    use byteorder::{WriteBytesExt, LittleEndian};

    let mut segment_number = [0u8; 4];
    (&mut segment_number[0..4]).write_u32::<LittleEndian>(m).unwrap();

    pedersen_hash_generators.push(
        find_group_hash(
            &segment_number,
            constants::PEDERSEN_HASH_GENERATORS_PERSONALIZATION,
            &tmp_params
        )
    );
}
```

If we neglect a language-specific parts the algorithm is the following:

- For `i` from 0 to 5 (non-inclusive)
- write `i` into byte array of length 4 as Little Endian
- run the `find_group_hash` routine from above using this 4 byte array as `m` parameter and supplying some constant for personalization

Two additional checks are required:

- Each generator is not a point of infinity
- Generators are distinct

## Test vectors

### Blake2s

```
    Creating generators using Blake2s
    Using personalization `Zcash_PH`
    Pedersen hash generators:
    Generator 0
    X = Fr(0x184570ed4909a81b2793320a26e8f956be129e4eed381acf901718dff8802135)
    Y = Fr(0x1c3a9a830f61587101ef8cbbebf55063c1c6480e7e5a7441eac7f626d8f69a45)
    Generator 1
    X = Fr(0x0afc00ffa0065f5479f53575e86f6dcd0d88d7331eefd39df037eea2d6f031e4)
    Y = Fr(0x237a6734dd50e044b4f44027ee9e70fcd2e5724ded1d1c12b820a11afdc15c7a)
    Generator 2
    X = Fr(0x00fb62ad05ee0e615f935c5a83a870f389a5ea2baccf22ad731a4929e7a75b37)
    Y = Fr(0x00bc8b1c9d376ceeea2cf66a91b7e2ad20ab8cce38575ac13dbefe2be548f702)
    Generator 3
    X = Fr(0x0675544aa0a708b0c584833fdedda8d89be14c516e0a7ef3042f378cb01f6e48)
    Y = Fr(0x169025a530508ee4f1d34b73b4d32e008b97da2147f15af3c53f405cf44f89d4)
    Generator 4
    X = Fr(0x07350a0660a05014168047155c0a0647ea2720ecb182a6cb137b29f8a5cfd37f)
    Y = Fr(0x3004ad73b7abe27f17ec04b04b450955a4189dd012b4cf4b174af15bd412696a)
```

### Keccak256

```
    Creating generators using Keccak256 (Ethereum style)
    Using personalization `Zcash_PH`
    Pedersen hash generators:
    Generator 0
    X = Fr(0x2dc8e1a19b312f1364389c41f05b47050d79dd94f89556cd6a299af2a18af827)
    Y = Fr(0x26ef1fdd7067acbadbee2c9e45b62ad0870266c33b77906ba3a0e579f052d962)
    Generator 1
    X = Fr(0x107014e1e603dbc9aa6ed137f9db7c1dfbfa6c4cac209e87973040083014d052)
    Y = Fr(0x291d41fb35848f18fbff70fbaf3872c83ffcddef3073bc561c75f44124656a1a)
    Generator 2
    X = Fr(0x06429b1fe714ccddc1c045dc77d910039270017a978b7b2581e66978d6c1f88b)
    Y = Fr(0x1dec8851eea374983ea82e4f7f45d8e6f4aecfd5f798a595309c16ffc0bf811b)
    Generator 3
    X = Fr(0x1d92fe609ef15389f37cdf014c204d0777d5824b855239ae3418fb4765d0ca48)
    Y = Fr(0x257e53e5da0507796de529ce91ec375d732337a4e9ceffcfe766efbc3226768a)
    Generator 4
    X = Fr(0x18ab8de17af7c13dbb6db75d36919fe9a453ab63b03cc840835d0bea4947d3c1)
    Y = Fr(0x193fad81cdc051494f93f7980d1c91245865618e69b61705f23353f88384c307)
```