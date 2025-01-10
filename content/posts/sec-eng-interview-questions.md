---
title: "(Better) Security Engineer Interview Questions"
date: 2024-12-02T20:12:34-05:00
draft: true
---

(Un)-fortunately, there's no standardized Leetcode-esque interview process for security engineers [^1]. There are a number of online resources for security engineer interview questions, but I found them to be too high level compared to interviews I've been in (explain encoding vs encryption vs hashing). A lot were missing answers. This is my attempt to collate some interesting questions + answers in one place!
 


# Existing resources

These are some other resources I've found helpful:

- [Security Engineering at Google: My Interview Study Notes](https://github.com/gracenolan/Notes/blob/master/interview-study-notes-for-security-engineering.md) by Grace Nolan
- [Web AppSec Interview Questions](https://tib3rius.com/interview-questions) by Tib3rius (*has answers!*) 
- [Application Security Engineer Interview Questions](https://ishaqmohammed.me/posts/application-security-engineer-interview-questions/) by Ishaq Mohammed
- [Cybersecurity Interview Questions](https://danielmiessler.com/p/infosec-interview-questions) by Daniel Miessler
- [Infosec core competencies](https://www.netmeister.org/blog/infosec-competencies.html) by Netmeister

# Some questions

## Cryptography 

##### What's the difference between symmetric and asymmetric encryption?

Symmetric encryption is like having a shared safe combination - both Alice and Bob need to know the exact same secret to encrypt and decrypt messages. Simple, but there's a catch: how do they securely share that secret in the first place?

Asymmetric encryption solves this with public/private keypairs. Alice can encrypt with Bob's public key (which can be shared freely), and _only_ Bob can decrypt it with his matching private key. When Bob wants to reply, he uses Alice's public key, and only she can read it with her private key.

Why pick one over the other? Symmetric encryption is faster but requires that shared secret problem to be solved first. Asymmetric is perfect for one-way communication (like sending someone an encrypted email) since you only need their public key. This is why most real-world systems (like TLS) use both - asymmetric to exchange a symmetric key, then symmetric for the actual data.
##### Does TLS use asymmetric or symmetric encryption?

Both! TLS starts with an asymmetric handshake to securely exchange keys (avoiding the **hard** problem of how to share a secret), then switches to symmetric encryption for the actual data because it's _way_ faster.

When a client connects, it tells the server what cipher suites it supports. These are strings like `TLS_AES_256_GCM_SHA384` that specify both the asymmetric part (like ECDHE-RSA for key exchange) and symmetric part (like AES-256-GCM for bulk encryption).

A server might support multiple suites, but modern TLS strongly prefers AEAD ciphers like AES-GCM. If you see older ciphers like CBC in your server logs, something's wrong!

##### Explain envelope encryption

Data at rest needs encryption, but encrypting _everything_ with one key is asking for trouble. Instead, use envelope encryption: generate data encryption keys (DEKs) for your actual data, then wrap those DEKs with a key encryption key (KEK).

This is clever for a few reasons: it limits how much data gets encrypted with any single key (important for key wear-out), makes key rotation way easier (just re-wrap the DEKs with a new KEK), and contains the blast radius if a key is compromised.

[Tink](https://developers.google.com/tink) and the [AWS Encryption SDK](https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/choose-keyring.html) are two libraries that support this pattern.  [Slack wrote a great piece on their engineering blog](https://slack.engineering/engineering-dive-into-slack-enterprise-key-management/) about building their Enterprise Key Management system on these principles.

##### What are the differences between AES-GCM and AES-CBC? Why might you prefer one over the other?
GCM gives you encryption _and_ integrity in one package (it's an AEAD cipher) while CBC only handles encryption. You need to add a MAC to CBC yourself, and getting that wrong is a common source of vulnerabilities. Remember [Lucky 13](https://www.imperialviolet.org/2013/02/04/luckythirteen.html)?

IV reuse is interesting here - CBC degrades somewhat gracefully while GCM [completely falls apart](https://hmcguinn.com/posts/cryptopals-set-3/). If IV uniqueness is hard to guarantee in your system, look at [GCM-SIV](https://en.wikipedia.org/wiki/AES-GCM-SIV). It's slightly slower but way more forgiving.

GCM's parallelization seems nice until you realize it breaks integrity guarantees if you modify blocks independently. Fun fact: this has bitten real implementations that tried to be "clever" with random access.
##### A developer signed a cookie with H(k | m) - is this secure?

Nope! This falls to a [length extension attack](https://en.wikipedia.org/wiki/Length_extension_attack). If you know H(k | m), you can actually continue hashing from there without knowing k. The attacker reconstructs the hash's internal state and can append whatever they want to the message. 

This bites a lot of hash functions - SHA-1, SHA-2, MD5 all use [Merkle-Damgård construction](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) which makes them vulnerable. SHA-3 dodges this by using a sponge construction. Interestingly, truncated variants like SHA-512/256 and SHA-384 are also safe since attackers can't reconstruct the full internal state from the truncated output.

Want to do this right? Use [HMAC](https://en.wikipedia.org/wiki/HMAC). It's specifically designed to avoid these pitfalls and it's available in every major crypto library. H(m | k) is not vulnerable, but it's still better to just use HMAC.

##### Explain certificate transparency (CT) logs

[This](https://emilymstark.com/2020/07/20/certificate-transparency-a-birds-eye-view.html) post by Emily Stark is a great overview of CT logs. The cliff notes:

Certificate transparency (CT) logs are an auditable record of issued TLS certificates. CT logs enforce transparency for CAs and create a record of what certificates are issued for a domain. CT logs have [caught CA compromises before](https://en.wikipedia.org/wiki/DigiNotar#Issuance_of_fraudulent_certificates)!

Before issuing a certificate, certificate authorities (CA) will submit a precertificate with a poisoned field indicating it should not be used to the CT log. A signed certificate timestamp gets returned to the CA, who can then issue the certificate with the SCT attached. 

[Safari and Chrome both require](https://developer.mozilla.org/en-US/docs/Web/Security/Certificate_Transparency#browser_requirements) that certificates trusted by the browser be recorded in a CT log. [Firefox does not](https://bugzilla.mozilla.org/show_bug.cgi?id=1281469). 

Under the hood, CT logs use a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) as an append-only ledger. Sometimes, [bit-flips happen](https://groups.google.com/a/chromium.org/g/ct-policy/c/S17_j-WJ6dI) which leads to the whole CT log needing to be thrown out and replaced. 

A side-effect of certificates being recorded in CT logs is that infroamtion about subdomains becomes public. You can use [crt.sh](https://crt.sh) to search for subdomains. 

##### What's the difference between hashing and signing?

Anyone can compute the hash of something. Hashes are collision resistant, meaning:

- You can't find two different inputs that hash to the same output (collision resistance)
- Given a hash output, you can't find an input that produces it (preimage resistance)
- Tiny changes to the input produces a completely different hash (avalanche effect)

Checking a message's hash verifies its integrity -- it proves the message hasn't been tampered with. But since anyone can compute a new hash for a modified message, hashing alone doesn't prove who created or modified it.

A digital signature provides both authenticity (proving who signed it) and integrity (proving it hasn't been modified). Signing requires the private key, while anyone can verify the signature. Modern signature schemes like EdDSA, RSA, and ECDSA sign a hash of the message instead of the raw message. This has a couple benefits:

- Efficiency (hashing reduces large messages to a fixed size)
- Security (prevents mathematical attacks possible with raw signatures)
- Proper integrity checking (even a 1-bit change completely changes the hash and invalidates the signature)

This also means that the security of a signature scheme relies on the hash-- a second input that hashes to the same value would also verify successfully!

## Web security

##### How can you protect against SSRF?

Input validation is the first layer here -- allowlist URL schemes or domains. There are **tons** of ways to bypass input validation so this is not sufficient. TOCTOU DNS rebinding attacks can also happen. [CVE-2022-41924](https://emily.id.au/tailscale) is a great read about a RCE in Tailscale on Windows that exploited DNS rebinding. 

In Node, you can use a [http.Agent to filter requests to private ranges](https://github.com/azu/request-filtering-agent). Network-level protections should also be enabled, disable access to RFC 1918 ranges using iptables.

Cloud metadata endpoints (169.254.169.254) are a juicy target - AWS IMDSv1 was particularly bad since a single SSRF could grab root credentials. Block these at the network level *and* enforce IMDSv2 which fixes this by requiring PUT requests and custom headers. The [hop limit can also be set to a max of 2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html). 

##### What is CORS? What does it protect against?

[CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) (cross-origin resource sharing) is a security mechanism that allows sites to relax the same-origin policy (SOP) and indicate what other origins the browser should allow loading resources from.

CORS protects against cross-origin attacks by requiring servers to explicitly opt-in to allowing other [origins](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#definition_of_an_origin) reading responses. If making a complex cross-origin request (like a PUT or DELETE with custom headers), the browser will send a pre-flight request. Servers specify what headers are allowed through `Access-Control-Allow-*` headers in the pre-flight request response. 

CORS does **not** protect against cross-origin requests from being made, only reading the responses. For state-changing operations, [CSRF tokens](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html#token-based-mitigation) should be used.

##### You're building a markdown editor that lets users edit HTML, how can you protect against XSS?

Markdown parsers are XSS nightmares because they're designed to _allow_ HTML - that's part of the spec! First, decide if you really need raw HTML support. If you don't, use a strict markdown-only parser and strip all HTML tags.

If you _do_ need HTML, there's two main options:
- Use [DOMPurify](https://github.com/cure53/DOMPurify) to sanitize. It handles weird edge cases like nested parsers and DOM clobbering.
- Render the HTML in a sandboxed iframe with `sandbox="allow-scripts"`. This gives you proper origin isolation - any XSS is contained to the sandboxed frame. 

One thing to watch out for is injecting dynamic content after sanitization. If the editor lets users inject variables or templates into the markdown (like `${user.name}`), you need to sanitize at render time, not just at parse time.

## Cloud security
##### You've found an SSRF vulnerability running on a EC2 instance, how can you exploit this?

##### How would you implement security for a Kubernetes cluster deployed on EKS?

##### How would you manage secrets for an application?

## Application security

##### A AWS Secret key was committed to our Github repository, what steps need to be taken?
##### An API system lets you used an admin API key, or a finer-grained one. Developers are lazy and use admin API keys on the client-side. How can you prevent this and nudge developers to update?
##### How can you prevent secrets from getting logged in an application?

This is actually a pretty tricky problem-- secrets have a nasty habit of sneaking into logs through error messages, debug traces, and HTTP traces. In 2018, [Twitter logged plaintext passwords](https://www.bleepingcomputer.com/news/security/twitter-admits-recording-plaintext-passwords-in-internal-logs-just-like-github/).

Getting this right requires a lot of trial and error, and playing Whack-A-Mole as new log sites get added. At a higher level, making sure secrets are scoped to areas where they're needed and not getting passed around all over the application makes a big difference. 

Languages like Python and JavaScript let you override `__str__` or `toJSON` methods on sensitive objects. [secret-value](https://github.com/transcend-io/secret-value) is a Typescript library that does this for you by wrapping secrets in an object. If your application uses a logging library like [Winston](https://github.com/winstonjs/winston) or [Log4j](https://logging.apache.org/log4j/2.x/index.html), you can modify the log stream to redact information.

An area that I'm not particularly familiar with, but is also worth knowing about is crash dumps. [Storm-0558 hacked into US Cabinet emails through a Microsoft crash dump](https://msrc.microsoft.com/blog/2023/09/results-of-major-technical-investigations-for-storm-0558-key-acquisition/) that didn't have the signing keys scrubbed.

Another approach is to inject canary values and alert if they appear in logs. Then, you can go and fix where in your application they're coming from. The [core of Datadog's sensitive data scanner](https://github.com/DataDog/dd-sensitive-data-scanner) library which is designed to detect secrets in logs is open-source on Github.

## Design

##### How would you design login for a web app?
This question covers:
- Password storage (bcrypt), maybe 2FA
- Session revocation (session table, JWT)

My first answer is "don't - instead use oauth with Github/Google", but that makes the question much less interesting!

##### We want to design a system to run user code, how can we sandbox it?

The right approach depends a lot on the use-case, some potential questions to consider:

- Is the system latency sensitive?
- Write/Run ratio for code (REPL vs cron job)
- What is the code doing? Does it need to interact with packages, drivers, or hardware?

A few wrong answers:
- AWS Lambda
	- Lambdas will get re-used between invocations, and an attacker can potentially modify state to affect the next invocation
- Docker container
	- A container is generally not a strong security boundary

Some options:
- VM
	- **Pros:** each VM receives it's own kernel
	- **Cons** large amount of overhead
- **Nsjail**: an open-source sandboxing library 
	- **Pros:** flexible with where it can run
	- **Cons:**
- Firecracker: micro VM 
	- **Pros:** fast
	- **Cons:** needs to be run on bare metal machines, much more expensive if you're using a cloud provider
- gVisor

I'm planning on writing an in-depth dive comparing different sandboxing technologies and who uses what. More to come on that later!



[^1]: Say what you want about Leetcode, but it is easily grindable and possible to get better at with effort. There's a huge mini-industry around interview prep and practice for software engineers that doesn't exist in the same way for security engineers. 

