---
title: "Vim has encryption? 😧"
date: 2025-01-10T06:21:57-08:00
draft: true
---

Vim apparently has built-in encryption features. From the [vim man page](https://linuxcommand.org/lc3_man_pages/vim1.html):

> -x  Use encryption when writing files.  Will prompt for  a crypt key.

The vim docs also call out:

> There is never 100% safety.  The encryption in Vim has not been tested for robustness.

If you've ever struggled to exit vim, don't even think about turning on the encryption -- *"when you reopen the file, Vim will ask for the key; if you enter the wrong key, Vim will 'decrypt' it to gibberish content. DO NOT SAVE such a gibberish buffer, or your data will be corrupted."*

### history of encryption in vim

The original encryption method in vim was `zip` (still the default). It's a stream cipher based on [PKZIP compression](https://en.wikipedia.org/wiki/PKZIP), which is about as secure as it sounds. Zip is vulnerable to [known-plaintext attacks](https://github.com/kimci86/bkcrack) and uses a RNG that isn't [cryptographically secure](https://math.ucr.edu/~mike/zipattacks.pdf).


Vim replaced `zip` with [`blowfish` in 2010](https://github.com/vim/vim/commit/40e6a71c6777242a254f1748766aa0e60764ebb3). This implementation used cipher feedback mode, with the same IV re-used for the first 8 blocks. David Leadbeater wrote an [excellent analysis of these problems](https://dgl.cx/2014/10/vim-blowfish) and helped contributed a patch for blowfish2.

`blowfish2` addressed the IV reuse issues and added a SHA256 MAC on the plaintext. Vim's documentation describes it as a "medium strong" encryption method. 

More recently, vim added support for two new methods based on [libsodium](https://doc.libsodium.org/):
- xchacha20: A newer method that was available but is now deprecated
- xchacha20v2: The current experimental method, using the [ChaCha20-Poly1305](https://en.wikipedia.org/wiki/ChaCha20-Poly1305) AEAD cipher

There's also an [interesting discussion on vim_dev](https://groups.google.com/g/vim_dev/c/EQWqj5OuR58) about using SHA-256 for the KDF from the password. 


### what to use instead
If you need to encrypt text files for vim, you probably should reach for [age](https://github.com/FiloSottile/age) which provides a more modern library and simple interface.
