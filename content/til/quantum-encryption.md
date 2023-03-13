---
title: "Revocable encryption through quantum mechanics"
date: 2023-03-12T18:25:49-07:00
draft: false
---

Today I learned that revocable encryption exists!

## A cryptographic dead person's switch
Imagine 1) you want to encrypt some data so that it is released with no interaction from you in the future. Also, 2) you want to be able to cancel the release of the data at any point before the release. Can you do it?

I built a [version of this](https://github.com/hmcguinn/dead-persons-switch) for my [Privacy Preserving Technologies](https://www.cs.unc.edu/~saba/priv_class/index.html) class with the caveat that you have to trust some threshold of servers to be honest. While a fun project, having to trust a threshold of the servers limits how useful it is. We want cryptographic guarantees for both of the above properties.

These two goals are somewhat at odds with each other. If you want to ensure data is released you could give it to a friend and ask them to release it. Or, if you want to get fancy you could use a [verifiable delay function](https://eprint.iacr.org/2018/601.pdf) to build a [time-lock puzzle](https://people.csail.mit.edu/rivest/pubs/RSW96.pdf) and put the ciphertext on a blockchain. These both allow for interaction-less release in the future. But, revocation is a problem. In the first case, you need to trust your friend not to keep a copy of the data (or peek at it!). In the second, it's impossible-- once it's on a blockchain it's there for good. Cancelling the release is similarly easy, you could just wait to publish the data until you'd like it available. But then it won't be available should your server shut down.

We need some way to both 1) ensure the interaction-less release of data, and, 2) let the user guaranteedly prevent or delay the release of the data. It turns out that thanks to quantum mechanics, you can do just that!

## Revocable quantum time-release encryption
Dominique Unruh introduces the idea of [revocable quantum time-release encryption](https://eprint.iacr.org/2013/606.pdf)! While I don't have enough of a physics background to understand the full details of the protocol, intuitively it makes sense how it can exist.

There are a number of papers that build on this idea, including [quantum encryption with certified deletion](https://arxiv.org/pdf/1910.03551.pdf) and [revocable cryptography from learning with errors](https://eprint.iacr.org/2023/325.pdf).