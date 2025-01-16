# 2DES
 
## Description:
As you know, there are DES and 3DES in reality. However, I don't like the existing composition. Hence, I invented 2DES, which is a combination of 2 DESs.
`nc 172.31.2.2 9487`
Author: ywc

## Source code:
```python
#!/usr/bin/env python
from Crypto.Cipher import DES
from Crypto.Util.Padding import pad
from random import choice
from os import urandom
from time import sleep

def encrypt(msg: bytes, key1, key2):
    des1 = DES.new(key1, DES.MODE_ECB)
    des2 = DES.new(key2, DES.MODE_ECB)
    return des2.encrypt(des1.encrypt(pad(msg, des1.block_size)))

def main():
    flag = open('flag.txt', 'r').read().strip().encode()

    print("This is a 2DES encryption service.")
    print("But you can only control one of the key.")
    print()

    while True:
        print("1. Encrypt flag")
        print("2. Decrypt flag")
        print("3. Exit")
        option = int(input("> "))

        if option == 1:
            # I choose a key
            # You can choose another one
            keyset = ["1FE01FE00EF10EF1", "01E001E001F101F1", "1FFE1FFE0EFE0EFE"]
            key1 = bytes.fromhex(choice(keyset))
            key2 = bytes.fromhex(input("Enter key2 (hex): ").strip())

            ciphertext = encrypt(flag, key1, key2)
            print("Here is your encrypted flag:", flush=True)
            print("...", flush=True)
            sleep(3)
            if ciphertext[:4] == flag[:4]:
                print(ciphertext)
                print("Hmmm... What a coincidence!")
            else:
                print("System error!")
            print()

        elif option == 2:
            print("Decryption are disabled")
            print()

        elif option == 3:
            print("Bye!")
            exit()

        else:
            print("Invalid option")
            print()

if __name__ == "__main__":
    main()
```
## Challenge:
The source code shows an implementation of "`2DES`" which uses 2 keys and two seperate encryptions, similar to 2-key `3DES` but with one less encryption. The program chooses one of 3 keys from a list for the first DES, then the user can choose another key for the second, and if the first four letters of the encrypted text match the flag then the program will print out your ciphertext, which will hopefully lead you to the flag if you manage to fully match the ciphertext to the flag.

## Solution:
To solve this challenge you need to create a key value that will fully reverse the encoding done by whichever key is chosen by the program. As we know the first 4 letters of the flag will be the flag format, I created a file containing     `TSC{dummytestdata}` to test the encryption and hopefully come to a solution. As somewhat of a crypto beginner I initally had no idea where to even begin as I thought that `DES` was a relatively secure algorithm, so I began to research into 2-key `3DES` and stumbled across this blog post about weak keys. `https://stackoverflow.com/questions/37542102/decrypting-tripledes-specified-key-is-a-known-weak-key-and-cannot-be-used`

It seemed that `3DES `has certain pairs of keys that do not interact well with each other, usually very basic keys, but these would severely weaken the encryption process making it at worst about as secure as single `DES`. However for this implementation of "`2DES`" the keys could completely undo each other revealing the plaintext! 

