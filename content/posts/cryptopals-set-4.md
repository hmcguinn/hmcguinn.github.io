---
title: "Cryptopals Set 4: Stream crypto and randomness"
date: 2022-12-28T10:46:02-05:00
draft: true
---


## 25. Break "random access read/write" AES CTR
> Back to CTR. Encrypt the recovered plaintext from this file (the ECB exercise) under CTR with a random key (for this exercise the key should be unknown to you, but hold on to it).
> 
> Now, write the code that allows you to "seek" into the ciphertext, decrypt, and re-encrypt with different plaintext. Expose this as a function, like, "edit(ciphertext, key, offset, newtext)".
> 
> Imagine the "edit" function was exposed to attackers by means of an API call that didn't reveal the key or the original plaintext; the attacker has the ciphertext and controls the offset and "new text".
> 
> Recover the original plaintext. 

For this challenge, we will mount a chosen-plaintext attack on a CTR mode ciphertext with an unknown key. Since CTR mode works by XOR'ing the keystream with the plaintext, if we control the plaintext we can recover the keystream and then ultimately obtain the plaintext!
```
pt = "I'm back and I'm ringin' the bell \nA rockin' on the mike while the fly girls yell \nIn ecstasy in the back of me \nWell that's my DJ Deshay cuttin' all them Z's \nHittin' hard and the girlies goin' crazy \nVanilla's on the mike, man I'm not lazy. \n\nI'm lettin' my drug kick in \nIt controls my mouth and I begin \nTo just let it flow, let my concepts go \nMy posse's to the side yellin', Go Vanilla Go! \n\nSmooth 'cause that's the way I will be \nAnd if you don't give a damn, then \nWhy you starin' at me \nSo get off 'cause I control the stage \nThere's no dissin' allowed \nI'm in my own phase \nThe girlies sa y they love me and that is ok \nAnd I can dance better than any kid n' play \n\nStage 2 -- Yea the one ya' wanna listen to \nIt's off my head so let the beat play through \nSo I can funk it up and make it sound good \n1-2-3 Yo -- Knock on some wood \nFor good luck, I like my rhymes atrocious \nSupercalafragilisticexpialidocious \nI'm an effect and that you can bet \nI can take a fly girl and make her wet. \n\nI'm like Samson -- Samson to Delilah \nThere's no denyin', You can try to hang \nBut you'll keep tryin' to get my style \nOver and over, practice makes perfect \nBut not if you're a loafer. \n\nYou'll get nowhere, no place, no time, no girls \nSoon -- Oh my God, homebody, you probably eat \nSpaghetti with a spoon! Come on and say it! \n\nVIP. Vanilla Ice yep, yep, I'm comin' hard like a rhino \nIntoxicating so you stagger like a wino \nSo punks stop trying and girl stop cryin' \nVanilla Ice is sellin' and you people are buyin' \n'Cause why the freaks are jockin' like Crazy Glue \nMovin' and groovin' trying to sing along \nAll through the ghetto groovin' this here song \nNow you're amazed by the VIP posse. \n\nSteppin' so hard like a German Nazi \nStartled by the bases hittin' ground \nThere's no trippin' on mine, I'm just gettin' down \nSparkamatic, I'm hangin' tight like a fanatic \nYou trapped me once and I thought that \nYou might have it \nSo step down and lend me your ear \n'89 in my time! You, '90 is my year. \n\nYou're weakenin' fast, YO! and I can tell it \nYour body's gettin' hot, so, so I can smell it \nSo don't be mad and don't be sad \n'Cause the lyrics belong to ICE, You can call me Dad \nYou're pitchin' a fit, so step back and endure \nLet the witch doctor, Ice, do the dance to cure \nSo come up close and don't be square \nYou wanna battle me -- Anytime, anywhere \n\nYou thought that I was weak, Boy, you're dead wrong \nSo come on, everybody and sing this song \n\nSay -- Play that funky music Say, go white boy, go white boy go \nplay that funky music Go white boy, go white boy, go \nLay down and boogie and play that funky music till you die. \n\nPlay that funky music Come on, Come on, let me hear \nPlay that funky music white boy you say it, say it \nPlay that funky music A little louder now \nPlay that funky music, white boy Come on, Come on, Come on \nPlay that funky music".encode('ascii')
key = os.urandom(16)
nonce = 0
nonce = nonce.to_bytes(8, 'little')

def encrypt():
    return ctr.ctr_mode(pt, key, nonce)

def decrypt(ct, key, offset=0):
    return ctr.ctr_mode(ct, key, nonce)

def edit(ct, key, offset, new_text):
    keystream = ctr.ctr_keystream(key, nonce, len(ct)//16+len(new_text)//16)
    new_ct = cbc.xor(new_text, keystream[offset:offset+len(new_text)])

    ct = ct[:offset] + new_ct + ct[offset+len(new_ct):]
    return ct
```

