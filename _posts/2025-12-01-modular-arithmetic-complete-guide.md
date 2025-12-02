---
title: Modular Arithmetic Complete Guide
date: 2025-12-01 13:00 +0530
categories: [Number Theory]
tags: [math, modular, number-theory, mod, inverse, exponentiation, crt, fermat, euler, ioi-prep]
excerpt: Complete guide to modular arithmetic — including modular inverse, exponentiation, Fermat’s theorem, Euler’s theorem, and the Chinese Remainder Theorem for competitive programming.
---

## 🔹 Mathematical Proofs (Beginner-friendly)

Short, intuitive proofs and small examples for the key facts used in this guide.

### 1. Correctness of binary modular exponentiation
Invariant: maintain `result * a^e ≡ A^E (mod m)` where `A^E` is the original power.  
Each loop:
- if current low bit of `e` is 1, `result ← result * a` (keeps invariant),
- then `a ← a*a`, `e ← e>>1` (keeps invariant).
When `e=0` we have `result ≡ A^E (mod m)`. This gives O(log e) steps.

Example: compute $2^{13}\bmod1000$ via binary method — repeated squaring and selective multiplies produce the same reduced value as direct power.

### 2. Fermat’s Little Theorem (sketch) — $a^{p-1}\equiv1\pmod p$
For prime $p$ and $a$ not divisible by $p$, the map $x\mapsto ax\pmod p$ permutes $\{1,\dots,p-1\}$.  
Multiply all values: $a^{p-1}(p-1)!\equiv(p-1)!\pmod p$. Cancel $(p-1)!$ (nonzero mod $p$) to get the result.  
Corollary: $a^{-1}\equiv a^{p-2}\pmod p$.

Example: $3^{6}\bmod7=1$, so $3^{-1}\equiv3^{5}\bmod7=5$.

### 3. Euler’s theorem (sketch) — $a^{\varphi(n)}\equiv1\pmod n$
If $\gcd(a,n)=1$, the multiplicative residues modulo $n$ form a group of size $\varphi(n)$. By Lagrange's theorem for finite groups, $a^{\varphi(n)}$ is the identity, i.e. $1$ mod $n$.

Example: $n=10$, $\varphi(10)=4$, and $3^4\bmod10=81\bmod10=1$.

### 4. Extended Euclidean Algorithm correctness (inverse)
Recurrence finds $x,y$ with $ax+by=\gcd(a,b)$. Base: $b=0$ returns $(1,0)$. Step: reduce to $(b,a\bmod b)$ and back-substitute using $a\bmod b=a-\lfloor a/b\rfloor b$. If $\gcd(a,m)=1$, the returned $x$ satisfies $ax\equiv1\pmod m$ (modular inverse).

Example: inverse of $3$ mod $11$ → extended gcd gives $x=4$ since $3·4+11·(−1)=1$, so $3^{-1}\equiv4$.

### 5. Chinese Remainder Theorem (intuition + correctness)
For pairwise coprime moduli $m_i$, construct $M=\prod m_i$, $M_i=M/m_i$. Each $M_i$ has inverse $y_i$ mod $m_i$. Then
$$x=\sum r_i M_i y_i\pmod M$$
satisfies all congruences because for each $i$, $M_j\equiv0\pmod{m_i}$ when $j\ne i$, and $M_i y_i\equiv1\pmod{m_i}$.

Example: solve
$x\equiv2\pmod3,\;x\equiv3\pmod4,\;x\equiv2\pmod5$.
Compute $M=60$, $M_1=20,y_1=2$, $M_2=15,y_2=3$, $M_3=12,y_3=3$. Then
$x=(2·20·2 + 3·15·3 + 2·12·3)\bmod60 = 47\bmod60$.

---

# ⚙️ Modular Arithmetic Complete Guide

Ye article tumhe **modular arithmetic ka complete roadmap** deta hai —  
basic se lekar advanced tak: modular inverse, modular exponentiation, Fermat, Euler aur CRT sab kuch ek jagah 🔥  

---

## 🔹 1. Modular Arithmetic Basics

If we write:

    a ≡ b (mod m)

It means `(a - b)` is divisible by `m`,  
yaani `a` aur `b` dono ka remainder same hai jab `m` se divide karte ho.

Example:

    17 ≡ 2 (mod 5)
    because 17 - 2 = 15 which is divisible by 5.

---

### 🔸 Basic Properties

1. (a + b) mod m = (a mod m + b mod m) mod m  
2. (a - b) mod m = (a mod m - b mod m + m) mod m  
3. (a × b) mod m = (a mod m × b mod m) mod m  

---

## 🔹 2. Modular Exponentiation

To calculate:

    a^b mod m

Direct method is slow if b is large,  
so we use **fast exponentiation (O(log b))** method.

    long long modExp(long long a, long long b, long long m) {
        long long result = 1;
        a %= m;
        while (b > 0) {
            if (b & 1)
                result = (result * a) % m;
            a = (a * a) % m;
            b >>= 1;
        }
        return result;
    }

