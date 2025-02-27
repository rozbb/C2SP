# ristretto255 VRF

[c2sp.org/vrf-r255](https://c2sp.org/vrf-r255)

This document defines the ECVRF-RISTRETTO255-SHA512 ciphersuite of the ECVRF
construction specified in [draft-irtf-cfrg-vrf-11][], similarly to
[draft-irtf-cfrg-vrf-11, Section 5.5][] and according to
[draft-irtf-cfrg-vrf-11, Section 7.10][].

## Conventions used in this document

`||` denotes concatenation. `0x` followed by two hexadecimal characters denotes
a byte value in the 0-255 range.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [BCP 14][] [RFC 2119][]
[RFC 8174][] when, and only when, they appear in all capitals, as shown here.

## ECVRF-RISTRETTO255-SHA512

* `suite_string` = `0xFF || "c2sp.org/vrf-r255"`

* The EC group G is the ristretto255 group defined in
  [draft-irtf-cfrg-ristretto255-decaf448-03][].

    * For this group, `ptLen` = `qLen` = 32, `cofactor` = 1, `B` is the canonical
    generator defined in [draft-irtf-cfrg-ristretto255-decaf448-03, Section 4][],
    and `q` = 2^252 + 27742317777372353535851937790883648493.

    * Since ristretto255 is an abstract prime order group, F (the base field)
    and E (the elliptic curve) are not defined. Instead of points on E which
    might not be on the prime-order subgroup, this ciphersuite simply deals
    directly with elements in G.

* `cLen` = 16

* The secret key `SK` is equal to the secret scalar `x` and is generated by
  selecting a random element of the scalar field, for example by obtaining 64
  bytes from a CSPRNG and reducing them modulo `q`, as suggested in
  [draft-irtf-cfrg-ristretto255-decaf448-03, Section 4.4][].

* The `ECVRF_nonce_generation` function is as specified in
  [#nonce-generation](#nonce-generation).

* The `int_to_string` and `string_to_int` functions respectively encode the
  integer as a string and interpret the string as an integer in little-endian
  representation.

    * In practice, most implementations neither will nor should deal with
      arbitrary integer types. Instead, every time this document or
      draft-irtf-cfrg-vrf-11 use `int_to_string` and `string_to_int` or perform
      arithmetic on integers, they're operating on elements of the scalar field
      (integers modulo `q`).

    * ristretto255 libraries commonly provide constant-time implementations of
      two kind of setters for scalar field elements: one that reduces the input
      modulo `q`, suitable for `k` in `ECVRF_nonce_generation`; and one that
      checks that the input is below `q`, suitable for `s` in
      `ECVRF_decode_proof`. `c` is always smaller than `q` due to its length, so
      either setter can be used. Right-padding with zeroes or truncation might be
      necessary to accommodate the various lengths.

* The `point_to_string` function is the ristretto255 encoding group function
  defined in [draft-irtf-cfrg-ristretto255-decaf448-03, Section 4.3.2][].

* The `string_to_point` function is the ristretto255 decoding group function
  defined in [draft-irtf-cfrg-ristretto255-decaf448-03, Section 4.3.1][].

* The hash function `Hash` is SHA-512 as specified in [RFC 6234][], with `hLen` = 64.

* The `ECVRF_encode_to_curve` function is as specified in
  [#encode-to-curve](#encode-to-curve).

    * `encode_to_curve_salt` = `PK_string`

It is RECOMMENDED for the `validate_key` parameter to `ECVRF_verify` to be always
TRUE. For this ciphersuite, the `ECVRF_validate_key` operation simply involves
checking that the public key is not the identity element.

### Nonce generation

The ECVRF-EDWARDS25519-SHA512 ciphersuites adopt the [RFC 8032][] key and
nonce generation procedures, which [are hard to prove formally][] and [require a
more convoluted argument for hash domain separation][]. Instead, here we use a
straightforwardly domain-separated `Hash` invocation to derive the nonce from the
inputs, similarly to how the ECVRF-P256-SHA256 ciphersuites work, but without
the dependency on HMAC-DRBG. This also enables `SK` to be equal to `x`, and along
with the simplified [#encode-to-curve](#encode-to-curve) procedure ensures that
every `Hash` instantiation in ECVRF-RISTRETTO255-SHA512 is domain-separated by a
fixed prefix.

`ECVRF_nonce_generation(SK, h_string)`

Input:

  * `SK` — ECVRF secret key, an integer between 0 and q-1

  * `h_string` — an octet string

Output:

  * `k` — nonce, an integer between 0 and q-1

Steps:

  1. `nonce_generation_domain_separator` = `0x81`
  2. `k_string` = `Hash(suite_string || nonce_generation_domain_separator
            || int_to_string(SK, 32) || h_string)`
  3. `k` = `string_to_int(k_string) mod q`

### Encode to curve

The [PWHVNRG17] proof models `ECVRF_encode_to_curve` as a random oracle separate
from any other `Hash` instantiation. [draft-irtf-cfrg-vrf-11, Section 7.8][]
argues that it's ok to use [irtf-cfrg-hash-to-curve][] for `ECVRF_encode_to_curve`
because it will always invoke `Hash` with a non-zero final input byte, while most
other uses of `Hash` in the document append a zero byte to the input. Since
ristretto255 provides a one-way map and `hLen` is conveniently the same size as
its input, in this document we get to skip all that complexity and use an
instantiation of `Hash` that's domain-separated like every other hash use in
draft-irtf-cfrg-vrf-11.

`ECVRF_encode_to_curve(encode_to_curve_salt, alpha_string)`

Input:

  * `encode_to_curve_salt` — public salt value, an octet string

  * `alpha_string` — value to be hashed, an octet string
  
Output:

  * `H` — hashed value, an element in G

Steps:

  1. `encode_to_curve_domain_separator` = `0x82`
  2. `hash_string` = `Hash(suite_string || encode_to_curve_domain_separator
            || encode_to_curve_salt || alpha_string)`
  3. `H` = `ristretto255_one_way_map(hash_string)`

`ristretto255_one_way_map` is the one-way map defined in
[draft-irtf-cfrg-ristretto255-decaf448-03, Section 4.3.4][].

## Test vector

In this section, octet strings are represented as hex, scalars are encoded with
`int_to_string`, and group elements are encoded with `point_to_string`. Newlines
are inserted for readability.

```
SK = x = 3431c2b03533e280b23232e280b34e2c3132c2b03238e280b23131e280b34500

PK = Y = 54136cd90d99fbd1d4e855d9556efea87ba0337f2a6ce22028d0f5726fcb854e

alpha = 633273702e6f72672f7672662d72323535

hash_string = 3907ed3453d308b0cb4ae071be7e5a80f7db05f11f5569016e3fa3996f730782
    1142133d0124fb3774d55ba6ccd14c11f71bf66038ec80b3f9973a1a6d69f5db

H = f245308737c2a888ba56448c8cdbce9d063b57b147e063ce36c580194ef31a63

k_string = b5eb28143d9defee6faa0c02ff0168b7ac80ea89fe9362845af15cabd100a91e
    d6251dfa52be36405576eca4a0970f91225b85c8813206d13bd8b42fd11a00fe

k = d32fcc5ae91ba05704da9df434f22fd4c2c373fdd8294bbb58bf27292aeec00a

Gamma = x*H = 0a97d961262fb549b4175c5117860f42ae44a123f93c476c439eddd1c0cff926

U = k*B = 9a30709d72de12d67f7af1cd8695ff16214d2d4600ae5f478873d2e7ed0ece73

V = k*H = 5e727d972b11f6490b0b1ba8147775bceb1a2cb523b381fa22d5a5c0e97d4744

c_string = 5c805525233e2284dbed45e593b8eea346184b1548e416a11c85f0091b7dba42
    c92eaea061d0f3378261fc360f5b3cf793020236a9aaec5bbff84c09c91d0555

c = 5c805525233e2284dbed45e593b8eea3

s = 1d5ca9734d72bcbba9738d5237f955f3b2422351149d1312503b6441a47c940c

pi = 0a97d961262fb549b4175c5117860f42ae44a123f93c476c439eddd1c0cff926
    5c805525233e2284dbed45e593b8eea31d5ca9734d72bcbba9738d5237f955f3
    b2422351149d1312503b6441a47c940c

beta = dd653f0879b48c3ef69e13551239bec4cbcc1c18fe8894de2e9e1c790e182736
    03bf1c6c25d7a797aeff3c43fd32b974d3fcbd4bcce916007097922a3ea3a794
```

[draft-irtf-cfrg-vrf-11]: https://www.ietf.org/archive/id/draft-irtf-cfrg-vrf-11.html
[draft-irtf-cfrg-vrf-11, Section 5.5]: https://www.ietf.org/archive/id/draft-irtf-cfrg-vrf-11.html#name-ecvrf-ciphersuites
[draft-irtf-cfrg-vrf-11, Section 7.10]: https://www.ietf.org/archive/id/draft-irtf-cfrg-vrf-11.html#name-futureproofing
[draft-irtf-cfrg-vrf-11, Section 7.8]: https://www.ietf.org/archive/id/draft-irtf-cfrg-vrf-11.html#name-hash-function-domain-separa
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[RFC 2119]: https://www.rfc-editor.org/info/rfc2119
[RFC 8174]: https://www.rfc-editor.org/info/rfc8174
[draft-irtf-cfrg-ristretto255-decaf448-03]: https://www.ietf.org/archive/id/draft-irtf-cfrg-ristretto255-decaf448-03.html
[draft-irtf-cfrg-ristretto255-decaf448-03, Section 4]: https://www.ietf.org/archive/id/draft-irtf-cfrg-ristretto255-decaf448-03.html#name-ristretto255
[draft-irtf-cfrg-ristretto255-decaf448-03, Section 4.4]: https://www.ietf.org/archive/id/draft-irtf-cfrg-ristretto255-decaf448-03.html#name-scalar-field
[draft-irtf-cfrg-ristretto255-decaf448-03, Section 4.3.2]: https://www.ietf.org/archive/id/draft-irtf-cfrg-ristretto255-decaf448-03.html#name-encode
[draft-irtf-cfrg-ristretto255-decaf448-03, Section 4.3.1]: https://www.ietf.org/archive/id/draft-irtf-cfrg-ristretto255-decaf448-03.html#name-decode
[draft-irtf-cfrg-ristretto255-decaf448-03, Section 4.3.4]: https://www.ietf.org/archive/id/draft-irtf-cfrg-ristretto255-decaf448-03.html#name-one-way-map
[RFC 6234]: https://www.rfc-editor.org/info/rfc6234
[RFC 8032]: https://www.rfc-editor.org/info/rfc8032
[are hard to prove formally]: https://eprint.iacr.org/2020/823
[require a more convoluted argument for hash domain separation]: https://www.ietf.org/archive/id/draft-irtf-cfrg-vrf-11.html#name-hash-function-domain-separa
[PWHVNRG17]: https://eprint.iacr.org/2017/099
[irtf-cfrg-hash-to-curve]: https://datatracker.ietf.org/doc/draft-irtf-cfrg-hash-to-curve/