Our edit function replaces the ciphertext at ``offset`` with the new encrypted ciphertext. We have:

```
new_ct = keystream ^ new_plaintext
```

If the new plaintext is all zeros, the ciphertext will just be the keystream. Once we have the keystream, we can XOR it with the original ciphertext to recover the original plaintext!

```
original_ct = encrypt()

null_block = ('\x00'*len(ct)).encode('ascii')
ctr_stream = edit(ct, key, 0, null_block)
# ctr_stream = \x00 ^ keystream
# ctr_stream = keystream
pt = cbc.xor(ctr_stream, ct)
```

If an attacker can edit the plaintext in CTR mode, they can break it!


## 26. CTR bitflipping
>  There are people in the world that believe that CTR resists bit flipping attacks of the kind to which CBC mode is susceptible.
> 
> Re-implement the CBC bitflipping exercise from earlier to use CTR mode instead of CBC mode. Inject an "admin=true" token. 

For this challenge, we're re-implementing our bitflipping attack from [challenge 16](https://cryptopals.com/sets/2/challenges/16).

The attack works the same way as the CBC bitflipping attack, except instead of modifying the previous block we change the block we want to affect directly. Since CTR ciphertexts are directly XOR'd with the keystream-- instead of being XOR'd with the previous block like CBC mode.


Now let's write a function to bitflip admin in! Our crafted plaintext is:

```
b'comment1%3Dcooking%2520MCs%3Buserdata%3DAAAAAAAAAAAAAAAAAAAA%3Bcomment2%3D%2520like%2520a%2520pound%2520of%2520bacon'
```

We know that we'll have a block consisting of just ``AAAAA``. This is what we're interested in changing and it's in the 4th block (bytes 48-64). We want to cancel out this block, and insert ``;admin=true`` into the ciphertext. Here's how we can:

```
string = "AAAAAAAAAAAAAAAA"
goal_string = ";admin=true;AAAA"

goal_string = string ^ string ^ goal_string
```

When XOR'd, the original string will cancel out leaving us with just the goal string!

```
def bitflip_decrypt(ct):
    global string
    goal_string = (";admin=true;AAAA").encode('ascii') # block length (16 bytes)
    string = string.encode('ascii')
    xord = cbc.xor(goal_string, string)

    modified_ct = ct[:48] + cbc.xor(xord, ct[48:64]) + ct[64:]
    pt = ctr.ctr_mode(modified_ct, key, nonce)

    return ";admin=true;" in str(pt)
```

