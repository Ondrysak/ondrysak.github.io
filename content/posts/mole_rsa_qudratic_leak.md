
---
title: m0leCon CTF 2025 Teaser - Exploiting RSA with a Quadratic Leak
description: Writeup on exploiting an RSA vulnerability using a leak in quadratic form.
date: 2024-09-15
tldr: Factoring RSA modulus using a quadratic leak.
draft: false
tags: ["ctf", "writeup", "crypto", "rsa", "quadratic leak", "number theory"]
---

## Exploiting RSA with a Quadratic Leak

In this writeup, we explore how to exploit an RSA vulnerability where a leak in quadratic form is provided. The challenge involves factoring the RSA modulus `n` given the equation:

```
leak ≡ p^2 + q^2 - p - q (mod n)
```

from the code below

```python
from Crypto.PublicKey import RSA
from Crypto.Util.number import long_to_bytes, bytes_to_long

flag = b'ptm{REDACTED}'
flag = bytes_to_long(flag)

key = RSA.generate(2048)
n = key.n
e = key.e
p, q = key.p, key.q

leak = (p**2 + q**2 - p - q)%key.n

ciph = pow(flag, key.e, key.n)
ciph = long_to_bytes(ciph)

print(f'{n = }')
print(f'{e = }')
print(f'{ciph = }')
print(f'{leak = }')
```

and the actual values

