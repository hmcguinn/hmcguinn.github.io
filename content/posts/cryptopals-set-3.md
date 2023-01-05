---
title: "Cryptopals Set 3: Block & stream crypto"
date: 2022-12-28T10:15:11-05:00
draft: false
---

## 17. The CBC padding oracle
> This is the best-known attack on modern block-cipher cryptography.
> 
> Combine your padding code and your CBC code to write two functions.
> 
> The first function should select at random one of the following 10 strings:
> 
> ```
> MDAwMDAwTm93IHRoYXQgdGhlIHBhcnR5IGlzIGp1bXBpbmc=
> MDAwMDAxV2l0aCB0aGUgYmFzcyBraWNrZWQgaW4gYW5kIHRoZSBWZWdhJ3MgYXJlIHB1bXBpbic=
> MDAwMDAyUXVpY2sgdG8gdGhlIHBvaW50LCB0byB0aGUgcG9pbnQsIG5vIGZha2luZw==
> MDAwMDAzQ29va2luZyBNQydzIGxpa2UgYSBwb3VuZCBvZiBiYWNvbg==
> MDAwMDA0QnVybmluZyAnZW0sIGlmIHlvdSBhaW4ndCBxdWljayBhbmQgbmltYmxl
> MDAwMDA1SSBnbyBjcmF6eSB3aGVuIEkgaGVhciBhIGN5bWJhbA==
> MDAwMDA2QW5kIGEgaGlnaCBoYXQgd2l0aCBhIHNvdXBlZCB1cCB0ZW1wbw==
> MDAwMDA3SSdtIG9uIGEgcm9sbCwgaXQncyB0aW1lIHRvIGdvIHNvbG8=
> MDAwMDA4b2xsaW4nIGluIG15IGZpdmUgcG9pbnQgb2g=
> MDAwMDA5aXRoIG15IHJhZy10b3AgZG93biBzbyBteSBoYWlyIGNhbiBibG93
> ```
> 
> ... generate a random AES key (which it should save for all future encryptions), pad the string out to the 16-byte AES block size and CBC-encrypt it under that key, providing the caller the ciphertext and IV.
> 
> The second function should consume the ciphertext produced by the first function, decrypt it, check its padding, and return true or false depending on whether the padding is valid.
> 
> This pair of functions approximates AES-CBC encryption as its deployed serverside in web applications; the second function models the server's consumption of an encrypted session token, as if it was a cookie.
> 
> It turns out that it's possible to decrypt the ciphertexts provided by the first function.
> 
> The decryption here depends on a side-channel leak by the decryption function. The leak is the error message that the padding is valid or not.
> 
> You can find 100 web pages on how this attack works, so I won't re-explain it. What I'll say is this:
> 
> The fundamental insight behind this attack is that the byte 01h is valid padding, and occur in 1/256 trials of "randomized" plaintexts produced by decrypting a tampered ciphertext.
> 
> 02h in isolation is *not* valid padding.
> 
> 02h 02h *is* valid padding, but is much less likely to occur randomly than 01h.
> 
> 03h 03h 03h is even less likely.
> 
> So you can assume that if you corrupt a decryption AND it had valid padding, you know what that padding byte is.
> 
> It is easy to get tripped up on the fact that CBC plaintexts are "padded". **Padding oracles have nothing to do with the actual padding on a CBC plaintext**. It's an attack that targets a specific bit of code that handles decryption. You can mount a padding oracle *on any CBC block*, whether it's padded or not. 

Understanding exactly how the CBC padding oracle works took a very long time. Getting my code to work took even longer. I even "knew" about this attack before! For this attack, we're not trying to recover the padding. It's different from the ECB byte-at-a-time oracle in that we're trying to learn the output of the block cipher decryption, not directly decoding the plaintext.

