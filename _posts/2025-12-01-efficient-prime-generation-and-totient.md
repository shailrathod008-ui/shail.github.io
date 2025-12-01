---
title: Efficient Prime Generation and Euler’s Totient Precomputation
date: 2025-12-01 14:00 +0530
categories: [Number Theory]
tags: [math, number-theory, primes, totient, sieve]
excerpt: Fast methods to compute primes and Euler’s Totient Function using sieve — the foundation of modular arithmetic and competitive programming number theory.
---

# 🔢 Efficient Prime Generation and Euler’s Totient Precomputation

Prime numbers aur Euler’s Totient Function (φ) dono number theory ke backbone topics hain.  
Agar tum efficient prime generation aur totient precomputation samajh jaate ho, to modular arithmetic aur cryptography ke bahut saare problems easy ho jaate hain.  
Chalo step-by-step samajhte hain 👇  

---

## 🔹 Step 1: Recall – Simple Sieve of Eratosthenes

Sieve of Eratosthenes ek efficient algorithm hai jo 1 se N tak ke saare primes nikalta hai.

    vector<bool> sieve(int n) {
        vector<bool> prime(n + 1, true);
        prime[0] = prime[1] = false;

        for (int i = 2; i * i <= n; i++) {
            if (prime[i]) {
                for (int j = i * i; j <= n; j += i)
                    prime[j] = false;
            }
        }
        return prime;
    }

✅ Time Complexity: O(n log log n)  
✅ Space Complexity: O(n)

---

## 🔹 Step 2: Store Prime List Directly

    vector<int> generatePrimes(int n) {
        vector<bool> isPrime(n + 1, true);
        vector<int> primes;

        isPrime[0] = isPrime[1] = false;
        for (int i = 2; i <= n; i++) {
            if (isPrime[i]) {
                primes.push_back(i);
                for (long long j = 1LL * i * i; j <= n; j += i)
                    isPrime[j] = false;
            }
        }
        return primes;
    }

➡️ Returns all prime numbers ≤ N.  
Example: generatePrimes(50) → [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]

---

## 🔹 Step 3: Euler’s Totient Function φ(n)

φ(n) = number of integers ≤ n that are coprime with n.

Formula:
φ(n) = n × (1 - 1/p₁) × (1 - 1/p₂) × … × (1 - 1/pₖ)
where pᵢ are the distinct prime factors of n.

Example:
φ(10) = 10 × (1 - 1/2) × (1 - 1/5) = 4

---

## 🔹 Step 4: Precompute φ(n) for all n ≤ N

Sieve ke similar approach se hum φ(n) sab numbers ke liye precompute kar sakte hain.

    vector<int> computeTotients(int n) {
        vector<int> phi(n + 1);
        for (int i = 0; i <= n; i++) phi[i] = i;

        for (int p = 2; p <= n; p++) {
            if (phi[p] == p) { // prime check
                for (int j = p; j <= n; j += p)
                    phi[j] -= phi[j] / p;
            }
        }
        return phi;
    }

✅ Time Complexity: O(n log log n)  
✅ Space Complexity: O(n)

---

### Example Output for n = 10

| n | φ(n) |
|---|------|
| 1 | 1 |
| 2 | 1 |
| 3 | 2 |
| 4 | 2 |
| 5 | 4 |
| 6 | 2 |
| 7 | 6 |
| 8 | 4 |
| 9 | 6 |
| 10 | 4 |

---

## 🔹 Step 5: Combined Example

    int N = 1000000;
    auto primes = generatePrimes(N);
    auto phi = computeTotients(N);

    cout << "Number of primes up to " << N << ": " << primes.size() << endl;
    cout << "phi(10) = " << phi[10] << endl;
    cout << "phi(100) = " << phi[100] << endl;

Output Example:

    Number of primes up to 1000000: 78498
    phi(10) = 4
    phi(100) = 40

---

## 🔹 Step 6: Applications

| Concept | Use |
|----------|-----|
| Euler’s theorem | a^φ(n) ≡ 1 (mod n) |
| Modular inverse | a⁻¹ ≡ a^{φ(n)-1} (mod n) |
| Cryptography | RSA keys use φ(n) = (p−1)(q−1) |
| Reducing powers | a^b mod n = a^{b mod φ(n)} mod n |

---

## 🔹 Step 7: Summary 💡

| Concept | Formula | Time |
|----------|----------|------|
| Prime generation | sieve | O(n log log n) |
| Totient precomputation | φ[i] -= φ[i]/p for primes p | O(n log log n) |
| Use case | modular inverse, cryptography, combinatorics | fast |

---

## 🔹 Step 8: Takeaway

Ek hi sieve-based precomputation se tum:
- Saare primes nikal sakte ho  
- φ(n) sab numbers ke liye precompute kar sakte ho  
- Modular inverse aur exponentiation turbo speed me solve kar sakte ho 🚀  