I tested some varietys of classic weak keys such as all 0s and 1s and combinations until I came across a list weak keys for `DES` online. There are 6 pairs in `DES` known as "Semiweak Key Pairs", consisting of:
```
01FE 01FE 01FE 01FE and FE01 FE01 FE01 FE01
1FE0 1FE0 0EF1 0EF1 and E01F E01F F10E F10E
01E0 01E0 01F1 01F1 and E001 E001 F101 F101
1FFE 1FFE 0EFE 0EFE and FE1F FE1F FE0E FE0E
011F 011F 010E 010E and 1F01 1F01 0E01 0E01
E0FE E0FE F1FE F1FE and FEE0 FEE0 FEF1 FEF1
```
And these turn out to be some of the keys in our the keyset list.
I then asked chatgpt to create me a script to test these keypairs until we had a match with the test flag I created:
```python
from Crypto.Cipher import DES
from Crypto.Util.Padding import pad
import itertools

keyset = ["1FE01FE00EF10EF1", "01E001E001F101F1", "1FFE1FFE0EFE0EFE"]

semiweak_keys = [
    bytes.fromhex("01FE01FE01FE01FE"),  # First semiweak key pair part 1
    bytes.fromhex("FE01FE01FE01FE01"),  # First semiweak key pair part 2
    bytes.fromhex("1FE01FE00EF10EF1"),  # Second semiweak key pair part 1
    bytes.fromhex("E01FE01FF10EF10E"),  # Second semiweak key pair part 2
    bytes.fromhex("01E001E001F101F1"),  # Third semiweak key pair part 1
    bytes.fromhex("E001E001F101F101"),  # Third semiweak key pair part 2
    bytes.fromhex("1FFE1FFE0EFE0EFE"),  # Fourth semiweak key pair part 1
    bytes.fromhex("FE1FFE1FFE0EFE0E"),  # Fourth semiweak key pair part 2
    bytes.fromhex("011F011F010E010E"),  # Fifth semiweak key pair part 1
    bytes.fromhex("1F011F010E010E01"),  # Fifth semiweak key pair part 2
    bytes.fromhex("E0FEE0FEF1FEE0FE"),  # Sixth semiweak key pair part 1
    bytes.fromhex("FEE0FEE0FEF1FEF1"),  # Sixth semiweak key pair part 2
]

flag_prefix = b"TSC{"  # Start of the flag
test_data = flag_prefix + b"dummyflagdata"  # Dummy flag for testing

def encrypt_2des(msg, key1, key2):
    des1 = DES.new(key1, DES.MODE_ECB)
    des2 = DES.new(key2, DES.MODE_ECB)
    return des2.encrypt(des1.encrypt(pad(msg, des1.block_size)))

def find_matching_key_for_ciphertext():
    for key1_hex in keyset:
        key1 = bytes.fromhex(key1_hex)
        print(f"Testing with key1: {key1.hex().upper()}")
        
        for weak_key2 in semiweak_keys:
            print(f"  Using semiweak key2: {weak_key2.hex().upper()}")
            
            ciphertext = encrypt_2des(test_data, key1, weak_key2)
            print(f"Ciphertext: {ciphertext.hex().upper()}")
            
            # Check if the ciphertext starts with the expected "TSC{"
            if ciphertext[:4] == flag_prefix:
                print("\n=== MATCH FOUND ===")
                print(f"Ciphertext (Match found!): {ciphertext.hex().upper()}")
                print(f"  key1: {key1.hex().upper()}")
                print(f"  key2: {weak_key2.hex().upper()}")
                print("====================\n")
                return key1, weak_key2, ciphertext
            else:
                print("No match.")

    print("\nNo match found after testing all keys.")
    return None

find_matching_key_for_ciphertext()
```
And running this script gave us a hit!
```
┌──(kali㉿kali)-[~/Downloads/share]
└─$ python ex.py
Testing with key1: 1FE01FE00EF10EF1
  Using semiweak key2: 01FE01FE01FE01FE
Ciphertext: F1482A7EF14707B31D88F51BD7D9143563782A7263D15B4B
No match.
  Using semiweak key2: FE01FE01FE01FE01
Ciphertext: DE9E3983F82D58C6DEE698E35C26A9EFBCAB4EBF7503D178
No match.
  Using semiweak key2: 1FE01FE00EF10EF1
Ciphertext: 1F52D1A697E44C62634C0358AA6077FABF7AFEA5042DE281
No match.
  Using semiweak key2: E01FE01FF10EF10E
Ciphertext: 5453437B64756D6D79666C61676461746107070707070707

=== MATCH FOUND ===
Ciphertext (Match found!): 5453437B64756D6D79666C61676461746107070707070707
  key1: 1FE01FE00EF10EF1
  key2: E01FE01FF10EF10E
====================
```
Decoding the hex printed back to us our full flag of `TSC{dummytestdata}` so we had a full working key pair that reverse each other, I then sent this key to the server, and kept repeating until the random function chose the correct `1FE01FE00EF10EF1` from the 2 other possible keys, and we got the flag.
```bash
┌──(kali㉿kali)-[~/Downloads/share]
└─$ nc 172.31.2.2 9487
This is a 2DES encryption service.
But you can only control one of the key.

1. Encrypt flag
2. Decrypt flag
3. Exit
> 1
Enter key2 (hex): E01FE01FF10EF10E
Here is your encrypted flag:
...
b'TSC{th3_Key_t0_br34k_DES_15_tHe_keY}\x04\x04\x04\x04'
Hmmm... What a coincidence!
```