---

## 🔹 3. Binary Exponentiation (Fast Power)

Concept: break exponent into binary form.

Example:  
13 = (1101)₂  
so a¹³ = a⁸ × a⁴ × a¹

Each step:
- if bit = 1 → multiply result by a
- square a each time
- shift b right

✅ Time: O(log b)

---

## 🔹 4. Modular Multiplicative Inverse

We want x such that:

    (a × x) mod m = 1

Inverse exists only if gcd(a, m) = 1.

---

### Method 1: Fermat’s Little Theorem (for prime m)

    a^(m-2) mod m = a^(-1) mod m

Example:

    a = 3, m = 7
    inverse = 3^(7-2) mod 7 = 3^5 mod 7 = 5

---

### Method 2: Extended Euclidean Algorithm (for any m)

    a×x + m×y = 1  →  x = a^(-1) mod m

    long long gcdExtended(long long a, long long b, long long &x, long long &y) {
        if (b == 0) { x = 1; y = 0; return a; }
        long long x1, y1;
        long long g = gcdExtended(b, a % b, x1, y1);
        x = y1;
        y = x1 - (a / b) * y1;
        return g;
    }

    long long modInverse(long long a, long long m) {
        long long x, y;
        long long g = gcdExtended(a, m, x, y);
        if (g != 1) return -1;
        return (x % m + m) % m;
    }

---

## 🔹 5. Modular Division

Division not directly possible under modulo.  
We replace it using inverse:

    (a / b) mod m = (a × b^(-1)) mod m

Example:

    (6 / 3) mod 7 = (6 × 3^(-1)) mod 7 = (6 × 5) mod 7 = 30 mod 7 = 2

---

## 🔹 6. Fermat’s Little Theorem (FLT)

If p is prime and a not divisible by p:

    a^(p-1) ≡ 1 (mod p)
    ⇒ a^(p-2) ≡ a^(-1) (mod p)

Example:

    3^(7-1) mod 7 = 3^6 mod 7 = 1

Use: fast modular inverse when mod is prime.

---

## 🔹 7. Euler’s Theorem

Generalization of Fermat’s theorem (for any n):

If gcd(a, n) = 1, then:

    a^φ(n) ≡ 1 (mod n)

where φ(n) = count of numbers < n that are coprime to n.

Example:

    a = 3, n = 10
    φ(10) = 4
    3^4 mod 10 = 81 mod 10 = 1 ✅

---

### Formula for φ(n)

If n = p₁^a₁ × p₂^a₂ × … × pₖ^aₖ then:

    φ(n) = n × (1 - 1/p₁) × (1 - 1/p₂) × … × (1 - 1/pₖ)

Example:
    φ(10) = 10 × (1 - 1/2) × (1 - 1/5) = 4

---

## 🔹 8. Extended Euclidean Algorithm

Finds x and y such that:

    a×x + b×y = gcd(a, b)

Used in modular inverse and CRT.

    long long gcdExtended(long long a, long long b, long long &x, long long &y) {
        if (b == 0) { x = 1; y = 0; return a; }
        long long x1, y1;
        long long g = gcdExtended(b, a % b, x1, y1);
        x = y1;
        y = x1 - (a / b) * y1;
        return g;
    }

---

## 🔹 9. Chinese Remainder Theorem (CRT)

Used to solve system of modular equations:

    x ≡ r₁ (mod m₁)
    x ≡ r₂ (mod m₂)
    ...
    x ≡ rₙ (mod mₙ)

If all mᵢ are pairwise coprime:

    M = m₁ × m₂ × … × mₙ
    Mᵢ = M / mᵢ
    yᵢ = Mᵢ^(-1) mod mᵢ
    x = Σ (rᵢ × Mᵢ × yᵢ) mod M

Example:

    x ≡ 2 (mod 3)
    x ≡ 3 (mod 4)
    x ≡ 2 (mod 5)

→ x = 47 mod 60 ✅

---

## 🔹 10. Summary Table 💡

| Concept | Formula | Works When |
|----------|----------|------------|
| Addition | (a + b) mod m | Always |
| Multiplication | (a × b) mod m | Always |
| Exponentiation | a^b mod m | Always |
| Modular inverse | a^(m−2) mod m | m prime |
| Euler’s theorem | a^φ(n) ≡ 1 mod n | gcd(a,n)=1 |
| Fermat’s theorem | a^(p−1) ≡ 1 mod p | p prime |
| Modular division | a × b^(-1) mod m | gcd(b,m)=1 |
| CRT | x = Σ(rᵢMᵢyᵢ) mod M | coprime moduli |

---

## 🔹 11. Takeaway 🚀

Mastering modular arithmetic gives you power to solve:
- modular inverse problems  
- combinatorics mod p  
- cryptography (RSA, Fermat)  
- and large-number computations without overflow  

