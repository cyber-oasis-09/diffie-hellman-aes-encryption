# Diffie-Hellman Key Agreement & AES-256 Encryption — Big-Integer Key Derivation in C#

A C#/.NET implementation of the receiving side of a Diffie-Hellman key exchange over a 256-bit modulus, followed by AES-256 encryption and decryption using the derived shared secret as the key.

> **Note:** This repo showcases the methodology, debugging process, and verified results from a graduate coursework project. The complete working implementation and the official assignment materials are withheld here in line with ASU's policy on not publicly posting course materials — full code available on request.

## The Problem

Given a set of command-line arguments, build a program that acts as one party of a Diffie-Hellman exchange and uses the resulting key for symmetric encryption:

- **Key agreement:** the modulus `N` is supplied as an exponent/constant pair (`N = 2^e − c`), along with the party's private exponent `x` and the other party's public value `g^y mod N`. No network exchange takes place; everything needed arrives up front, so the whole protocol can be computed as a single party.
- **Encryption:** derive a 256-bit AES key from the shared secret, decrypt a supplied ciphertext, and encrypt a supplied plaintext, both using a supplied 128-bit IV passed in hex.
- **Output:** the decrypted text and the encrypted bytes, printed as a comma-separated pair.

The values involved (~256 bits) are far beyond what `int`, `long`, or `decimal` can hold, so the entire computation has to run on arbitrary-precision arithmetic.

## Why This Approach

- **Diffie-Hellman** lets two parties arrive at the same secret over an open channel because `(g^y)^x ≡ (g^x)^y (mod N)`, while recovering `x` from `g^x mod N` requires solving the discrete logarithm problem.
- **Modular exponentiation, not exponentiation-then-reduce.** Computing `(g^y)^x` outright would produce a number with hundreds of millions of digits. `BigInteger.ModPow` uses square-and-multiply and reduces at every step, so intermediate values never exceed the size of the modulus.
- **A shared secret is a number, not a key.** AES-256 needs exactly 32 bytes, so the integer has to be converted to a fixed-length byte string. How that conversion is done (byte order, sign handling, length) determines whether the output matches at all.
- **Compact modulus encoding.** Expressing `N` as `2^e − c` keeps the arguments short and readable and makes it easy to sanity-check a value by hand while debugging.

## Approach (High Level)

1. Parse the IV from a space-separated hex string into bytes.
2. Rebuild the modulus with `BigInteger`: `N = 2^N_e − N_c`.
3. Compute the shared secret with `BigInteger.ModPow(gy, x, N)`.
4. Convert the secret to a 32-byte AES key (unsigned, little-endian, normalized to exactly 32 bytes).
5. Decrypt the given ciphertext with AES-256-CBC (PKCS7 padding).
6. Encrypt the given plaintext with the same key and IV.
7. Print `decryptedText,ENCRYPTED HEX BYTES`.

Not every input is needed: `g^y mod N` is already provided, so the `g_e` / `g_c` pair exists purely as a debugging aid for checking `g` by hand.

## The Debugging Story

The result that matters isn't just "it worked" — it's that each ambiguity in the specification was resolved by testing against the worked example instead of guessing.

- **Key byte order.** The specification never says how to turn the shared secret into 32 bytes. A big-endian conversion made decryption fail with a padding error; a little-endian conversion (the default byte order of `BigInteger.ToByteArray()`) reproduced the specification's expected output exactly. A padding failure on decrypt turned out to be the clearest signal that the *key* was wrong rather than the AES call.
- **Sign byte and key length.** `BigInteger.ToByteArray()` returns signed two's-complement bytes by default, so a secret with its high bit set can come back as 33 bytes with a trailing `0x00`, and a secret with leading zero bytes can come back shorter than 32. The final version requests an unsigned representation and normalizes the array to exactly 32 bytes, so the key length is correct for any secret rather than only for the sample.
- **Cross-checking before trusting the C#.** The derivation logic (modulus construction, modular exponentiation, byte order, CBC/PKCS7) was checked against the worked example in an independent reference implementation before being run in the course environment, which isolated the byte-order question from any C# API details.
- **Argument handling.** The IV and ciphertext arrive as space-separated hex strings, so they must be quoted on the command line or each byte becomes its own argument. Copying the long decimal values out of a document can also insert stray line breaks, so hex parsing strips all whitespace defensively. Running the program with no arguments at all throws an `IndexOutOfRangeException`, which is a reminder that input validation is a separate concern from the cryptography.
- **A stray exponent cast.** `BigInteger.Pow` takes an `int` exponent, so the modulus exponent (255) is cast to `int`. Only the exponent is narrowed; every large value stays a `BigInteger`, which keeps the "no `int`/`long`/`decimal` for big values" constraint intact.

## Verified Result

- **Worked example:** program output matched the specification's expected output exactly, byte-for-byte, for both the decrypted text and the encrypted bytes.
- **Autograder:** on the grader's hidden test values, the submission reported that both the encrypted and decrypted text were correct.
- **Final grade:** 100/100 (verified via the course autograder).

## Tools

- .NET Core 8.0 / C#
- `System.Numerics.BigInteger` (arbitrary-precision arithmetic, `ModPow`)
- `System.Security.Cryptography.Aes` (AES-256-CBC)
- zyBooks / zyLabs (submission and autograding)

## Future Directions

- Replace the raw shared-secret-as-key step with a proper key derivation function (for example HKDF over SHA-256), since using a Diffie-Hellman output directly as a key is a teaching simplification rather than a production practice.
- Move from unauthenticated AES-CBC to authenticated encryption (AES-GCM) to demonstrate why integrity matters alongside confidentiality.
- Add input validation for argument count, hex format, and public-value range (rejecting degenerate values such as 0, 1, or `N − 1`).
- Build a two-party version over a local socket so the exchange actually happens between two processes.
- Extend to elliptic-curve Diffie-Hellman (X25519) and compare key sizes and performance against the finite-field version.
- Pair it with a small-parameter discrete-log brute-force demo to show concretely why the modulus size matters.
