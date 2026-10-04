# Many-Time Pad Attack – Decrypting Message 13

## Overview

The goal of this exercise was to decrypt **Message 13** from a collection of ciphertexts encrypted using a stream cipher.

The vulnerability was that the **same keystream was reused for multiple messages**. A stream cipher should never reuse the same keystream because XORing two ciphertexts encrypted with the same key eliminates the key and exposes information about the plaintexts.

Using this weakness, I gradually recovered the keystream and used it to decrypt Message 13.

---

## 1. Convert the Ciphertexts from Hex to Bytes

The ciphertexts were provided as hexadecimal strings. I first converted them into bytes:

```python
sb = [bytes.fromhex(c) for c in ciphers]
```

This was necessary because XOR operations work on numerical byte values rather than the hexadecimal text representation.

The same conversion was performed on Message 13. The ciphertext itself was not modified; only its representation was changed.

---

## 2. Identify the Keystream-Reuse Vulnerability

For a stream cipher:

```text
Ciphertext = Plaintext XOR Key
C = P XOR K
```

If the same keystream is reused:

```text
C1 = P1 XOR K
C2 = P2 XOR K
```

XORing the ciphertexts gives:

```text
C1 XOR C2
= P1 XOR K XOR P2 XOR K
```

Since:

```text
K XOR K = 0
```

we obtain:

```text
C1 XOR C2 = P1 XOR P2
```

The keystream disappears from the equation.

This was the fundamental vulnerability behind the attack.

---

## 3. XOR the Ciphertexts Against Each Other

Instead of manually XORing every pair of ciphertexts, I created a Python function:

```python
def xor_bytes(a, b):
    return bytes(x ^ y for x, y in zip(a, b))
```

This allowed the ciphertexts to be compared efficiently.

---

## 4. Use Spaces as the First Clue

A useful property of ASCII is that XORing a space (`0x20`) with many English letters changes their case while still producing an alphabetic ASCII value.

Therefore, when:

```text
C1[position] XOR C2[position]
```

produces an alphabetic character, it provides evidence that **one of the two plaintexts may contain a space at that position**.

This does not prove which message contains the space. It only provides a useful hypothesis.

---

## 5. Score Possible Spaces

Instead of manually checking every pair of messages, I automated the process using:

```python
space_candidate()
calculate_space_scores()
```

With 12 known ciphertexts, each message could be compared against the other 11 messages.

Therefore, the maximum original space score was:

```text
11
```

A high score suggested that a particular plaintext position could contain a space.

These scores were treated as evidence rather than certainty.

---

## 6. Recover Possible Keystream Bytes

Once I suspected that a plaintext character was a space, I could calculate the corresponding keystream byte.

Starting with:

```text
C = P XOR K
```

we can rearrange it as:

```text
K = C XOR P
```

For a suspected space:

```text
K[position] = C[position] XOR 0x20
```

I implemented this using:

```python
recover_keystream_byte(ciphertext, position, plaintext_char)
```

Once a plaintext character was guessed correctly, the actual keystream byte for that position became known.

---

## 7. Test Possible Key Bytes Against Every Message

I did not automatically trust each space hypothesis.

I created:

```python
decrypt_position()
```

For example, if I suspected that **Message 4 at position 50** contained a space, I calculated:

```text
possible_key = ciphertext[50] XOR 0x20
```

I then applied that possible key byte to position 50 of all the other ciphertexts.

If the resulting characters looked like sensible English characters across multiple messages, this provided stronger evidence that the original hypothesis was correct.

---

## 8. Build the Keystream Gradually

I initialized the keystream with unknown values:

```python
keystream = [None] * len(secret)
```

As positions were recovered, I updated the corresponding byte:

```python
keystream[position] = key_byte
```

This allowed the keystream to be reconstructed gradually instead of requiring the entire key to be discovered at once.

---

## 9. Partially Decrypt Message 13

Using the recovered key bytes, I created:

```python
decrypt_with_keystream()
```

For every known keystream position, the function calculated:

```text
plaintext byte = ciphertext byte XOR key byte
```

Unknown positions were represented using `_`.

For example, a partial result could look similar to:

```text
The Web as I env_saged...
```

These partial English fragments became useful for generating new plaintext hypotheses.

---

## 10. Generate New Plaintext Hypotheses

Once recognizable words started appearing, surrounding context could be used to suggest missing characters.

For example:

```text
The Web as I env_saged
```

strongly suggests:

