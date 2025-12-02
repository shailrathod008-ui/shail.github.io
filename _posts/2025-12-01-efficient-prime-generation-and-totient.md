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

## 🔹 Mathematical Proofs (Beginner-friendly, with math)


### 1. Why the Sieve of Eratosthenes works (short proof)

Claim: after running the sieve up to $N$, an index $n$ is unmarked iff $n$ is prime.

Proof idea (one line): every composite $n$ has a smallest prime factor $p \le \sqrt{n}$, so when the algorithm processes $p$ it marks $n$.

More formally:
- If $n$ is composite then $\exists$ prime $p$ with $p\mid n$ and $p\le\sqrt{n}$. The sieve marks multiples of $p$ starting at $p^2$, and since $p\le\sqrt{n}$ we have $p^2\le n$, so $n$ will be marked by $p$ (or by a smaller prime).
- If $n$ is prime no smaller prime divides it, so it is never marked.

Thus unmarked numbers are exactly the primes.

Small-step run (2..30) — show remaining unmarked after key steps:

Start: $2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30$

After $p=2$ (mark multiples of 2): $2,3,\,[4],5,[6],7,[8],9,[10],11,[12],13,[14],15,[16],17,[18],19,[20],21,[22],23,[24],25,[26],27,[28],29,[30]$

After $p=3$ (mark multiples of 3): $2,3,5,7,[9],11,[15],13,[21],17,19,[27],23,25,29$

After $p=5$ (mark multiples of 5 starting at $5^2=25$): $2,3,5,7,11,13,17,19,23,[25],29$

Stop when $p^2>N$ (here $7^2=49>30$). Final primes $\le 30$: $2,3,5,7,11,13,17,19,23,29$.

---

### 2. Why $\displaystyle \phi(n)=n\prod_{p\mid n}\left(1-\frac{1}{p}\right)$ (intuition + examples)

Idea: $\phi(n)$ counts integers $1\le k\le n$ that are not divisible by any prime factor of $n$. Each distinct prime factor $p$ removes a fraction $1/p$ of numbers; the remaining fraction multiplies.

Key facts:
- For a prime power $p^a$: $\phi(p^a)=p^a-p^{a-1}=p^a\left(1-\frac{1}{p}\right)$.
- $\phi$ is multiplicative on coprime arguments: if $\gcd(x,y)=1$ then $\phi(xy)=\phi(x)\phi(y)$. Combine to get the product formula.

Formula:
$$
\phi(n)=n\prod_{p\mid n}\left(1-\frac{1}{p}\right).
$$

Examples:
- $n=10=2\cdot5$: $\displaystyle \phi(10)=10\Big(1-\tfrac{1}{2}\Big)\Big(1-\tfrac{1}{5}\Big)=10\cdot\tfrac{1}{2}\cdot\tfrac{4}{5}=4$. Coprime numbers: $\{1,3,7,9\}$.
- $n=13$ (prime): $\displaystyle \phi(13)=13\Big(1-\tfrac{1}{13}\Big)=12$.
- $n=36=2^2\cdot3^2$: $\displaystyle \phi(36)=36\Big(1-\tfrac{1}{2}\Big)\Big(1-\tfrac{1}{3}\Big)=36\cdot\tfrac{1}{2}\cdot\tfrac{2}{3}=12$.

---

### 3. Why the Totient Sieve is correct (short invariant + examples)

Algorithm invariant: initialize $\phi[i]=i$. For each prime $p$ and each multiple $j$ of $p$ do
$$
\phi[j]\leftarrow \phi[j]-\frac{\phi[j]}{p}.
$$
Each distinct prime factor $p$ of $j$ effectively multiplies $\phi[j]$ by $(1-\tfrac{1}{p})$. After processing all distinct primes dividing $j$ we obtain
$$
\phi[j]=j\prod_{p\mid j}\left(1-\frac{1}{p}\right),
$$
which is Euler's formula.

Small updates (demonstration):
- $j=10$:
  - start: $\phi[10]=10$
  - after $p=2$: $\phi[10]=10-10/2=5$
  - after $p=5$: $\phi[10]=5-5/5=4$
- $j=36$:
  - start: $\phi[36]=36$
  - after $p=2$: $\phi[36]=36-36/2=18$
  - after $p=3$: $\phi[36]=18-18/3=12$
- $j=13$ (prime):
  - start: $\phi[13]=13$
  - after $p=13$: $\phi[13]=13-13/13=12$

These updates match the product formula and are what the sieve implements.

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
