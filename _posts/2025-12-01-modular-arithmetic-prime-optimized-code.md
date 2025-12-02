---
title: Modular Arithmetic + Prime Code
date: 2025-12-01 13:00 +0530
categories: [Number Theory]
tags: [math, modular, number-theory, mod, inverse, exponentiation, crt, fermat, euler]
excerpt: Modular Arithmetic + Prime Code
---
## 🔹 Mathematical Proofs (Beginner-friendly)

Short, intuitive proofs for the algorithms in the post. Uses simple invariants and tiny examples.

### 1. Binary (fast) modular exponentiation — why it works

Invariant used by the loop: at any point, let res be the accumulated result and a be the current base, e the remaining exponent. The invariant is
$$
\text{res}\cdot a^{e} \equiv A^{E}\pmod m
$$
where $A^{E}$ is the original power being computed. Each loop step either multiplies res by a (when the low bit of e is 1) or squares a and halves e. These operations preserve the invariant, and when $e=0$ we get $\text{res}\equiv A^{E}\pmod m$, which is the answer.

Example:
- Compute $2^{13}\bmod 1000$ with binary method. Steps produce the same final result $819$ as plain exponentiation reduced mod 1000.

### 2. Fermat’s Little Theorem — justification and inverse using it

Statement: if $p$ is prime and $\gcd(a,p)=1$ then
$$
a^{p-1}\equiv 1\pmod p.
$$
Sketch proof: multiply the residues $1,2,\dots,p-1$ by $a$ modulo $p$. The set $\{a\cdot1,\dots,a\cdot(p-1)\}$ is a permutation of $\{1,\dots,p-1\}$. Multiplying all terms gives
$$
a^{p-1}(p-1)!\equiv (p-1)!\pmod p.
$$
Since $(p-1)!\not\equiv0\pmod p$, we can cancel it to get $a^{p-1}\equiv1\pmod p$.

Using this, when modulus $m$ is prime, the inverse of $a$ is
$$
a^{-1}\equiv a^{m-2}\pmod m.
$$

Example:
- $p=7$, $a=3$: $3^{6}\bmod7=1$, and inverse $3^{-1}\equiv3^{5}\bmod7=5$ (indeed $3\cdot5\equiv1\pmod7$).

### 3. Extended Euclidean Algorithm — correctness for inverse

The recursive Extended Euclid returns integers $x,y$ with
$$
ax+by=\gcd(a,b).
$$
Proof idea: base case when $b=0$ returns $(1,0)$ since $a\cdot1+b\cdot0=a$. For $b\ne0$ we recurse on $(b,a\bmod b)$ which gives coefficients for $\gcd(b,a\bmod b)$. Expressing $a\bmod b=a-\lfloor a/b\rfloor b$ and substituting yields coefficients for $\gcd(a,b)$. This recursion constructs valid $x,y$ by linear combination.

If $\gcd(a,m)=1$, the $x$ produced satisfies $ax\equiv1\pmod m$ and is the modular inverse.

Examples:
- Inverse of $3$ mod $11$: extended gcd yields $x=4$ since $3\cdot4+11\cdot(-1)=1$, so $3^{-1}\equiv4\pmod{11}$.
- If $\gcd(a,m)\ne1$ (e.g., $a=3,m=9$) the algorithm returns gcd$>1$ and no inverse exists.

### 4. Sieve of Eratosthenes — short proof

Claim: after running sieve up to $N$, unmarked numbers are exactly primes $\le N$.

Reason: every composite $n$ has a smallest prime divisor $p\le\sqrt n$. When sieve processes $p$ it marks all multiples of $p$ starting at $p^2\le n$, so $n$ is marked. Primes are never marked because no smaller prime divides them. Thus unmarked numbers are primes.

Example:
- Sieve up to $50$ finds primes: $2,3,5,7,11,13,17,19,23,29,31,37,41,43,47$.

---