```bash
cat .\output.txt 
n = 18981841105895636526675685852772632158842968216380018810386298633786155635996081803989776444322885832862508871280454015266715098097212762303823817669399277859053310692838636039765017523835850272300243126797277954387477501506925762258179024160726332434715115654485883242651742714579029928789693932848304595300833348009779987606253681750351002940239431890559231554840489325837351081804589516122959648786979395257143753954514874848903523580352634665455711644772319481340052722305319056316013353993453687301538982087215697860640865435677605005049256026879380517059221652220476972139202570771113865372482940490650024591211
e = 65537
ciph = b'/>\xf0a\xe4\x95a\xb8!h\x11[\xa01\xa2\x08\xd2\xa0\xd7G\xb0CU`\x97\xc7\xa2\xe9\x94\x10=\xf7\xd6y{\x01u\xcbl\xe1\x7f\x8d\xa0\xd8_\xba\x8a\xd9\x85\xc1\xcc\rK\xc7\xb6\xfb.(\x99%\x1f\x0b\xeci\nR\x8b\xcc>\x86\x18\xe1ie\x19\x9dI\xda\x9bNh8N\xe5\xf2\xf3kP\xeb\x02d\xdei\xadd\xa0\x17WKs~3ohp\xeb\x83\x17>[\x87K\xd1\x0e>\xf6\xeeR\x08\xc4y\x9c\xc7\xbf\xdaq\x9a\xff\x99EQSg\x04"H\xc0=\xfc\x1b\xdcA\x95\xb9\x15m\xe6\x04N\x81%\xda\xa5\xb2P\x02\xbb\xb9\x83\xbf\xcfR\xffe\x0fT\xd9=f\xc1\xab\xd3:\xfb\xf8k\xc0\xd1]V\xc6pK\x86T\xfe\xd5\xadb\x7f\x1b\xb1\x9d9\xe51\xb8\'BY\xcb\x934\x1e\xfb\xc8\x96V4\x0f\xb3\xe4\xd9\x92e;v\x88\x8e\x16\x14?\xe9ew\x95\x8c\xa2\xfc\xc3\xf7,\xd50\xc3@\x11|\x8a\xe0\xee\xf3u\xa0\x8a_?A2[\xad\x8e\x81\x99\xf0\r'
leak = 110277838449088002585424592968018361142419851286027284241723700708406129526007148628583636124051067276081936889565484783806534793568491888186904160969266846545579940493300734427497600123059580090561311191997050900418560468769108541786093143743441372956591688646550593311523980201035231511541259853198512190703006995530566933548862787123808685889850952840156903871573965022335639190635103252052979042642912530238710466180279568731038056810280199578414064760950459385877300845399348928704648108281000674478903445454838155321984920936780746021819652974978142789457099370813760033995294668404782700700352547809925049256
```

### Understanding the Leak

The leak provides information related to the prime factors `p` and `q` of `n`. By manipulating the given leak, we can derive a quadratic equation that helps us factor `n`. Here's how it works:

1. **Leak Information**: The leak is provided in the form of a quadratic equation involving the primes `p` and `q`. Specifically, the equation is:

   ```
   leak ≡ p^2 + q^2 - p - q (mod n)
   ```

   This equation suggests that we can express the leak in terms of `p` and `q`, and use it to derive a quadratic equation involving their sum, `S = p + q`.

2. **Quadratic Identity**: We use the identity `p^2 + q^2 = (p + q)^2 - 2pq` to rewrite the leak as:

   ```
   leak ≡ (p + q)^2 - 2n - (p + q) (mod n)
   ```

   Here, `pq = n` (since `n` is the product of the primes `p` and `q`), which simplifies the equation to:

   ```
   leak ≡ S^2 - S - 2n (mod n)
   ```

   This equation can be solved for `S` using the quadratic formula.

### The Exploit

#### Step 1: Deriving the Quadratic Equation

Given the equation from the leak:

```
leak ≡ p^2 + q^2 - p - q (mod n)
```

We rewrite it using the identity:

```
leak ≡ (p + q)^2 - 2n - (p + q) (mod n)
```

Let `S = p + q`. This simplifies to:

```
leak ≡ S^2 - S - 2n (mod n)
```

Now, we can express this as a quadratic equation without the modulus:

```
S^2 - S - (leak + k * n) = 0
```

Where `k` is an integer that adjusts for potential multiples of `n`.

#### Step 2: Solving the Quadratic Equation

We solve the quadratic equation for `S` using the quadratic formula:

```
S = (1 ± sqrt(1 + 4*(leak + k * n)))/2
```

Here, we iterate over possible values of `k` and check if the discriminant `(1 + 4*(leak + k * n))` is a perfect square. If it is, we can calculate `S` and then `p` and `q`.

#### Step 3: Factoring `n` and Decrypting

Once we determine `S = p + q`, the factors `p` and `q` are found by solving:

```
p + q = S
p * q = n
```

With `p` and `q` known, we compute `φ(n)`, find the private exponent `d`, and decrypt the ciphertext. Here's how the code works in practice:

### Python Code Explanation

The following Python code implements the above logic to factor the RSA modulus and decrypt the ciphertext:

```python
from math import isqrt
from Crypto.Util.number import long_to_bytes, inverse

# Given n, e, ciph, and leak
n = 18981841105895636526675685852772632158842968216380018810386298633786155635996081803989776444322885832862508871280454015266715098097212762303823817669399277859053310692838636039765017523835850272300243126797277954387477501506925762258179024160726332434715115654485883242651742714579029928789693932848304595300833348009779987606253681750351002940239431890559231554840489325837351081804589516122959648786979395257143753954514874848903523580352634665455711644772319481340052722305319056316013353993453687301538982087215697860640865435677605005049256026879380517059221652220476972139202570771113865372482940490650024591211
e = 65537
ciph = b'/>\xf0a\xe4\x95a\xb8!h\x11[\xa01\xa2\x08\xd2\xa0\xd7G\xb0CU`\x97\xc7\xa2\xe9\x94\x10=\xf7\xd6y{\x01u\xcbl\xe1\x7f\x8d\xa0\xd8_\xba\x8a\xd9\x85\xc1\xcc\rK\xc7\xb6\xfb.(\x99%\x1f\x0b\xeci\nR\x8b\xcc>\x86\x18\xe1ie\x19\x9dI\xda\x9bNh8N\xe5\xf2\xf3kP\xeb\x02d\xdei\xadd\xa0\x17WKs~3ohp\xeb\x83\x17>[\x87K\xd1\x0e>\xf6\xeeR\x08\xc4y\x9c\xc7\xbf\xdaq\x9a\xff\x99EQSg\x04"H\xc0=\xfc\x1b\xdcA\x95\xb9\x15m\xe6\x04N\x81%\xda\xa5\xb2P\x02\xbb\xb9\x83\xbf\xcfR\xffe\x0fT\xd9=f\xc1\xab\xd3:\xfb\xf8k\xc0\xd1]V\xc6pK\x86T\xfe\xd5\xadb\x7f\x1b\xb1\x9d9\xe51\xb8\'BY\xcb\x934\x1e\xfb\xc8\x96V4\x0f\xb3\xe4\xd9\x92e;v\x88\x8e\x16\x14?\xe9ew\x95\x8c\xa2\xfc\xc3\xf7,\xd50\xc3@\x11|\x8a\xe0\xee\xf3u\xa0\x8a_?A2[\xad\x8e\x81\x99\xf0\r'
leak = 110277838449088002585424592968018361142419851286027284241723700708406129526007148628583636124051067276081936889565484783806534793568491888186904160969266846545579940493300734427497600123059580090561311191997050900418560468769108541786093143743441372956591688646550593311523980201035231511541259853198512190703006995530566933548862787123808685889850952840156903871573965022335639190635103252052979042642912530238710466180279568731038056810280199578414064760950459385877300845399348928704648108281000674478903445454838155321984920936780746021819652974978142789457099370813760033995294668404782700700352547809925049256


# Convert ciph to integer if it's in bytes
if isinstance(ciph, bytes):
    ciph = int.from_bytes(ciph, 'big')

MAX_K = 1000000  # You can adjust this limit based on your resources

for k in range(MAX_K):
    D_squared = 1 + 4 * leak + 4 * k * n
    D = isqrt(D_squared)
    if D * D == D_squared:
        S_candidates = [(1 + D) // 2, (1 - D) // 2]
        for S in S_candidates:
            if S > 0 and S < n:
                D2_squared = S * S - 4 * n
                if D2_squared >= 0:
                    D2 = isqrt(D2_squared)
                    if D2 * D2 == D2_squared:
                        p = (S + D2) // 2
                        q = (S - D2) // 2
                        if p * q == n:
                            print("Found factors!")
                            print(f"p = {p}")
                            print(f"q = {q}")
                            # Now compute private key components
                            phi = (p - 1) * (q - 1)
                            d = inverse(e, phi)
                            # Decrypt the ciphertext
                            flag_int = pow(ciph, d, n)
                            flag = long_to_bytes(flag_int)
                            print(f"Flag: {flag}")
                            exit()
```

#### Code Walkthrough

- **Imports**: We import necessary libraries for math operations and cryptographic utilities.
- **Variables**: The given RSA parameters include the modulus `n`, public exponent `e`, ciphertext `ciph`, and the leak value.
- **Ciphertext Conversion**: If the ciphertext is in bytes, it's converted to an integer for processing.
- **Iterating Over k**: The code iterates over potential values of `k` to solve for `S`, ensuring the discriminant is a perfect square.
- **Solving for p and q**: Once a valid `S` is found, the corresponding `p` and `q` values are calculated, and the code checks if they correctly factor `n`.
- **Decrypting the Ciphertext**: After finding `p` and `q`, the private key `d` is calculated, and the ciphertext is decrypted to reveal the flag.

## Why the Search Space is Smaller in an RSA Quadratic Leak Attack

The key insight that makes the search space much smaller in an RSA attack involving a quadratic leak.

### 1. Quadratic Equation and `p + q`

The leak is given as:

```
leak ≡ p^2 + q^2 - p - q (mod n)
```

By manipulating this equation, we derived a quadratic form involving `S = p + q`:

```
S^2 - S - (leak + k * n) = 0
```

This is a quadratic equation in `S` (where `S = p + q`). Normally, finding two large primes `p` and `q` by brute force would involve checking a vast number of potential pairs `p` and `q`, making it computationally infeasible for large `n`. However, by focusing on `S` instead, we reduce the complexity significantly.

### 2. Discriminant as a Perfect Square

The quadratic equation for `S` is solved using the quadratic formula:

```
S = (1 ± sqrt(1 + 4*(leak + k * n)))/2
```

Here, the key point is that the expression inside the square root, called the discriminant:

```
D^2 = 1 + 4*(leak + k * n)
```

must be a perfect square for `S` to be an integer. The requirement for the discriminant to be a perfect square greatly reduces the number of possible values for `k`. If the discriminant is not a perfect square, then the corresponding `S` would not yield integer values for `p` and `q`, making the corresponding `k` invalid.

### 3. Search Space Reduction

Instead of searching over all possible pairs `p` and `q`, which would be infeasible, the algorithm only needs to iterate over possible values of `k`. The number of valid `k` values is much smaller because it must satisfy the condition that `D^2` is a perfect square.

Given that `k` is an integer and only certain values of `k` will result in a perfect square for the discriminant, the algorithm effectively narrows down the possibilities to a manageable set. Once a valid `S = p + q` is found, it becomes straightforward to solve for `p` and `q` by solving the quadratic equation `x^2 - Sx + n = 0`.

### 4. Efficiency Gains

- **Quadratic Reduction**: Instead of dealing with the complexity of two unknowns (the large primes `p` and `q`), the problem is reduced to a single unknown `S = p + q`.
- **Perfect Square Condition**: The condition that the discriminant must be a perfect square filters out most of the invalid candidates for `k`, dramatically reducing the search space.
- **Direct Factoring**: Once a valid `k` gives an integer `S`, the factorization of `n` into `p` and `q` is immediate.


```bash
python .\sol.py
Found factors!
p = 143125281782416009706859847355990296042448284896710979917870298666128046947387773279723109414364422079040366407054354066842700094377705158187913015737336135975689114612440664825223308423840255373057625672080120880064164570493121313466590904906282427310175864090387017963389661584214322547107473751463751914467
q = 132623956225654710889697216335897143716270437075787240261040201936596224312214742992316741484930970708099716962699765964724942297757102181289126503147878055832200350652096164114782859459612903845763823794502334498985175549784061474707320565790793521524296832423247175094640850952793567503777217992756361481433
Flag: b'ptm{Nonlinear_Algebra_Preserved_Over_Large_Integers}'
```

### Conclusion

By leveraging the quadratic leak, we can factor the RSA modulus `n` and ultimately decrypt the message. This attack demonstrates the importance of secure key generation and management in cryptographic systems. The provided code shows how this process can be automated to exploit the vulnerability effectively.