Our ``bitflip_encrypt`` function is the same as [challenge 16](https://hmcguinn.com/posts/cryptopals-set-2/) except using CTR mode.
```
ct = bitflip_encrypt(string)
detected = bitflip_decrypt(ct)
# True
```

CTR mode, just like CBC mode, has no ciphertext integrity and is vulnerable to the same bitflipping attacks given an attacker knows some of the plaintext.


## 27. Recover the key from CBC with IV=Key
> Take your code from the CBC exercise and modify it so that it repurposes the key for CBC encryption as the IV.
> 
> Applications sometimes use the key as an IV on the auspices that both the sender and the receiver have to know the key already, and can save some space by using it as both a key and an IV.
> 
> Using the key as an IV is insecure; an attacker that can modify ciphertext in flight can get the receiver to decrypt a value that will reveal the key.
> 
> The CBC code from exercise 16 encrypts a URL string. Verify each byte of the plaintext for ASCII compliance (ie, look for high-ASCII values). Noncompliant messages should raise an exception or return an error that includes the decrypted plaintext (this happens all the time in real systems, for what it's worth).
> 
> Use your code to encrypt a message that is at least 3 blocks long:
> 
> ```
> AES-CBC(P_1, P_2, P_3) -> C_1, C_2, C_3
> ```
> 
> Modify the message (you are now the attacker):
> 
> ```
> C_1, C_2, C_3 -> C_1, 0, C_1
> ```
> 
> Decrypt the message (you are now the receiver) and raise the appropriate error if high-ASCII is found.
> 
> As the attacker, recovering the plaintext from the error, extract the key:
> 
> ```
> P'_1 XOR P'_3
> ```

For this challenge, we decrypt a CBC encrypted ciphertext raising an error that returns the decrypted plaintext if a non-ASCII value is detected.

```
def decrypt(ct):
    pt = no_pad_cbc_decrypt(ct, key, iv)

    try:
        pt.decode('ascii')
    except Exception:
        return True, pt
    return False, pt
```

We encrypt using the same function as [challenge 16](https://cryptopals.com/sets/2/challenges/16), but with the IV set to the key.

```
ct = encrypt(string)
```

By modifying the ciphertext so the second block becomes all 0's, we can recover the key!

```
modified_ct = ct[:16] + bytearray(('\x00'*16).encode('ascii')) + ct[:16]  
```

If we get an error (we should!) we can recover the key:

```
error, pt = decrypt(ct)
if error:
    recovered_key = xor(pt[32:48], pt[:16])
    print(recovered_key) # original key, same as IV
```

The reason this works: 

```
p_1 = IV ^ D(c_1)  
p_3 = 0 ^ D(c_1) = D(c_1)

p_1 ^ p_3 = D(c_1) ^ D(c_1) ^ IV 
          = IV
```

The IV is the same as the key which we recover with the above! 




## 28. Implement a SHA-1 keyed MAC
> Find a SHA-1 implementation in the language you code in.
>
> Don't cheat. It won't work.
>
> Do not use the SHA-1 implementation your language already provides (for instance, don't use the "Digest" library in Ruby, or call OpenSSL; in Ruby, you'd want a pure-Ruby SHA-1).
> 
> Write a function to authenticate a message under a secret key by using a secret-prefix MAC, which is simply:
> 
> ```
> SHA1(key || message)
> ```
> 
> Verify that you cannot tamper with the message without breaking the MAC you've produced, and that you can't produce a new MAC without knowing the secret key. 

This challenge was luckily simple enough! Once you find a SHA-1 implementation online (or dive into the [RFC](https://www.rfc-editor.org/rfc/rfc3174) and roll your own crypto) we implement:

```
def prefix_mac(message):
    return sha1(key + message)
```
## 29. Break a SHA-1 keyed MAC using length extension
> Secret-prefix SHA-1 MACs are trivially breakable.
> 
> The attack on secret-prefix SHA1 relies on the fact that you can take the ouput of SHA-1 and use it as a new starting point for SHA-1, thus taking an arbitrary SHA-1 hash and "feeding it more data".
> 
> Since the key precedes the data in secret-prefix, any additional data you feed the SHA-1 hash in this fashion will appear to have been hashed with the secret key.
> 
> To carry out the attack, you'll need to account for the fact that SHA-1 is "padded" with the bit-length of the message; your forged message will need to include that padding. We call this "glue padding". The final message you actually forge will be:
> 
> ```
> SHA1(key || original-message || glue-padding || new-message)
> ```
> 
> (where the final padding on the whole constructed message is implied)
> 
> Note that to generate the glue padding, you'll need to know the original bit length of the message; the message itself is known to the attacker, but the secret key isn't, so you'll need to guess at it.
> 
> This sounds more complicated than it is in practice.
> 
> To implement the attack, first write the function that computes the MD padding of an arbitrary message and verify that you're generating the same padding that your SHA-1 implementation is using. This should take you 5-10 minutes.
> 
> Now, take the SHA-1 secret-prefix MAC of the message you want to forge --- this is just a SHA-1 hash --- and break it into 32 bit SHA-1 registers (SHA-1 calls them "a", "b", "c", &c).
> 
> Modify your SHA-1 implementation so that callers can pass in new values for "a", "b", "c" &c (they normally start at magic numbers). With the registers "fixated", hash the additional data you want to forge.
> 
> Using this attack, generate a secret-prefix MAC under a secret key (choose a random word from /usr/share/dict/words or something) of the string:
> 
> ```
> "comment1=cooking%20MCs;userdata=foo;comment2=%20like%20a%20pound%20of%20bacon"
> ```
> 
> Forge a variant of this message that ends with ";admin=true". 
## 30. Break an MD4 keyed MAC using length extension
> Second verse, same as the first, but use MD4 instead of SHA-1. Having done this attack once against SHA-1, the MD4 variant should take much less time; mostly just the time you'll spend Googling for an implementation of MD4. 
## 31. Implement and break HMAC-SHA1 with an artificial timing leak
> The psuedocode on Wikipedia should be enough. HMAC is very easy.
> 
> Using the web framework of your choosing (Sinatra, web.py, whatever), write a tiny application that has a URL that takes a "file" argument and a "signature" argument, like so:
> 
> ```
> http://localhost:9000/test?file=foo&signature=46b4ec586117154dacd49d664e5d63fdc88efb51
> ```
> 
> Have the server generate an HMAC key, and then verify that the "signature" on incoming requests is valid for "file", using the "==" operator to compare the valid MAC for a file with the "signature" parameter (in other words, verify the HMAC the way any normal programmer would verify it).
> 
> Write a function, call it "insecure_compare", that implements the == operation by doing byte-at-a-time comparisons with early exit (ie, return false at the first non-matching byte).
> 
> In the loop for "insecure_compare", add a 50ms sleep (sleep 50ms after each byte).
> 
> Use your "insecure_compare" function to verify the HMACs on incoming requests, and test that the whole contraption works. Return a 500 if the MAC is invalid, and a 200 if it's OK.
> 
> Using the timing leak in this application, write a program that discovers the valid MAC for any file. 

This challenge doesn't take advantage of a flaw in SHA1, but instead attacks the timing side channel that checks if the HMAC is correct. Getting this challenge exactly right is a little finicky as the timing over the network is sensitive!

We can start by implementing [HMAC-SHA1](https://en.wikipedia.org/wiki/HMAC#Implementation):
```
def hmac(key, msg, blockSize=64, output=20):
    block_sized_key = computeBlockSizedKey(key, blockSize)

    o_key_pad = cbc.xor(block_sized_key, bytearray(('\x5c' * blockSize).encode('ascii')))
    i_key_pad = cbc.xor(block_sized_key, bytearray(('\x36' * blockSize).encode('ascii')))

    return sha.sha1(o_key_pad + sha.sha1(i_key_pad + msg))

def computeBlockSizedKey(key, blockSize=64):
    if len(key) > blockSize:
        return sha.sha1(key)
    elif len(key) < blockSize:
        return key + (b'\x00'*(64-len(key)))
    else:
        return key
```

Next, let's setup our web server. I chose to use [web.py](https://webpy.org/) since it looked simple and was suggested in the challenge writeup.

It all fits into a single file which is nice!

```
import sha_hmac
import os

urls = ('/(.*)')
app = web.application(urls, globals())

key = os.urandom(16)

def insecure_compare(valid, sig):
    if len(valid) != len(sig):
        return False

    for i in range(len(hmac)):
        if hmac[i] != sig[i]:
            return False
        time.sleep(.05)

    return True


class hello:
    def GET(self, name):
        data = web.input()

        if 'file' not in data:
            data['file'] = ""
        if 'signature' not in data:
            data['signature'] = ""
        file = data['file']
        sig = data['signature']

        hmac = sha_hmac.hmac(key, file.encode('ascii')).hex()

        if len(hmac) != len(sig):
            return web.InternalError('invalid len')

        for i in range(len(hmac)):
            if hmac[i] != sig[i]:
                return web.InternalError('invalid mac')
            time.sleep(.05)
        return "200"

if __name__ == "__main__":
    app.run()
```

When you navigate to ``http://localhost:8080/`` you'll get a 500 error unless you pass a file and a valid signature for that file.

The strategy to recover the signature is straightforward. After, we'll see how we can improve it for efficiency and to be resilient to network noise.
1. Guess the first hex digit of the HMAC signature
    - Pad to signature length (40 characters)
2. Start timer
3. Send GET request with ``guessed_signature`` and ``file`` as URL parameters.
4. End timer. 
5. Record the taken to process request. 
6. Repeat guessing for each digit, if one digit takes longer to receive a response, that's likely the right character!
7. Repeat until you have the full HMAC signature.

When I first tried this, it got about half-way through before getting a character wrong. Naively, I figured we could just try submitting each guess a few times, taking the average. Unfortunately, that didn't work for me. We'll have to come up with something a little more clever!

Taking the average is a good idea, but is still influenced by outliers. If one request takes way too long, it'll skew the average even if the rest were normal. This could overwhelm the valid guess that did take slightly longer but had a lower average due to the outlier! We could get fancy with statistics, but I decided to try each guess 5 times and throw out the slowest and fastest time. It looks like this:

```
def average_times(file, signature):
    tries = 5
    arr = []
    for i in range(tries):
        _, time_taken = try_signature(file, signature)
        arr.append(time_taken)
    arr.sort()

    return sum(arr[1:-1]) / tries - 2
```

Using that, we build this function to recover the HMAC signature!

```
def attack_hmac(file):
    forged_sig = ""
    unknown_len = 39

    hex_digits = ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "a", "b", "c", "d", "e", "f", "0"]
    average = None
    while len(forged_sig) < 40:
        for digit in hex_digits:
            temp_sig = forged_sig + digit + "0"*unknown_len
            average_time = average_times(file, temp_sig)

            # 30 milliseconds longer
            if average and average_time > (average+30000000):
                forged_sig += digit
                unknown_len -= 1
                break
            average = average_time

        average = None
    return forged_sig
```

You'll see the character ``0`` is in our digits list twice. Since we need an average to compare against, we need one extra evaluation for each guess.

This approach works and is fairly straightforward, but does take a while! It took about 30 minutes to recover the HMAC signature.
## 32. Break HMAC-SHA1 with a slightly less artificial timing leak
> Reduce the sleep in your "insecure_compare" until your previous solution breaks. (Try 5ms to start.)

> Now break it again. 
