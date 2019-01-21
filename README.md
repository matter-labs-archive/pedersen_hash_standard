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

TODO