```text
The Web as I envisaged
```

Therefore, `i` becomes a reasonable hypothesis for the missing character.

These contextual guesses were still tested rather than accepted automatically.

---

## 11. Test Message 13 Guesses Against Messages 1–12

I created:

```python
guess_secret_char(position, guess)
```

The function took a guessed character from Message 13 and calculated the possible keystream byte:

```text
K[position] = C13[position] XOR guessed_character
```

That possible key byte was then applied to the same position in Messages 1–12.

If the resulting characters across the other messages appeared reasonable, it provided evidence that the Message 13 guess was likely correct.

---

## 12. Automate Repetitive Testing

Initially, the process required repeatedly performing the following steps:

1. Try a character.
2. Calculate the possible key byte.
3. Apply the byte to the other messages.
4. Inspect the resulting characters.
5. Accept or reject the hypothesis.

To reduce this repetitive work, I created:

```python
review_candidates()
```

The function automated the calculations while keeping the final decision manual.

The program displayed the results and asked whether the candidate should be added to the keystream.

This meant the analysis remained **human-guided rather than fully automatic**.

---

## 13. Add an English-Likeness Score

Manually inspecting every result became slow, so I created:

```python
plaintext_quality()
```

This function rewarded characters commonly found in English plaintext, including:

- Letters
- Spaces
- Common punctuation

Unusual special characters received little or no credit.

The resulting percentage represented **how English-like the output appeared**, not the probability that the guess was correct.

For example:

```text
100%
```

does **not** mean that the guess is certainly correct. It only means that all tested characters satisfied the scoring rules.

---

## 14. Recognize the Limitation of the Scoring System

An important limitation became apparent during testing.

Several different candidate characters could receive a score of `100%`.

For example, if every decrypted result happened to be an alphabetic character, the scoring function could consider all of them completely valid even though only one candidate was actually correct.

Therefore, the score was used to:

```text
Filter plausible candidates
```

rather than:

```text
Prove that a candidate was correct
```

Context and evidence from the other ciphertexts were still necessary.

---

## 15. Correct Incorrect Keystream Bytes

Because the techniques produced hypotheses rather than guaranteed answers, some early guesses were incorrect.

When later evidence contradicted an earlier guess, I replaced the corresponding keystream byte and tested a new hypothesis.

For example, one partial result initially produced:

```text
Tie
```

Further analysis showed that the correct plaintext was:

```text
Tim
```

This required correcting the corresponding keystream byte.

---

## 16. Use Surrounding Context for Difficult Positions

Towards the end of the ciphertext, fewer messages were long enough to provide evidence for each position.

As a result, multiple guesses could receive similar English-likeness scores.

I created:

```python
show_context(position, guess)
```

This displayed surrounding decrypted characters from the other messages.

Recognizable word fragments provided stronger evidence than looking at individual characters alone.

I continued this process until all the keystream positions required to decrypt Message 13 had been recovered.

---

## 17. Produce the Final Plaintext

Finally, I used:

```python
decrypt_with_keystream()
```

to XOR Message 13 with the recovered keystream.

Once there were no unknown keystream positions remaining, the complete plaintext was recovered.

### Decrypted Message 13

```text
The Web as I envisaged it we have not seen it yet.The future is still so much bigger than the past Tim Berners-Lee
```

---

## Functions Used

The main Python functions used during the attack were:

```python
xor_bytes()
space_candidate()
calculate_space_scores()
recover_keystream_byte()
decrypt_position()
decrypt_with_keystream()
guess_secret_char()
review_candidates()
plaintext_quality()
show_context()
```

Together, these functions allowed me to identify likely spaces, recover individual keystream bytes, test hypotheses against all ciphertexts, evaluate plaintext quality, correct mistakes, and eventually reconstruct the plaintext of Message 13.

---

## Conclusion

This exercise demonstrates why **keystream reuse is a critical vulnerability in stream ciphers**.

The attack did not require directly breaking the underlying cipher. Instead, it exploited the fact that the same keystream was used more than once.

By combining:

```text
Ciphertext XOR comparisons
        ↓
Space detection
        ↓
Keystream-byte hypotheses
        ↓
Cross-message verification
        ↓
Partial plaintext recovery
        ↓
English/context analysis
        ↓
Keystream reconstruction
        ↓
Message 13 plaintext
```

I was able to progressively recover enough of the keystream to decrypt the target message.

The exercise highlights an important cryptographic rule:

> **A stream-cipher keystream must never be reused.**