---
title: "Cryptopals Set 2: Block crypto"
date: 2022-12-23T00:00:08-05:00
draft: false
---

Continuing with [Cryptopals](https://cryptopals.com/), the second set focuses on block ciphers. These were a lot of fun! Trickier than the second set but also more useful and fun to work on!


## 9. Implement PKCS#7 Padding

> A block cipher transforms a fixed-sized block (usually 8 or 16 bytes) of plaintext into ciphertext. But we almost never want to transform a single block; we encrypt irregularly-sized messages.
> 
> One way we account for irregularly-sized messages is by padding, creating a plaintext that is an even multiple of the blocksize. The most popular padding scheme is called PKCS#7.
> 
> So: pad any block to a specific block length, by appending the number of bytes of padding to the end of the block. For instance,
> 
> ```
> "YELLOW SUBMARINE"
> ```
> 
> ... padded to 20 bytes would be:
> 
> ```
> "YELLOW SUBMARINE\x04\x04\x04\x04"
> ```

[Wikipedia](https://en.wikipedia.org/wiki/Padding_(cryptography)#PKCS#5_and_PKCS#7) is pretty clear on how to implement PKCS#7. At first, I missed that if the unpadded text is a multiple of the block length, an extra block is added at the end!

```
def pad(unpadded, block_len):
    to_pad = (block_len - len(unpadded)) if len(unpadded) % block_len != 0 else block_len
    padded = bytearray(unpadded)
    for i in range(to_pad):
        padded.append(to_pad)
    return padded
```

## 10. Implement CBC mode

> CBC mode is a block cipher mode that allows us to encrypt irregularly-sized messages, despite the fact that a block cipher natively only transforms individual blocks.
> 
> In CBC mode, each ciphertext block is added to the next plaintext block before the next call to the cipher core.
> 
> The first plaintext block, which has no associated previous ciphertext block, is added to a "fake 0th ciphertext block" called the initialization vector, or IV.
> 
> Implement CBC mode by hand by taking the ECB function you wrote earlier, making it encrypt instead of decrypt (verify this by decrypting whatever you encrypt to test), and using your XOR function from the previous exercise to combine them.
> 
> The file here is intelligible (somewhat) when CBC decrypted against "YELLOW SUBMARINE" with an IV of all ASCII 0 (\x00\x00\x00) 

Implementing CBC mode is the first challenge where it starts to get a bit more complex. We'll write an encryption and a decryption function, making sure that we pad the blocks to 16-bytes. Also, since we already wrote an ECB encrypt/decrypt functions, we'll leverage those.

```
def cbc_encrypt(pt, key, iv):
    padded = pad(pt, len(pt) + (16 - len(pt) % 16)) # pad to next multiple of 16
    blocks = list(divide_chunks(padded, 16))

    output = bytearray()
    prev = iv
    for block in blocks:
        xord = xor(prev, block)
        ct = ecb_encrypt(xord, key)
        prev = ct

        output += ct
    return output
```

Decrypt is almost identical, except we unpad at the end:

```
def cbc_decrypt(ct, key, iv):
    blocks = list(divide_chunks(ct, 16))

    output = bytearray()
    prev = iv
    for block in blocks:
        pt = ecb_decrypt(block, key)
        xord = xor(pt, prev)
        prev = block

        output += xord

    unpadded = unpad(output)
    return unpadded
```
## 11. An ECB/CBC detection oracle

> Now that you have ECB and CBC working:
> 
> Write a function to generate a random AES key; that's just 16 random bytes.
> 
> Write a function that encrypts data under an unknown key --- that is, a function that generates a random key and encrypts under it.
> 
> The function should look like:
> ```
> encryption_oracle(your-input)
> => [MEANINGLESS JIBBER JABBER]
> ```
> 
> Under the hood, have the function append 5-10 bytes (count chosen randomly) before the plaintext and 5-10 bytes after the plaintext.
> 
> Now, have the function choose to encrypt under ECB 1/2 the time, and under CBC the other half (just use random IVs each time for CBC). Use rand(2) to decide which to use.
> 
> Detect the block cipher mode the function is using each time. You should end up with a piece of code that, pointed at a block box that might be encrypting ECB or CBC, tells you which one is happening. 

Now we're really starting to get into it! This challenge relies on the hamming_dist() function we wrote for challenge 3, the divide_chunks() from challenge 5, and our ECB/CBC encryption. One of my favorite things about Cryptopals is how much each challenge builds off of each other. Also, this means a lot of restructuring my code to be reused the next time I need it!

Let's start off with our function to encrypt a ciphertext using either ECB or CBC.

```
def encrypt_rand(data):
    key = gen_key()
    before, after = random.randint(5, 10), random.randint(5, 10)
    bytes_before, bytes_after = os.urandom(before), os.urandom(after)

    data = bytes_before + data + bytes_after
    padded_data = pad(data, len(data) + (16 - len(data) % 16))

    ecb = random.randint(0, 1)
    ct = None

    if ecb:
        ct = ecb_encrypt(padded_data, key)
    else:
        iv = os.urandom(16)
        ct = cbc_encrypt(data, key, iv)
    return ct, ecb
```

One thing to note, for ECB mode we're encrypting the padded plaintext while for CBC mode we're encrypting an unpadded plaintext. These are effectively equivalent since my cbc_encrypt() will pad the data. cbc_encrypt() uses ecb_encrypt() under the hood though, and I don't want to double-pad data.

Next, let's write our oracle. The approach is somewhat similar to challenge 8, where the presence of a repeated block definitely means ECB was used. We want to see if we can get better than that though!

```
def oracle(ct, pt):
    chunks = list(divide_chunks(ct, 16))
    s = set()
    ecb = False
    for i in range(len(chunks)-1):
        if str(chunks[i]) in s:
            ecb = True
            return ecb
        s.add(str(chunks[i]))

    return ecb
```

Above we have an oracle that given a ciphertext, determines whether ECB or CBC mode was used to encrypt it. We're passing a long plaintext of all 0's. Whatever the random bytes prepended/appended, we'll always have two blocks of full 0's. This ensures a repeated block and makes easy detection of ECB mode!

## 12. Byte-at-a-time ECB decryption (Simple)

>  Copy your oracle function to a new function that encrypts buffers under ECB mode using a consistent but unknown key (for instance, assign a single random key, once, to a global variable).
> 
> Now take that same function and have it append to the plaintext, BEFORE ENCRYPTING, the following string:
> 
> ```
> Um9sbGluJyBpbiBteSA1LjAKV2l0aCBteSByYWctdG9wIGRvd24gc28gbXkg
> aGFpciBjYW4gYmxvdwpUaGUgZ2lybGllcyBvbiBzdGFuZGJ5IHdhdmluZyBq
> dXN0IHRvIHNheSBoaQpEaWQgeW91IHN0b3A/IE5vLCBJIGp1c3QgZHJvdmUg
> YnkK
> ```
> 
> Base64 decode the string before appending it. Do not base64 decode the string by hand; make your code do it. The point is that you don't know its contents.
> 
> What you have now is a function that produces:
> 
> ```
> AES-128-ECB(your-string || unknown-string, random-key)
> ```
> 
> It turns out: you can decrypt "unknown-string" with repeated calls to the oracle function!
> 
> Here's roughly how:
> 
> 1. Feed identical bytes of your-string to the function 1 at a time --- start with 1 byte ("A"), then "AA", then "AAA" and so on. Discover the block size of the cipher. You know it, but do this step anyway.
> 2. Detect that the function is using ECB. You already know, but do this step anyways.
> 3. Knowing the block size, craft an input block that is exactly 1 byte short (for instance, if the block size is 8 bytes, make "AAAAAAA"). Think about what the oracle function is going to put in that last byte position.
> 4. Make a dictionary of every possible last byte by feeding different strings to the oracle; for instance, "AAAAAAAA", "AAAAAAAB", "AAAAAAAC", remembering the first block of each invocation.
> 5. Match the output of the one-byte-short input to one of the entries in your dictionary. You've now discovered the first byte of unknown-string.
> 6. Repeat for the next byte.
> 
> This is the first challenge we've given you whose solution will break real crypto. Lots of people know that when you encrypt something in ECB mode, you can see penguins through it. Not so many of them can decrypt the contents of those ciphertexts, and now you can. If our experience is any guideline, this attack will get you code execution in security tests about once a year.



Before this challenge, I knew that ECB leaks data in the form of a Tux but did not know that you could exploit it to fully recover plaintexts! That's exactly what we'll be doing for this challenge.

As an aside, there's an awesome, Warholesque ECB Tux image I found [here.](https://words.filippo.io/the-ecb-penguin/) 

![ECB'd Tux](https://words.filippo.io/content/images/2015/11/POP-xsmall.png)

Let's start by picking a random key to encrypt everything under and writing a function to give us encrypted ciphertexts:

```
b64 = "Um9sbGluJyBpbiBteSA1LjAKV2l0aCBteSByYWctdG9wIGRvd24gc28gbXkgaGFpciBjYW4gYmxvdwpUaGUgZ2lybGllcyBvbiBzdGFuZGJ5IHdhdmluZyBqdXN0IHRvIHNheSBoaQpEaWQgeW91IHN0b3A/IE5vLCBJIGp1c3QgZHJvdmUgYnkK"
p_b64 = base64.b64decode(b64)
rand_key = os.urandom(16)

def ecb_baat(pt):
    pt = pt + p_b64
    ct = ecb_encrypt(pt, rand_key)
    return ct
```

To decode a block, we'll follow this algorithm:

1. Create 15 byte plaintext
2. Encrypt 15 byte plaintext + all possible 16th bytes (255)
3. Pass 15 byte plaintext to oracle
4. We can match the received ciphertext to see what the 16th byte was! We now know the first byte of the unknown string
5. Repeat, creating a 14 byte plaintext + our decoded byte

```
def guess_block(index, prev):
    plaintext = prev
    for j in range(15, -1, -1):
        d = {}
        data = bytearray(("A"*(j)).encode('ascii'))
        data += bytearray(plaintext.encode('ascii'))
        
        # store all possible last bytes
        for i in range(0, 255):
            data.append(i)

            ct = ecb_baat(data)
            d[str(ct[(16*index):16*(index+1)])] = i

        # decode unknown byte using oracle
        data = bytearray(("A"*(j)).encode('ascii'))
        ct = ecb_baat(data)
        plaintext += chr(d[str(ct[(16*index):16*(index+1)])])

    return plaintext 
```

Using this function, we can decode the 9 blocks of the unknown string. To each block, we'll pass in what we've decoded up until now.

```
decoded = ''
for i in range(0, 9):
    decoded = guess_block(i, decoded)
```

After 9 blocks, we're left with a fully decoded plaintext!

## 13. ECB cut-and-paste

> Write a k=v parsing routine, as if for a structured cookie. The routine should take:
> 
> ```
> foo=bar&baz=qux&zap=zazzle
> ```
> 
> ... and produce:
> 
> ```
> {
>   foo: 'bar',
>   baz: 'qux',
>   zap: 'zazzle'
> }
> ```
> 
> (you know, the object; I don't care if you convert it to JSON).
> 
> Now write a function that encodes a user profile in that format, given an email address. You should have something like:
> 
> ```
> profile_for("foo@bar.com")
> ```
> 
> ... and it should produce:
> 
> ```
> {
>   email: 'foo@bar.com',
>   uid: 10,
>   role: 'user'
>  }
> ```
> 
> ... encoded as:
> 
> ```
> email=foo@bar.com&uid=10&role=user
> ```
> 
> Your "profile_for" function should not allow encoding metacharacters (& and =). Eat them, quote them, whatever you want to do, but don't let people set their email address to "foo@bar.com&role=admin".
> 
> Now, two more easy functions. Generate a random AES key, then:
> 
> 1. Encrypt the encoded user profile under the key; "provide" that to the "attacker".
> 2. Decrypt the encoded user profile and parse it.
> 
> Using only the user input to ``profile_for()`` (as an oracle to generate "valid" ciphertexts) and the ciphertexts themselves, make a ``role=admin`` profile. 

In our 13th challenge, we'll be taking advantage of the fact that ECB has no ciphertext integrity! Also, since ECB of a block always results in the same ciphertext, we can easily "cut-and-paste" a new block in! The tricky part is lining up the blocks so that the text still makes sense when decrypted.

Let's start by writing our ``encode_profile()`` and ``parse()`` function. 

```
def parse(unparsed):
    key_values = unparsed.split("&")

    d = {}
    for key_value in key_values:
        key, value = key_value.split("=")
        d[key] = value
    return d

def encode_profile(profile):
    profile = profile.replace("&", "").replace("=", "")

    email = "email=" + profile
    role = "role=user"
    uid = "uid=10"

    return email + "&" + uid + "&" + role
```

There are better ways to strip the encoding characters, but this is good enough for us.

To make our testing easier, we want to encrypt and decrypt a profile under a fixed key:


```
key = os.urandom(16)

def encrypt_under_key(profile):
    pt = encode_profile(profile)
    pt = pt.encode('ascii')

    return padded_ecb_encrypt(pt, key)

def decrypt_under_key(ct, key):
    pt = padded_ecb_decrypt(ct, key)
    pt = pt.decode('ascii')
    return parse(pt)
```

Our objective is to get a profile with the role admin. To do this, we'll need to get an encrypted block of role=admin and then splice it into our new ciphertext. Let's look at how we're constructing the plaintext to be encrypted:

```
    return "email=" + profile + "&uid=10&role=user"
```

We have control only over the email profile name. We want to have the plaintext end with "role=admin". To do this, we'll pass in a profile with text "role=admin" crafted so that it is in its own block. Since this is also the last block, we'll need sufficient padding so that it decrypts successfully.


```
  |   6 bytes  +   10 bytes    |   6 bytes + 10 bytes   |      16 bytes         |    1 byte + 15 bytes   |
  |   "email=" + "AAAAAAAAAA"  |    "admin" + '\x0A'*10 |  "&uid=10&role=use"   |     "r" + '\x0f'*15    |
  |       first block          |       second block     |     third block       |        fourth block    |
```

We pad the beginning of our string with A's so that our admin string ends up in its own block. We will splice this second block onto the end of our new ciphertext.

Now let's craft our goal ciphertext. The important part is that we have "user" end up in its own block.

```
  | 6 bytes  + X bytes +   13 bytes      |   4 bytes        |
  | "email=" + profile + "&uid=10&role=" |    "user"        |
  |        first blocks                  |   last block     |
```

To do this, we need the first blocks to be a multiple of 16. 6 + 13 = 19. So we need to craft a message of 13 bytes!

Let's set our email to:
```
profile = "foooo@bar.com" # 13 bytes
```

Now we can splice our last block onto this!


```
profile = "foooo@bar.com"
ct1 = encrypt_under_key(profile)

padded = bytearray(("A"*10).encode('ascii'))
last_block = bytearray('admin'.encode('ascii'))
for i in range(0, 11):
    last_block.append(11)

# 'AAAAAAAAAAadmin\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b'
padded = (padded + last_block).decode('ascii')
ct1 = encrypt_under_key(padded)

admin_last_block = ct1[16:]
spliced_ct = ct0[:32] + ct1[16:32]
profile = decrypt_under_key(spliced_ct, key)

# {'email': 'foooo@bar.com', 'uid': '10', 'role': 'admin'}
```

Looks good! By lining up our blocks carefully, we can cut our desired plaintext onto another and get admin!

## 14. Byte-at-a-time ECB decryption (Harder)

> Take your oracle function from #12. Now generate a random count of random bytes and prepend this string to every plaintext. You are now doing:
> 
> ```
> AES-128-ECB(random-prefix || attacker-controlled || target-bytes, random-key)
> ```
> 
> Same goal: decrypt the target-bytes.
> Stop and think for a second.
> 
> What's harder than challenge #12 about doing this? How would you overcome that obstacle? The hint is: you're using all the tools you already have; no crazy math is required.
> 
> Think "STIMULUS" and "RESPONSE".


I was originally a little confused on how to approach this problem. If we generate a prefix of random length each time the oracle encrypts, it would be difficult to successfully decrypt. I think it's still doable, but looking at other peoples write-ups, it seems they generate a fixed random prefix each time the file is run. That's more doable! Let's take it on.

We already have the code to do byte-at-a-time ECB decryption from challenge 12. The only complication, is we don't know how many bytes we need to be 1-byte short of a block. Once we learn the prefix length, we can approach the challenge just like 12.

```
def decode_length():
    for i in range(48):
        pt = ("A"*i).encode('ascii')
        ct = ecb_baat_harder(pt)
        chunks = list(divide_chunks(ct, 16))

        s = set()
        for j in range(len(chunks)):
            if str(chunks[j]) in s:
                return (16 - i) % 16
            s.add(str(chunks[j]))
```

The idea here is to add bytes until we see a repeated ECB ciphertext block. If we see a repeat, we know that there are some bytes filling up the padding block and then 32 bytes of "A". So, if we've added `i` blocks of padding to get a repeated block, that must mean there is (16 - i) % 16 blocks of prefix!

```
  |  random prefix      padding      |  "A"*16   |   "A"*16  |
  |  ..............   (16 - i) % 16  |    16     |    16     |
```

Know that we know the length of the prefix, we just pad the first block of our plaintext to it's full size (16 bytes). Then we solve the same way as in challenge 13! 

## 15. PKCS#7 padding validation

> Write a function that takes a plaintext, determines if it has valid PKCS#7 padding, and strips the padding off.
> 
> The string:
> 
> ```
> "ICE ICE BABY\x04\x04\x04\x04"
> ```
> 
> ... has valid padding, and produces the result "ICE ICE BABY".
> 
> The string:
> 
> ```
> "ICE ICE BABY\x05\x05\x05\x05"
> ```
> 
> ... does not have valid padding, nor does:
> 
> ```
> "ICE ICE BABY\x01\x02\x03\x04"
> ```
> 
> If you are writing in a language with exceptions, like Python or Ruby, make your function throw an exception on bad padding.
> 
> Crypto nerds know where we're going with this. Bear with us. 

For this challenge, we're extending our PKCS#7 padding from challenge 9 to throw exceptions on invalid padding.


```
def safe_unpad(padded):
    to_unpad = None
    if type(padded) != bytearray:
        to_unpad = int.from_bytes(padded[-1].encode('ascii'), byteorder='big', signed=False)
    else:
        to_unpad = padded[-1]

    unpadded = to_unpad
    last_byte = padded[-1]
    i = len(padded)-1

    while i >= len(padded)-to_unpad:
        if i < 0 or padded[i] != last_byte:
            raise ValueError
        i -= 1
        unpadded -= 1
    if unpadded >= 1:
        raise ValueError

    return padded[:-to_unpad]
```

The basic idea, a padded text should have `i` bytes of padding with value `i` where `i` is the value of the last byte.

## 16. CBC bitflipping attacks

> Generate a random AES key.
> 
> Combine your padding code and CBC code to write two functions.
> 
> The first function should take an arbitrary input string, prepend the string:
> 
> ```
> "comment1=cooking%20MCs;userdata="
> ```
> 
> .. and append the string:
> 
> ```
> ";comment2=%20like%20a%20pound%20of%20bacon"
> ```
> 
> The function should quote out the ";" and "=" characters.
> 
> The function should then pad out the input to the 16-byte AES block length and encrypt it under the random AES key.
> 
> The second function should decrypt the string and look for the characters ";admin=true;" (or, equivalently, decrypt, split the string on ";", convert each resulting string into 2-tuples, and look for the "admin" tuple).
> 
> Return true or false based on whether the string exists.
> 
> If you've written the first function properly, it should not be possible to provide user input to it that will generate the string the second function is looking for. We'll have to break the crypto to do that.
> 
> Instead, modify the ciphertext (without knowledge of the AES key) to accomplish this.
> 
> You're relying on the fact that in CBC mode, a 1-bit error in a ciphertext block:
> 
> 1. Completely scrambles the block the error occurs in
> 2. Produces the identical 1-bit error(/edit) in the next ciphertext block.
> 

Similar to the ECB cut-and-paste, this is another attack on ciphertext integrity. We'll take advantage of the fact that a 1-bit flip in a block flips the same bit in the next block. Our goal is to get ";admin=true;" to be a part of the plaintext. However, we'll quote out ; and = characters, so we'll have to flip some bits to get there!

The challenge asks us to place our text in between a suffix and a prefix, let's do that:


```
from urllib.parse import quote

prefix = "comment1=cooking%20MCs;userdata="
suffix = ";comment2=%20like%20a%20pound%20of%20bacon"

def bitflip_encrypt(data):
    data = prefix + data + suffix
    data = quote(data)
    data = data.encode('ascii')

    return cbc_encrypt(data, key, iv)

known_plaintext = "AAAAAAAAAAAAAAAAAAAAAAAAAA"
ct = bitflip_encrypt(known_plaintext)
```

Next, let's figure out how we can decrypt it so that we get our target string. CBC xor's the previous ciphertext against the decrypted next block. If we know what the decrypted block is, we can XOR the previous block to get our goal text!



```
def bitflip_decrypt(ct):
    goal_string = ";admin=true;AAAA".encode('ascii')
    known_plaintext = known_plaintext.encode('ascii')
    xord = xor(goal_string, known_plaintext)

    modified_ct = ct[:32] + xor(xord, ct[32:48]) + ct[48:]
    pt = cbc_decrypt(modified_ct, key, iv)
    return ";admin=true;" in str(pt)
detected = bitflip_decrypt(ct)
# True
```

It was a little tricky getting the blocks to line up.  My xor() function drops the remainder if one buffer is shorter than another. So, we append "AAAA" to our goal string to pad it to 16 bytes. The plaintext we're interested in is at ct[48:64]. However, since each block gets xor'd with the previous one, we xor our target string into the block before!

# Conclusion

The second set of Cryptopals starts to dive into block ciphers and some attacks on ECB and CBC mode. The byte-at-a-time ECB decryption was extra interesting to work on. I didn't realize that ECB could be exploited this way using a chosen-plaintext attack.
