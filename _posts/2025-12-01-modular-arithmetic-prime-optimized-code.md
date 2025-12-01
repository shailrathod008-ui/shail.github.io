---
title: Modular Arithmetic + Prime Code
date: 2025-12-01 13:00 +0530
categories: [Number Theory]
tags: [math, modular, number-theory, mod, inverse, exponentiation, crt, fermat, euler]
excerpt: Modular Arithmetic + Prime Code
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