```cpp
// ===============================
// Number Theory Utils
// Modular Exponentiation, Inverse, Primes
// ===============================

#include <bits/stdc++.h>
using namespace std;

using int64 = long long;

const int64 MOD = 1'000'000'007; // common prime modulus

// -------------------------------
// Fast Modular Exponentiation
// Computes a^e mod m in O(log e)
// -------------------------------
int64 mod_pow(int64 a, int64 e, int64 m = MOD) {
    a %= m;
    if (a < 0) a += m;
    int64 res = 1;
    while (e > 0) {
        if (e & 1) res = (__int128)res * a % m; // safe for 64-bit
        a = (__int128)a * a % m;
        e >>= 1;
    }
    return res;
}

// -----------------------------------------
// Modular Inverse (when m is prime)
// Uses Fermat's Little Theorem:
// a^(m-2) ≡ a^(-1) (mod m)
// Requires: gcd(a, m) = 1 and m is prime
// -----------------------------------------
int64 mod_inv_prime(int64 a, int64 m = MOD) {
    // returns a^(-1) mod m (m must be prime)
    return mod_pow(a, m - 2, m);
}

// ----------------------------------------------------
// Extended Euclidean Algorithm
// Finds x, y such that: a*x + b*y = gcd(a, b)
// Used for modular inverse when m is NOT prime
// ----------------------------------------------------
int64 gcd_ext(int64 a, int64 b, int64 &x, int64 &y) {
    if (b == 0) {
        x = 1;
        y = 0;
        return a;
    }
    int64 x1, y1;
    int64 g = gcd_ext(b, a % b, x1, y1);
    x = y1;
    y = x1 - (a / b) * y1;
    return g;
}

// -----------------------------------------------------------
// Modular Inverse for any modulus (not necessarily prime)
// Returns -1 if inverse does not exist (i.e., gcd(a, m) != 1)
// -----------------------------------------------------------
int64 mod_inv_any(int64 a, int64 m) {
    int64 x, y;
    int64 g = gcd_ext(a, m, x, y);
    if (g != 1) return -1; // inverse doesn't exist
    x %= m;
    if (x < 0) x += m;
    return x;
}

// ===============================
// PRIME UTILITIES
// ===============================

// ---------------------------
// Simple O(sqrt(n)) prime test
// ---------------------------
bool is_prime(int64 n) {
    if (n <= 1) return false;
    if (n <= 3) return true;
    if (n % 2 == 0) return false;
    for (int64 i = 3; i * i <= n; i += 2) {
        if (n % i == 0) return false;
    }
    return true;
}

// --------------------------------------
// Sieve of Eratosthenes (0..N)
// Fills is_prime[] and primes[]
// --------------------------------------
struct Sieve {
    int N;
    vector<bool> is_prime;
    vector<int> primes;

    Sieve(int n) {
        init(n);
    }

    void init(int n) {
        N = n;
        is_prime.assign(N + 1, true);
        if (N >= 0) is_prime[0] = false;
        if (N >= 1) is_prime[1] = false;

        for (int i = 2; i * i <= N; ++i) {
            if (is_prime[i]) {
                for (int j = i * i; j <= N; j += i)
                    is_prime[j] = false;
            }
        }
        primes.clear();
        for (int i = 2; i <= N; ++i)
            if (is_prime[i])
                primes.push_back(i);
    }
};

// ===============================
// Example usage (comment out in CP)
// ===============================

int main() {
    // 1) modular exponentiation
    cout << "2^10 mod MOD = " << mod_pow(2, 10) << "\n";

    // 2) modular inverse with prime MOD
    int64 inv2 = mod_inv_prime(2);
    cout << "inverse of 2 mod MOD = " << inv2 << "\n";
    cout << "check: 2 * inv2 mod MOD = " << (2 * inv2 % MOD) << "\n";

    // 3) modular inverse with any modulus
    int64 a = 3, m = 10;
    int64 inv_any = mod_inv_any(a, m);
    cout << "inverse of " << a << " mod " << m << " = " << inv_any << "\n";
    if (inv_any != -1)
        cout << "check: a * inv_any % m = " << (a * inv_any % m) << "\n";

    // 4) prime check
    cout << "is 97 prime? " << (is_prime(97) ? "YES" : "NO") << "\n";

    // 5) sieve
    Sieve sv(50);
    cout << "primes up to 50: ";
    for (int p : sv.primes) cout << p << " ";
    cout << "\n";

    return 0;
}
```
