# Classic
 
## Description:
The classics never fade.

## Source code:
```python
import os
import string
import secrets

flag = os.getenv("FLAG") or "TSC{test_flag}"

charset = string.digits + string.ascii_letters + string.punctuation
A, B = secrets.randbelow(2**32), secrets.randbelow(2**32)
assert len(set((A * x + B) % len(charset) for x in range(len(charset)))) == len(charset)

enc = "".join(charset[(charset.find(c) * A + B) % len(charset)] for c in flag)
```

## Encrypted flag:
``o`15~UN;;U~;F~U0OkW;FNW;F]WNlUGV``

## Challenge:
This program uses the `Affine Cipher` to encrypt the flag.
The character set is made from all letters, numbers and punctuation, giving a length of `94` characters.
The encoding uses two values, A and B, which are randomly chosen using `secrets.randbelow(2**32)`.

- A: Multiplier for the character index.
- B: Addition constant for the character index.

These values are used in the encoding formula:
`charset[(charset.find(c) * A + B) % len(charset)]`
The encoding is an affine transformation, where each character c is transformed to a new character based on its index in the charset using the formula:
`new_index = (old_index * A + B) % len(charset)`
This is essentially a linear transformation of the indices of the characters.

## Solution:

To reverse the encoding, we need to solve the inverse of this affine transformation. Given that the transformation is:
`new_index = (old_index * A + B) % len(charset)`
This can be rearranged to `old_index = (new_index - B) * A_inv % len(charset)`
The extended euclidean algorithm can be used to calculate the inverse of A, and I asked chatgpt to write me a function to calculate this:
```python
def mod_inverse(a, m):
    # Extended Euclidean algorithm
    m0, x0, x1 = m, 0, 1
    while a > 1:
        q = a // m
        m, a = a % m, m
        x0, x1 = x1 - q * x0, x0
    if x1 < 0:
        x1 += m0
    return x1
```
Finally I brute forced the values of A and B between `0` and `93` as they are modulo with 94 in the encoding process to find the correct combination with this solve script
```python
import string

# Charset and constants from the encoding process
charset = string.digits + string.ascii_letters + string.punctuation
charset_len = len(charset)
enc = "o`15~UN;;U~;F~U0OkW;FNW;F]WNlUGV"

def mod_inverse(a, m):
    # Extended Euclidean algorithm
    m0, x0, x1 = m, 0, 1
    while a > 1:
        q = a // m
        m, a = a % m, m
        x0, x1 = x1 - q * x0, x0
    if x1 < 0:
        x1 += m0
    return x1

for A in range(0, 93):
    for B in range(0, 93):
        # Calculate A_inv (modular inverse of A modulo len(charset))
        try:
            A_inv = mod_inverse(A, charset_len)
        except:
            continue

        # Decode the encoded string
        decoded_flag = []
        for c in enc:
            # Find the encoded character index
            enc_index = charset.find(c)
            if enc_index == -1:
                raise ValueError(f"Character {c} not found in charset")

            # Reverse the encoding formula
            original_index = (enc_index - B) * A_inv % charset_len
            decoded_flag.append(charset[original_index])

        # Join decoded characters to form the decoded flag
        decoded_flag = ''.join(decoded_flag)
        print("Decoded flag:", decoded_flag, A, B)
```
Running this script and then grepping for `TSC` finds us the decoded flag which occured when `A=29` and `B=27`
