---
layout: post
title: "Padding Oracle Attack or the Virtues of a Glomar response"
date: 2018-05-14 20:33
disqus: y
categories: cryptography
---
<script type="text/javascript"
    src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

While working on the excellent [CryptoPals](https://cryptopals.com/) challenges, I came across the [Padding Oracle Attack](https://en.wikipedia.org/wiki/Padding_oracle_attack).

The particular version of the attack I was working through, was on [CBC](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#CBC) mode Decryption when used with [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) symmetric key cryptography. The attack has nothing to do with AES and everything to do with this block cipher mode of operation.

Various forms of this attack have been discovered as late as 2016!

It highlighted the importance of saying as little as possible in the case of failure. You are leaking information when you say no and additionally give an explicit reason for it. In response, the attacker can calibrate the next request to match your expectations.

## Background

_A whirlwind explanation of AES with CBC mode follows_

AES is a symmetric block cipher in which you can encrypt/decrypt a "block" i.e. a fixed length of bytes using a shared secret/key.

A mode of operation like CBC is a way to securely handle data larger than the block size of the block cipher.

In CBC mode, a block of plaintext is XOR'ed with the previous ciphertext block and then encrypted. While decrypting, we reverse the process; we first decrypt the block and then XOR it with the previous ciphertext block to get the plain text.

It is better explained in the following way:

### Encryption:

![CBC Encryption](/images/CBC_encryption.svg)

Mathematically, this can be written as:

$$C_i = E_K(P_i \oplus C_{i - 1}),$$

$$C_0 = IV$$

Where \\( P_i \\) is the \\( i \\)'th plain text block, \\( C_i \\) is the \\( i \\)'th encrypted block, \\( IV \\) is a randomly chosen block which is called Initialisation Vector and \\( E_K \\) is the encryption algorithm.

### Decryption:

![CBC Decryption](/images/CBC_decryption.svg)

Mathematically, this can be written as:

$$P_i = D_K(C_i) \oplus C_{i - 1},$$

$$C_0 = IV$$

Where \\( P_i \\) is the \\( i \\)'th plain text block, \\( C_i \\) is the \\( i \\)'th encrypted block, \\( IV \\) is the same Initialisation Vector as the one used in the encryption process and \\( D_K \\) is the decryption algorithm.

### Padding

While using a block cipher mode, we require that the plaintext is of a size which is a multiple of the block length. To achieve this, we pad the plain text with bytes at the end of the message. A common way is [PKCS7](https://en.wikipedia.org/wiki/Padding_(cryptography)#PKCS7), in which the value of each byte added is equal to the number of bytes added, such as:

{% highlight bash %}
01
02 02
03 03 03
04 04 04 04
05 05 05 05 05
06 06 06 06 06 06
etc.
{% endhighlight %}

If the text is already a multiple of the block length, we still pad it with 16 16's. This enables us to remove ambiguity.

Therefore, the padding is constructed in such a way that when the text is decrypted, we check the value of the last byte, make sure that that many preceding bytes have the same value and then discard them.

## The Attack

For us, the interesting thing to notice is that we can influence a decrypted block of plain text, by manipulating the previous ciphertext block. This doesn't mean that we can break the said previous ciphertext block.

A padding oracle is a function when given ciphertext, decrypts it and checks if the padding on the decrypted text is valid or not. This function is useful because if padding isn't correct, the decrypted text is certainly corrupted. We don't need to leak this extra information about the padding to the user. But sometimes it does happen.

Padding oracles have been found in many web frameworks including Ruby on Rails, Java ServerFaces and ASP.NET. An example of a padding oracle in Java is the exception `javax.crypto.BadPaddingException` with the message `Given final block not
properly padded`. Some more padding oracles can be found in [this paper](https://www.usenix.org/legacy/event/woot10/tech/full_papers/Rizzo.pdf).

Let's work an actual example before we formalise what we are doing.

Let's use AES as our block cipher algorithm with a block and key size of 128 bits or 16 bytes.

- Let the shared key be `YELLOW SUBMARINE` which when represented as bytes is `[89, 69, 76, 76, 79, 87, 32, 83, 85, 66, 77, 65, 82, 73, 78, 69]`. Notice that the key is 16 bytes. In real life, please make sure that your key is generated using a cryptographically secure random number generator.

- Let for the sake of this example Initialisation Vector be `[48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48]`. Again, in reality, this should be generated using a cryptographically secure random number generator.

- Let the plain text be `Here comes the sun (doo doo doo doo).` which in bytes is `[72, 101, 114, 101, 32, 99, 111, 109, 101, 115, 32, 116, 104, 101, 32, 115, 117, 110, 32, 40, 100, 111, 111, 32, 100, 111, 111, 32, 100, 111, 111, 32, 100, 111, 111, 41, 46]`. The plain text is of length 37 but we require it to be a multiple of 16. So we add a padding of 11 to it. So the plain text is now `[72, 101, 114, 101, 32, 99, 111, 109, 101, 115, 32, 116, 104, 101, 32, 115, 117, 110, 32, 40, 100, 111, 111, 32, 100, 111, 111, 32, 100, 111, 111, 32, 100, 111, 111, 41, 46, 11, 11, 11, 11, 11, 11, 11, 11, 11, 11, 11]`.

- When encrypted using the given key and I.V., we get a cipher text of `[210, 182, 226, 53, 98, 183, 56, 39, 223, 19, 26, 253, 41, 22, 252, 111, 224, 230, 163, 75, 150, 93, 81, 242, 114, 80, 192, 153, 192, 19, 97, 251, 224, 107, 139, 11, 85, 141, 183, 227, 159, 180, 155, 249, 196, 225, 248, 13]`.

- We will focus on the 1st and the 2nd blocks of the ciphertext. This is available to the attacker. Our intention will be to get the last byte of the 2nd block of the corresponding plain text (which is `32`). We will do this by modifying the last byte of the 1st block. The value of the last byte of the 1st block is `111`. Let modify the last byte of the 1st encrypted block be its original value xor'ed with 1 and with 0. So, \\( 111 \oplus 1 \oplus 0 = 110\\).

- Let's just consider the modified first ciphertext block and the original second ciphertext block as a complete message and then send it to the padding oracle. 2nd block when AES decrypted is `[167, 216, 194, 29, 6, 216, 87, 7, 187, 124, 117, 221, 77, 121, 147, 79]`. So in CBC mode, the last byte will be \\( 79 \oplus 110 = 33\\). So the oracle will see 33 as the last byte but it isn't a valid padding. So it will return a false.

- Lets change the last byte of the 1st block of cipher text to be \\( 111 \oplus 1 \oplus 1 = 111 \\).

- Now the padding oracle will see \\( 79 \oplus 111 = 32 \\) as the last byte which is again not a valid padding and so it will return false.

- Essentially we keep on incrementing \\( x \\) in \\( 111 \oplus 1 \oplus x \\) till the padding oracle returns true.

- We find for \\( x = 32\\), \\( 111 \oplus 1 \oplus 32 = 78\\) and \\( 79 \oplus 78 = 1\\). So for this value of x, the padding oracle will return true as `1` is a valid padding ending. Notice that x has the same value as the last byte of the 2nd plaintext block for which the oracle returned true. This is not an accident! (This is almost always true. There is a corner case discussed later.)

Let's try to understand what is happening.

Given the ciphertext, \\( [C_0, C_1, \cdots, C_N] \\), lets say we want to recover the plaintext, \\( P_1 \\) from ciphertext \\( C_1 \\). We will achieve this by guessing the last byte of the message, then the second last byte and so on.

Let's focus on the message comprising of just \\( [C_0, C_1] \\).

We can construct \\( C'_0 \\) such that it is same as \\( C_0 \\) but its last byte's value is value of last byte of \\( C_0 \\) xored with 1 and xored with \\( x \\). Or \\( C'_0 = \vert c_1 \vert \cdots \vert c_m \oplus x \oplus 1 \vert \\), where \\( c_1 \\) is the 1st byte of \\( C_0 \\) etc., \\( m \\) is number of bytes in the block and \\( x \\) is a byte whose value is 0. We then send the message to \\( [{C'}_0, C_1] \\) to the padding oracle function which will return a true/false result. Focussing on the \\( P_1 \\) and \\( {P'}_1 \\),

$$P_1 = D_K(C_1) \oplus C_0$$

$${P'}_1 = D_K(C_1) \oplus {C'}_0 $$

Substituting \\( C' \\) and only focusing on the last byte, we get,

$${p'}_m = p_m \oplus x \oplus 1$$

If the function says that this is not a valid padding, we simply increment \\( x \\).

We keep on doing this, till the function returns a true value. At worst, we've tried the oracle 256 times at worst (all possibilities for a byte) and we will get at least one true value.

An important property of the xor function is that \\( x \oplus x = 0 \\). So if \\( p_m = x \\), then the value of \\( {p'}_m \\) will be \\( 1 \\) which is a valid ending padding value and the padding oracle would return true for a valid padding for \\( [C_0, C_1] \\) as such!! So there is at least one value of \\( x \\) for which the padding oracle would return true.

There is a corner case that the padding oracle can return true for more than one value of \\( x \\). See [Stack Overflow](https://crypto.stackexchange.com/questions/40800/is-the-padding-oracle-attack-deterministic) for details on how to break this.

By construction, we have just guessed the value of the last byte of \\( C_1 \\).

We can now work backwards to construct a new \\( C'_0 \\)  to target a padding of `02 02` in the end state, then `03 03 03` etc. to decipher the whole block \\( C_1 \\).

It took us only \\( m \cdot 256 \\) guesses at worst to decipher a ciphertext block of size \\( m \\) without knowing the key. If we were to break the key by brute force, it would take \\( 256 ^ m \\) guesses. We didn't need to break the secret key to get to the plain text.

Why did this happen?

It happened because the attacker was able to deduce what is a padding failure and what is not. It was also able to bombard the padding oracle with many requests.

In case of a data encryption/decryption error, we should be returning a generic error and not a specific error.

Additionally, we should not be decrypting any data we can't verify first, in the sense did the sender mean to send this exact message or not. So we should be sending a [MAC](https://en.wikipedia.org/wiki/Message_authentication_code) along with the ciphertext and we should only be trying to decrypt after validating the MAC.

Till next time!
