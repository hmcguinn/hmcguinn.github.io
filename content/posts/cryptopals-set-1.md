---
title: "Cryptopals Set 1: Basics"
date: 2022-12-21T13:12:35-05:00
draft: false
---

[Cryptopals](https://cryptopals.com) is a set of cryptography challenges by the NCC Group. Maciej Ceglowski wrote a great blog about them [here](https://blog.pinboard.in/2013/04/the_matasano_crypto_challenges/). They've been out for awhile and I originally tried to work through them four years ago in high school. I got to about the second challenge and then was stumped. Now, four years and a CS degree later, I'm back at it! They've been a ton of fun to work on so far and definitely challenging. I've "known" academically a lot of the attacks covered but implementing them is a different story. 

This is a walkthrough of how I've approached the challenges in Set 1!

## 1. Convert hex to base64

>  The string:
> ```
> 49276d206b696c6c696e6720796f757220627261696e206c696b65206120706f69736f6e6f7573206d757368726f6f6d
> ```
> 
> Should produce:
> 
> ```
> SSdtIGtpbGxpbmcgeW91ciBicmFpbiBsaWtlIGEgcG9pc29ub3VzIG11c2hyb29t
> ```
> 
> So go ahead and make that happen. You'll need to use this code for the rest of the exercises.
> Cryptopals Rule
> 
> Always operate on raw bytes, never on encoded strings. Only use hex and base64 for pretty-printing.


The challenges start off fairly simple. For this one, we'll just convert this hex-encoded string to a b64-encoded string. The challenge wants us to operate on raw bytes, so we'll first have to convert the hex-encoded-string to bytes:
```
import base64

def hex_to_bytes(s):
    return bytes.fromhex(s)

def bytes_to_b64(b):
    return base64.b64encode(b)

hex_string = "49276d206b696c6c696e6720796f757220627261696e206c696b65206120706f69736f6e6f7573206d757368726f6f6d"
hex_bytes = hex_to_bytes(hex_string)
b64_bytes = bytes_to_b64(hex_bytes)
# b'SSdtIGtpbGxpbmcgeW91ciBicmFpbiBsaWtlIGEgcG9pc29ub3VzIG11c2hyb29t'
```

Looks good!

## 2. Fixed XOR

> Write a function that takes two equal-length buffers and produces their XOR combination.
> 
> If your function works properly, then when you feed it the string:
> 
> ```
> 1c0111001f010100061a024b53535009181c
> ```
> 
> ... after hex decoding, and when XOR'd against:
> 
> ```
> 686974207468652062756c6c277320657965
> ```
> 
> ... should produce:
> 
> ```
> 746865206b696420646f6e277420706c6179
> ```

Next up is to XOR two byte buffers together. The challenge input is given as a hex string, so like last time, we'll have to convert this to bytes first. Luckily we just wrote a function to do that!


```
def xor(b1, b2):
    xord = bytearray(min(len(b1), len(b2)))
    for i in range(min(len(b1), len(b2))):
        xord[i] = b1[i] ^ b2[i]
    return xord

b1 = hex_to_bytes("1c0111001f010100061a024b53535009181c")
b2 = hex_to_bytes("686974207468652062756c6c277320657965")

xord = xor(b1, b2)

# 746865206b696420646f6e277420706c6179
# bytearray(b"the kid don\'t play")
```

## 3. Single-byte XOR cipher

>  The hex encoded string:
> ```
> 1b37373331363f78151b7f2b783431333d78397828372d363c78373e783a393b3736
> ```
> 
> ... has been XOR'd against a single character. Find the key, decrypt the message.
> 
> You can do this by hand. But don't: write code to do it for you.
> 
> How? Devise some method for "scoring" a piece of English plaintext. Character frequency is a good metric. Evaluate each output and choose the one with the best score. 
> 

We're given a hex encoded string that's been XOR'd with an unknown key.

```
ciphertext = hex_to_bytes("1b37373331363f78151b7f2b783431333d78397828372d363c78373e783a393b3736")
```

Our algorithm is fairly simple, we'll try XOR'ing all possible keys with our ciphertext. We know it's a single character so there are only 2^8 possible keys. We can then score the result to see which are most likely.
```
def detect_xor(ct):
    candidates = []
    for i in range(256):
        candidate = xor(ct, i.to_bytes(1, 'big')*len(ct))
        num_valid = 0
        for j in candidate:
            # space or lowercase
            if  int(j) == 32 or (int(j) >= 97 and int(j) <= 122):
                num_valid += 1
        candidates.append((num_valid, candidate, chr(i)))
    candidates.sort()
    return candidates[-1]
```


To score our messages, we assume that it consists of mostly lowercase characters and spaces. The one with the most lowercase characters and spaces is our plaintext!

```
plain = detect_xor(ciphertext)[1:]
# (bytearray(b"Cooking MC\'s like a pound of bacon"), 'X')
```

Here, the key our ciphertext was XOR'd with was the character 'X'.

## 4. Detect single-character XOR

> One of the 60-character strings in this file has been encrypted by single-character XOR.
> 
> Find it. 
> 
>  (Your code from #3 should help.) 

This challenge is pretty similar to the previous one. We're given a number of strings, one of which was "encrypted" by single-character XOR. Using the detect_xor() function we wrote for the last challenge, we can try each line individually.

```
with open('detect_xor.txt') as f:
    lines = f.readlines()

arr = []
for line in lines:
    detect = detect_xor(hex_to_bytes(line))
    arr.append((detect, line))

arr.sort()
plaintext = arr[-1][0][1]
ciphertext = arr[-1][1]

# bytearray(b'Now that the party is jumping')
# '7b5a4215415d544115415d5015455447414c155c46155f4058455c5b523f'

```

Again, we're making a similar assumption that the most likely plaintext has the most lowercase characters and spaces. There are fancier ways to do this (frequency of characters in English (ETAOIN)) but this is simple and works.

## 5. Implement repeating-key XOR

>
>  Here is the opening stanza of an important work of the English language:
> 
> ```
> "Burning 'em, if you ain't quick and nimble
> I go crazy when I hear a cymbal"
> ```
> 
> Encrypt it, under the key "ICE", using repeating-key XOR.
> 
> In repeating-key XOR, you'll sequentially apply each byte of the key; the first byte of plaintext will be XOR'd against I, the next C, the next E, then I again for the 4th byte, and so on.
> 
> It should come out to:
> 
> ```
> 0b3637272a2b2e63622c2e69692a23693a2a3c6324202d623d63343c2a26226324272765272
> a282b2f20430a652e2c652a3124333a653e2b2027630c692b20283165286326302e27282f
> ```

Let's implement a very secure cipher!


```
pt = "Burning 'em, if you ain't quick and nimble\nI go crazy when I hear a cymbal".encode('ascii')
key = "ICE".encode('ascii')

def xor_encrypt(key, pt):
    key = (key*(len(pt)//len(key)+1))[:len(pt)]
    return xor(key, pt)

ct = xor_encrypt(key, pt)
# '0b3637272a2b2e63622c2e69692a23693a2a3c6324202d623d63343c2a26226324272765272a282b2f20430a652e2c652a3124333a653e2b2027630c692b20283165286326302e27282f'
```

The tricky part here is to expand our repeating key to the right length. We already have the XOR function from challenge 2. We repeat the key until it's longer than our plaintext, and then splice the repeated key so that the lengths match exactly. Then we can just XOR them!

## 6. Break repeating-key XOR

> There's a file here. It's been base64'd after being encrypted with repeating-key XOR.
> 
> Decrypt it.
> 
> Here's how:
> 1. Let KEYSIZE be the guessed length of the key; try values from 2 to (say) 40.
> 2. Write a function to compute the edit distance/Hamming distance between two strings. The Hamming distance is just the number of differing bits. The distance between:
> 3. For each KEYSIZE, take the first KEYSIZE worth of bytes, and the second KEYSIZE worth of bytes, and find the edit distance between them. Normalize this result by dividing by KEYSIZE.
> 4. The KEYSIZE with the smallest normalized edit distance is probably the key. You could proceed perhaps with the smallest 2-3 KEYSIZE values. Or take 4 KEYSIZE blocks instead of 2 and average the distances.
> 5. Now that you probably know the KEYSIZE: break the ciphertext into blocks of KEYSIZE length.
> 6. Now transpose the blocks: make a block that is the first byte of every block, and a block that is the second byte of every block, and so on.
> 7. Solve each block as if it was single-character XOR. You already have code to do this.
> 8. For each block, the single-byte XOR key that produces the best looking histogram is the repeating-key XOR key byte for that block. Put them together and you have the key.
> 
> This code is going to turn out to be surprisingly useful later on. Breaking repeating-key XOR ("Vigenere") statistically is obviously an academic exercise, a "Crypto 101" thing. But more people "know how" to break it than can actually break it, and a similar technique breaks something much more important. 
> 

This is the trickiest challenge so far. Luckily, we're given pretty specific steps on how we can break it!

Let's start by writing a function to calculate the hamming distance:

```
def hamming_dist(b1, b2):
    diff = 0
    for i in range(min(len(b1[i]), len(b2[i]))):
        xor = b1[i] ^ b2[i]
        distance = 0
        while xor:
            if xor & 1:
                distance += 1
            xor = xor >> 1
        diff += distance
    return diff


s1 = "this is a test".encode('ascii')
s2 = "wokka wokka!!!".encode('ascii')
hamming_distance = hamming_dist(s1, s2)
# 37
```

Now we need to figure out what the keysize is so we can apply our detect_xor() function from challenge 4. I found that it worked better when I average a couple hamming distances, rather than just the first two ciphertexts.


```
min_hamming = 10000
guessed_size = 0
distances = []
for keysize in range(2, 40):
    g1, g2, g3, g4 = ct[0:keysize*1], ct[keysize*1:keysize*2], ct[keysize*2:keysize*3], ct[keysize*3:keysize*4]

    dist = ((hamming_dist(g1, g2)) +
            (hamming_dist(g2, g3)) +
            (hamming_dist(g3, g4))) / (3*keysize)
    if dist < min_hamming:
        min_hamming = dist
        guessed_size = keysize

    distances.append((dist, keysize))


print(guessed_size)
```

Next for each keysize, we'll splice together the ciphertexts and break it as single-character XOR! We have this helper function courtesy of [geeks for geeks](https://www.geeksforgeeks.org/break-list-chunks-size-n-python/) that splits a list into chunks of length n. We'll also write a quick helper function using our logic from challenge 3 to score the likelihood a decoded plaintext is valid.

```
def divide_chunks(c, n):
    for i in range(0, len(c), n):
        yield c[i:i + n]

def score(s):
    num_valid = 0
    for j in s:
        # space or lowercase character
        if  int(j) == 32 or (int(j) >= 97 and int(j) <= 122):
            num_valid += 1
    return num_valid
```

The steps to break:

1. Split into chunks of length keysize
2. Reorder len(ct)/keysize chunks of length keysize into keysize chunks of length len(ct)/keysize
3. Break keysize chunks individually as single character XOR
4. Decrypt ciphertext using guessed key
5. Score decoded plaintexts by likelihood

```
plaintexts = []
for _, keysize in distances:
    split = list(divide_chunks(ct, keysize))
    broken = [[] for _ in range(keysize)]

    for i in range(len(split)):
        for j in range(len(split[i])):
            broken[j].append(split[i][j])

    guessed_key = []
    for block in broken:
        guessed_key.append(break_single_key_xor(block)[0])

    guess_pt = xor_encrypt((''.join(guess).encode('ascii')), ct)
    plaintexts.append((score(guess_pt), guess_pt, ''.join(guessed_key)))

plaintexts.sort()
key = plaintexts[-1][2]
# Terminator X: Bring the noise
```

## 7. AES in ECB mode

>  The Base64-encoded content in this file has been encrypted via AES-128 in ECB mode under the key
> 
> ```
> "YELLOW SUBMARINE".
> ```
> 
> (case-sensitive, without the quotes; exactly 16 characters; I like "YELLOW SUBMARINE" because it's exactly 16 bytes long, and now you do too).
> 
> Decrypt it. You know the key, after all.
> 
> Easiest way: use OpenSSL::Cipher and give it AES-128-ECB as the cipher.
> **Do this with code.**
> 
> You can obviously decrypt this using the OpenSSL command-line tool, but we're having you get ECB working in code for a reason. You'll need it a lot later on, and not just for attacking ECB.

For this challenge, we're implementing AES in ECB mode.

```
with open('encrypted-cbc.txt') as f:
    file = base64.b64decode(f.read())

key = "YELLOW SUBMARINE".encode('ascii')
```

Let's write two functions to ECB encrypt/decrypt. Since I'm using Pyca's [Cryptography library](https://cryptography.io/en/latest/), I need to pad the plaintext to a multiple of the blocksize. For this specific challenge our ciphertext is already a multiple of the blocksize so we can leave that out for now.


```
def ecb_encrypt(pt, key):
    cipher = Cipher(algorithms.AES(key), modes.ECB())
    encryptor = cipher.encryptor()
    return encryptor.update(pt) + encryptor.finalize()

def ecb_decrypt(ct, key):
    cipher = Cipher(algorithms.AES(key), modes.ECB())
    decryptor = cipher.decryptor()

    return decryptor.update(ct)

pt = ecb_decrypt(file, key)
```

## 8. Detect AES in ECB mode

> In this file are a bunch of hex-encoded ciphertexts.
> 
> One of them has been encrypted with ECB.
> 
> Detect it.
> 
> Remember that the problem with ECB is that it is stateless and deterministic; the same 16 byte plaintext block will always produce the same 16 byte ciphertext. 

For this challenge, we're detecting which ciphertext has been encrypted with ECB. To do this, we'll make the assumption that there is some repeating block of plaintext and therefore a repeating block of ciphertext too!

First, we'll read in the file and convert the hex-encoded string to bytes.
```
with open('8.txt') as f:
    lines = f.readlines()

arr = []
for line in lines:
    arr.append(hex_to_bytes(line))
```

Next, we'll check each ciphertext for a repeating block:

```
encrypted = None
for i in arr:
    s = set()
    ct = list(divide_chunks(i, 16))
    for j in ct:
        if str(j) in s:
            encrypted = i
            break
        s.add(str(j))
    if encrypted:
        break

print(encrypted.hex())
```

There is definitely a better way than casting the bytearray to a string to find repeated blocks, but it's easy enough to do it this way!

# Conclusion

The first set of Cryptopals was a lot of fun to work on! Breaking repeating-key XOR was probably my favorite one to work on, and getting some of the implementation right was a little tricky. The concepts here will come up again later when we look at breaking fixed nonce CTR encryption!
