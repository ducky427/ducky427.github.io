---
layout: post
title: "Padding Oracle Attack or the Virtues of a Glomar response"
date: 2018-05-14 20:33
disqus: y
categories: cryptography
---
<script type="text/javascript"
    src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

While working on the excellent [CryptoPals](https://cryptopals.com/) challenges, I came across the [Padding Oracle Attack](https://en.wikipedia.org/wiki/Padding_oracle_attack).

The particular version of the attack I was working through, was on [CBC](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#CBC) mode Decryption when used with [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) symmetric key cryptography. The attack has nothing to do with AES and everything to do with the block cipher mode of operation.

It highlighted the importance of saying as little as possible in the case of failure. You are leaking information when you say no and give a reason for that! The response can now be calibrated to match you are expecting.

## Background

_A whirlwind explanation of AES with CBC mode follows_

AES is a symmetric block cipher in which you can encrypt/decrypt a "block" i.e. a fixed length of bytes using a shared secret.

A mode of operation like CBC is a way to securely handle data larger than the block size of the block cipher.

In CBC mode, a block of plain text is XOR'ed with the previous ciphertext block and then encrypted. While decrypting we reverse the process, we first decrypt the block and then XOR it with the previous ciphertext block to get the plain text.

It is better explained in the following way:

### Encryption:

![CBC Encryption](/images/CBC_encryption.svg)

Mathematically, this can be written as:

$$C_i = E_K(P_i \oplus C_{i - 1}),$$

$$C_0 = IV$$

Where \\( P_i \\) is the i'th plaintext block, \\( C_i \\) is the i'th encrypted block, \\( IV \\) is a randomly chosen block which is called Initialisation Vector and \\( E_K \\) is the encryption algorithm.

### Decryption:

![CBC Decryption](/images/CBC_decryption.svg)

Mathematically, this can be written as:

$$P_i = D_K(C_i) \oplus C_{i - 1},$$

$$C_0 = IV$$

Where \\( P_i \\) is the i'th plaintext block, \\( C_i \\) is the i'th encrypted block, \\( IV \\) is the same Initialisation Vector as the one used in the encryption process and \\( D_K \\) is the decryption algorithm.

### Padding

While using a block cipher mode, we require that the plain text be of a size which is a multiple of block length. To achieve this, we pad the plain text with bytes at the end of the message. A common way is [PKCS7](https://en.wikipedia.org/wiki/Padding_(cryptography)#PKCS7), in which the value of each byte added is equal to the number of bytes added, such as:

{% highlight bash %}
01
02 02
03 03 03
04 04 04 04
05 05 05 05 05
06 06 06 06 06 06
etc.
{% endhighlight %}


## The Attack

For us the interesting thing to notice is that we can influence a decrypted block of plain text, by manipulating the previous cipher text block. This doesn't mean that we can break the said previous cipher text block.

A padding oracle is a function when given ciphertext, decrypts it and checks if the padding on the decrypted text is valid or not. This function is useful because if padding isn't correct, the decrypted text is certainly corrupted. We don't need to leak this extra information about the padding to the user. But sometimes it does happen!

Back to our attack, given the ciphertext, \\( C_0, C_1, ..., C_N \\), lets say we want to recover the plaintext, \\( P_1 \\). We will achieve this by guessing the last byte of the message, then the second last byte and so on.

Let's focus on the message comprising of just \\( [C_0, C_1] \\).


We can construct \\( C'_1 \\) which is the same as \\( C_1 \\) but \\( {C'}_1^m = C_1^m \oplus 1 \oplus x\\), where \\( m \\) represents the last byte number for the block and \\( x \\) is a byte whose value is 0. We then send the message to \\( [C_0, {C'}_1] \\) to the padding oracle function. So

$$P_1 = D_K(C_1) \oplus C_0$$

$${P'}_1 = D_K(C_1) \oplus {C'}_0 $$

Substituting \\( C' \\), we get,

$${P'}_1 = D_K(C_1) \oplus C_0 \oplus 1 \oplus x$$

$$       = P_1 \oplus 1^m \oplus x$$

If the function says that this is not a valid padding, we simply increment \\( x \\).

We keep on doing this, till the function returns a true value. At worst, we've tried the oracle 256 times (all possibilites for a byte).

What have we learnt from this?

Notice that if \\( x \\) has the same value of as the last bit of \\( D_K(C_1) \\), then

Till next time!