![CBC decryption](https://upload.wikimedia.org/wikipedia/commons/2/2a/CBC_decryption.svg)

To mount this attack, we'll rely on an oracle that decrypts a ciphertext and tells us whether it has valid padding or not:

```
def padding_oracle(ct, iv):
    pt = cbc_decrypt(ct, key, iv)

    try:
        pkcs_7.safe_unpad(pt)
        return True
    except ValueError:
        return False
```

To reconstruct the intermediate state of a block decryption, we need to XOR in the previous block.

Here's how we can mount a padding oracle attack to recover a block. Each iteration, we'll aim to recover the **i'th** last block (starting at 1). To do this, we need the padding to be valid. For example:

```
i=1      ...........\x01
i=2      ........\x02\x02  
i=3      ....\x03\x02\x02  
```

 strategy for the attack:

1. Start by creating our intermediate state array ``i_state = [0]*16``
2. Create a temporary array ``padding = [i]*16``
3. `` temp = padding ^ i_state``
4. Try all 255 values for ``temp[i] = guessed_byte`` and ask the oracle for a decryption of the generated ciphertext
    1. If the oracle returns that the padding is valid, store the intermediate state ``i_state[i] = [i ^ guessed_byte]``
    2. Else, keep trying bytes until we get valid padding. Once we've recovered the entire intermediate state (16 bytes), recover the plaintext with ``plaintext = ciphertext ^ i_state``

Let's think through how and why this works!

- One of the 255 possible bytes will give us ``\x01`` as padding when XOR'd with the unknown intermediate state.
  - We're trying to find ``\x01 = guessed_byte ^ i_state[-1]``
- Once we know the guessed byte that gives us valid padding, we can recover the intermediate state:
  - ``i_state[-1] = guessed_byte ^ \x01``
- This works to recover the last byte, but we'll have to adjust slightly to recover a full block. Instead of just guessing the last byte, we also need to modify the rest of the padding. For ``\x02\x02`` to be valid padding, the last two blocks need to be ``\x02``. We'll have to modify the bytes we know to achieve this.
- We now want to recover the second to last block of the intermediate state. Set the last byte = ``\x02 ^ i_state``. When it's decrypted, we'll have ``\x02 ^ i_state ^ i_state = \x02``. Valid padding!

Our padding oracle takes a ciphertext and an IV. We could do this block by block, modifying the previous block. Or, we can work each block individually, modifying the IV instead. 

Here's a function to recover a block's intermediate state:
```
def decode_block(block):
    i_state = [0]*16
    for i in range(1, 17):
        padding = [i]*16

        for guessed_byte in range(0, 256):
            padding[-i] = guessed_byte
            temp = cbc.xor(bytes(padding), bytes(i_state))

            valid = padding_oracle(block, temp)
            if valid:
                if i == 1:
                    padding[-2] = 0  # set to 0
                    mod_block = bytes(padding)
                    still_valid = padding_oracle(block, mod_block)
                    if not still_valid:
                        continue
                i_state[-i] = (guessed_byte ^ i)
                break
    return i_state
```

Using this function, we can recover each block individually. Then, we can recover the plaintext using the intermediate state. As we recover the intermediate state, we'll XOR it with the previous block to recover the plaintext! 

```
blocks = list(divide_chunks(ct, 16))
pt = bytearray()
for i in range(len(blocks)):
    temp_i_state = decode_block(blocks[i])
    pt += cbc.xor(bytes(temp_i_state), iv)
    iv = blocks[i]

print(pt)
```





## 18. Implement CTR, the stream cipher mode
> The string:
> 
> ```
> L77na/nrFsKvynd6HzOoG7GHTLXsTVu9qvY/2syLXzhPweyyMTJULu/6/kXX0KSvoOLSFQ==
> ```
> 
> ... decrypts to something approximating English in CTR mode, which is an AES block cipher mode that turns AES into a stream cipher, with the following parameters:
> 
> ```
>   key=YELLOW SUBMARINE
>     nonce=0
>     format=64 bit unsigned little endian nonce,
>     64 bit little endian block count (byte count / 16)
> ```
> 
> CTR mode is very simple.
> 
> Instead of encrypting the plaintext, CTR mode encrypts a running counter, producing a 16 byte block of keystream, which is XOR'd against the plaintext.
> 
> For instance, for the first 16 bytes of a message with these parameters:
> 
> ```
> keystream = AES("YELLOW SUBMARINE",
>                 "\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00")
> ```
> 
> ... for the next 16 bytes:
> 
> ```
> keystream = AES("YELLOW SUBMARINE",
>                 "\x00\x00\x00\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00")
> ```
> 
> ... and then:
> 
> ```
> keystream = AES("YELLOW SUBMARINE",
>                 "\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x00\x00\x00\x00\x00\x00")
> ```
> 
> CTR mode does not require padding; when you run out of plaintext, you just stop XOR'ing keystream and stop generating keystream.
> 
> Decryption is identical to encryption. Generate the same keystream, XOR, and recover the plaintext.
> 
> Decrypt the string at the top of this function, then use your CTR function to encrypt and decrypt other things. 

For our next challenge, we're implementing AES CTR mode. CTR mode is really nice to work with, and we'll only need one function for encryption/decryption. We can get an idea of how it works from [Wikipedia:](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_(CTR))

![CTR mode](https://upload.wikimedia.org/wikipedia/commons/4/4d/CTR_encryption_2.svg)

```
def ctr_mode(pt, key, nonce):
    output = bytearray()

    pt = list(divide_chunks(pt, 16))
    ctr = 0
    for block in pt:
        ctr_bytes = nonce + ctr.to_bytes(8, 'little')

        cipher = Cipher(algorithms.AES(key), modes.ECB())
        encryptor = cipher.encryptor()
        keystream = encryptor.update(ctr_bytes) + encryptor.finalize()

        xord = xor(keystream, block)
        output += xord

        ctr += 1
    return output
```



## 19. Break fixed-nonce CTR mode using substitutions

> Take your CTR encrypt/decrypt function and fix its nonce value to 0. Generate a random AES key.
> 
> In successive encryptions (not in one big running CTR stream), encrypt each line of the base64 decodes of the following, producing multiple independent ciphertexts:
> 
> ```
> SSBoYXZlIG1ldCB0aGVtIGF0IGNsb3NlIG9mIGRheQ==
> Q29taW5nIHdpdGggdml2aWQgZmFjZXM=
> RnJvbSBjb3VudGVyIG9yIGRlc2sgYW1vbmcgZ3JleQ==
> RWlnaHRlZW50aC1jZW50dXJ5IGhvdXNlcy4=
> SSBoYXZlIHBhc3NlZCB3aXRoIGEgbm9kIG9mIHRoZSBoZWFk
> T3IgcG9saXRlIG1lYW5pbmdsZXNzIHdvcmRzLA==
> T3IgaGF2ZSBsaW5nZXJlZCBhd2hpbGUgYW5kIHNhaWQ=
> UG9saXRlIG1lYW5pbmdsZXNzIHdvcmRzLA==
> QW5kIHRob3VnaHQgYmVmb3JlIEkgaGFkIGRvbmU=
> T2YgYSBtb2NraW5nIHRhbGUgb3IgYSBnaWJl
> VG8gcGxlYXNlIGEgY29tcGFuaW9u
> QXJvdW5kIHRoZSBmaXJlIGF0IHRoZSBjbHViLA==
> QmVpbmcgY2VydGFpbiB0aGF0IHRoZXkgYW5kIEk=
> QnV0IGxpdmVkIHdoZXJlIG1vdGxleSBpcyB3b3JuOg==
> QWxsIGNoYW5nZWQsIGNoYW5nZWQgdXR0ZXJseTo=
> QSB0ZXJyaWJsZSBiZWF1dHkgaXMgYm9ybi4=
> VGhhdCB3b21hbidzIGRheXMgd2VyZSBzcGVudA==
> SW4gaWdub3JhbnQgZ29vZCB3aWxsLA==
> SGVyIG5pZ2h0cyBpbiBhcmd1bWVudA==
> VW50aWwgaGVyIHZvaWNlIGdyZXcgc2hyaWxsLg==
> V2hhdCB2b2ljZSBtb3JlIHN3ZWV0IHRoYW4gaGVycw==
> V2hlbiB5b3VuZyBhbmQgYmVhdXRpZnVsLA==
> U2hlIHJvZGUgdG8gaGFycmllcnM/
> VGhpcyBtYW4gaGFkIGtlcHQgYSBzY2hvb2w=
> QW5kIHJvZGUgb3VyIHdpbmdlZCBob3JzZS4=
> VGhpcyBvdGhlciBoaXMgaGVscGVyIGFuZCBmcmllbmQ=
> V2FzIGNvbWluZyBpbnRvIGhpcyBmb3JjZTs=
> SGUgbWlnaHQgaGF2ZSB3b24gZmFtZSBpbiB0aGUgZW5kLA==
> U28gc2Vuc2l0aXZlIGhpcyBuYXR1cmUgc2VlbWVkLA==
> U28gZGFyaW5nIGFuZCBzd2VldCBoaXMgdGhvdWdodC4=
> VGhpcyBvdGhlciBtYW4gSSBoYWQgZHJlYW1lZA==
> QSBkcnVua2VuLCB2YWluLWdsb3Jpb3VzIGxvdXQu
> SGUgaGFkIGRvbmUgbW9zdCBiaXR0ZXIgd3Jvbmc=
> VG8gc29tZSB3aG8gYXJlIG5lYXIgbXkgaGVhcnQs
> WWV0IEkgbnVtYmVyIGhpbSBpbiB0aGUgc29uZzs=
> SGUsIHRvbywgaGFzIHJlc2lnbmVkIGhpcyBwYXJ0
> SW4gdGhlIGNhc3VhbCBjb21lZHk7
> SGUsIHRvbywgaGFzIGJlZW4gY2hhbmdlZCBpbiBoaXMgdHVybiw=
> VHJhbnNmb3JtZWQgdXR0ZXJseTo=
> QSB0ZXJyaWJsZSBiZWF1dHkgaXMgYm9ybi4=
> ```
> 
> (This should produce 40 short CTR-encrypted ciphertexts).
> 
> Because the CTR nonce wasn't randomized for each encryption, each ciphertext has been encrypted against the same keystream. This is very bad.
> 
> Understanding that, like most stream ciphers (including RC4, and obviously any block cipher run in CTR mode), the actual "encryption" of a byte of data boils down to a single XOR operation, it should be plain that:
> 
> ```
> CIPHERTEXT-BYTE XOR PLAINTEXT-BYTE = KEYSTREAM-BYTE
> ```
> 
> And since the keystream is the same for every ciphertext:
> 
> ```
> CIPHERTEXT-BYTE XOR KEYSTREAM-BYTE = PLAINTEXT-BYTE (ie, "you don't
> say!")
> ```
> 
> Attack this cryptosystem piecemeal: guess letters, use expected English language frequence to validate guesses, catch common English trigrams, and so on. 

I started to work on this one but decided to skip finishing it after realizing how tedious it is. The basic idea is to use common bigrams/trigrams to attack two ciphertexts XOR'd together. From there, cribbing can be used to decrypt the ciphertexts. There's a much easier way to do this though which we'll see in the next challenge!


## 20. Break fixed-nonce CTR statistically

> In this file find a similar set of Base64'd plaintext. Do with them exactly what you did with the first, but solve the problem differently.
> 
> Instead of making spot guesses at to known plaintext, treat the collection of ciphertexts the same way you would repeating-key XOR.
> 
> Obviously, CTR encryption appears different from repeated-key XOR, but with a fixed nonce they are effectively the same thing.
> 
> To exploit this: take your collection of ciphertexts and truncate them to a common length (the length of the smallest ciphertext will work).
> 
> Solve the resulting concatenation of ciphertexts as if for repeating- key XOR, with a key size of the length of the ciphertext you XOR'd. 

CTR mode with a fixed-nonce is equivalent to repeated-key XOR! Each plaintext is XOR'd against the same keystream, so we can break it in the same way we broke the ciphertexts in [challenge 6](https://cryptopals.com/sets/1/challenges/6).

We start by combining the *i'th* bytes of each ciphertext:
```
max_len = max(cts, key=len)

blocks = [[] for i in range(max_len)]
for block in cts:
    for i in range(len(block)):
        blocks[i].append(block[i])
```

After we have each block, we'll break each one individually as single key XOR. After we've attacked each block, we can combine the results to give us the keystream. By XOR'ing this keystream with each ciphertext, we can then recover the plaintext!

```
guessed_key = []
for block in blocks:
    guessed_key.append(break_single_key_xor(block)[0]) # take only the guessed key
```

Since I wrote my ``break_single_key_xor()`` function to return the key as a string, we'll combine each recovered key and convert it to bytes. Then, we can XOR it with each ciphertext to recover the plaintexts!
```
b_guessed_key = bytearray()
for i in guessed_key:
    b_guessed_key.append(ord(i))

for block in cts:
    recovered_pt = xor(b_guessed_key, block)
    print(guess_pt)
```

## 21. Implement the MT19937 Mersenne Twister RNG

> You can get the psuedocode for this from Wikipedia.
>
> If you're writing in Python, Ruby, or (gah) PHP, your language is probably already giving you MT19937 as "rand()"; don't use rand(). Write the RNG yourself. 

This challenge is straightforward using the [psuedocode from Wikipedia](https://en.wikipedia.org/wiki/Mersenne_Twister#Pseudocode):

```
# CONSTANTS:
w, n, m, r = 32, 624, 397, 31
a = 0x9908b0df
u, d, = 11, 0xFFFFFFFF
s, b = 7, 0x9d2c5680
t, c = 15, 0xefc60000
l = 18
mask_32 = 0xffffffff
f = 1812433253

class MT19937:
    def __init__(self, seed):
        self.mt = [0]*(n)
        self.index = n+1
        self.lower_mask = (1 << r) - 1
        self.upper_mask = mask_32 & ~self.lower_mask

        self.index = n
        self.mt[0] = seed
        for i in range(1, n):
            self.mt[i] = mask_32 & (f * (self.mt[i-1] ^ (self.mt[i-1] >> (w-2))) + i)

    def extract_num(self):
        if self.index >= n:
            if self.index > n:
                print("maybe seed by default")
                raise ValueError
            self.twist()

        self.index += 1

        y = self.mt[self.index]
        y1 = y ^ ((y >> u) & d)
        y2 = y1 ^ ((y1 << s) & b)
        y3 = y2 ^ ((y2 >> t) & c)
        y4 = (y3 ^ (y3 >> l) & mask_32)

        return y4

    def twist(self):
        for i in range(0, n):
            x = (self.mt[i] & self.upper_mask) + (self.mt[(i+1) % n] & self.lower_mask)

            xA = x >> 1
            if (x % 2) != 0:
                xA = xA ^ a
            self.mt[i] = self.mt[(i+m) % n] ^ xA
        self.index = 0
```

## 22. Crack an MT19937 seed
> Make sure your MT19937 accepts an integer seed value. Test it (verify that you're getting the same sequence of outputs given a seed).
> 
> Write a routine that performs the following operation:
> 
> - Wait a random number of seconds between, I don't know, 40 and 1000.
> - Seeds the RNG with the current Unix timestamp
> - Waits a random number of seconds again.
> - Returns the first 32 bit output of the RNG.
> 
> You get the idea. Go get coffee while it runs. Or just simulate the passage of time, although you're missing some of the fun of this exercise if you do that.
> 
> From the 32 bit RNG output, discover the seed. 

In this challenge, we're recovering a seed given the first 32 bits of output of a Mersenne Twister. 

The challenge asks us to seed the MT using the current unix timestamp + some offset in seconds. We then return the output:

```
def seed_and_return():
    unix = int(time.time())

    offset = random.randint(40, 1000)
    print(unix + offset)

    mt = MT19937(unix + offset)
    return mt.extract_num()
```

Once we're given this output we want to figure out what the seed is (the unix timestamp + offset). To do this, we try a range around the current timestamp checking if the output is equal. Once it is, we know we've recovered the seed:

```
def crack_seed(output):
    unix = int(time.time())

    for i in range(1050):
        mt = MT19937(unix + i)
        if mt.extract_num() == output:
            return unix + i
```

If you've worked through the [SEED Labs](https://seedsecuritylabs.org/index.html), this challenge is very similar to their [PRNG lab](https://seedsecuritylabs.org/Labs_20.04/Files/Crypto_Random_Number/Crypto_Random_Number.pdf). 

## 23. Clone an MT19937 RNG from its output
> The internal state of MT19937 consists of 624 32 bit integers.
> 
> For each batch of 624 outputs, MT permutes that internal state. By permuting state regularly, MT19937 achieves a period of 2**19937, which is Big.
> 
> Each time MT19937 is tapped, an element of its internal state is subjected to a tempering function that diffuses bits through the result.
> 
> The tempering function is invertible; you can write an "untemper" function that takes an MT19937 output and transforms it back into the corresponding element of the MT19937 state array.
> 
> To invert the temper transform, apply the inverse of each of the operations in the temper transform in reverse order. There are two kinds of operations in the temper transform each applied twice; one is an XOR against a right-shifted value, and the other is an XOR against a left-shifted value AND'd with a magic number. So you'll need code to invert the "right" and the "left" operation.
> 
> Once you have "untemper" working, create a new MT19937 generator, tap it for 624 outputs, untemper each of them to recreate the state of the generator, and splice that state into a new instance of the MT19937 generator.
> 
> The new "spliced" generator should predict the values of the original. 

I'm still working on this one! Untempering the bitwise operations is a little tricky :)

## 24. Create the MT19937 stream cipher and break it

> You can create a trivial stream cipher out of any PRNG; use it to generate a sequence of 8 bit outputs and call those outputs a keystream. XOR each byte of plaintext with each successive byte of keystream.
> 
> Write the function that does this for MT19937 using a 16-bit seed. Verify that you can encrypt and decrypt properly. This code should look similar to your CTR code.
> 
> Use your function to encrypt a known plaintext (say, 14 consecutive 'A' characters) prefixed by a random number of random characters.
> 
> From the ciphertext, recover the "key" (the 16 bit seed).
> 
> Use the same idea to generate a random "password reset token" using MT19937 seeded from the current time.
> 
> Write a function to check if any given password token is actually the product of an MT19937 PRNG seeded with the current time. 

I'm still working on this one as well!
