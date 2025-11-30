---
layout: post
title: "LeetCode Weekly Contest 478 – Q3. Minimum Mirror Pair Distance (Medium)"
date: 2025-11-30 15:30:00 +0530
categories: [DSA, LeetCode]
tags: [leetcode, contest, hashmap, string, optimization, medium]
description: "My notes and C++ solutions for LeetCode Weekly Contest 478 – Q3, starting from brute force O(n²) to optimized O(n)."
image:
  path: /assets/img/hello-world-banner.jpg
  alt: "LeetCode Weekly Contest 478 – Q3"
  lqip: /assets/img/hello-world-banner.jpg
  no_lazy: false
---

> Problem link: [LeetCode Weekly Contest 478 – Q3 (Minimum Mirror Pair Distance)](https://leetcode.com/contest/weekly-contest-478/problems/minimum-absolute-distance-between-mirror-pairs/)

---

### 🧩 Problem Summary

- Tumhe ek integer array `nums` diya gaya hai.  
- Har element ka **mirror** value uska reverse number hai. (e.g. `123` → `321`)  
- Tumhe aisa pair `(i, j)` (with `i < j`) find karna hai jisme `reverse(nums[i]) == nums[j]`.  
- Return karo **minimum distance** `(j - i)` of such a pair.  
- Agar koi mirror pair exist nahi karta, return `-1`.

---

### 🐢 Brute Force Approach – O(n²)

Idea:  
- Sab numbers ko string me convert karo.  
- Har index `i` ke liye string reverse karo aur int me convert karo.  
- Phir `j = i + 1 ... n-1` tak check karo koi `nums[j]` us reverse ke equal hai ya nahi.  
- Distance `j - i` ka minimum store karo.

```cpp
class Solution {
public:
    int minMirrorPairDistance(vector<int>& nums) {
        int mini = INT_MAX;
        vector<string> sn;

        for (auto n : nums) {
            sn.push_back(to_string(n));
        }

        for (int i = 0; i < (int)sn.size(); i++) {
            reverse(sn[i].begin(), sn[i].end());
            int a = stoi(sn[i]);

            for (int j = i + 1; j < (int)sn.size(); j++) {
                int b = stoi(sn[j]);
                if (a == b) {
                    mini = min(mini, j - i);
                    break;
                }
            }
        }

        return mini == INT_MAX ? -1 : mini;
    }
};
```

---

### ⏱️ Complexity

- **Time:** `O(n^2)` (due to sorting)
- **Space:** `O(n)` (no extra data structure)

---

### Optimized Approach – O(n) Using Hash Map

Key Idea:

- Har number ka last index store karte chalo ek hash map me:
  - value -> last seen index

- Current index i pe:

  - `rev = reverseInt(nums[i])` nikaalo.

  - Agar rev pehle dekha hai (map me hai), to `answer = min(answer, i - lastIndex[rev])`.

  - Phir `lastIndex[nums[i]] = i` update kar do.

---

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    int reverseInt(int x) {
        int rev = 0;
        while (x > 0) {
            rev = rev * 10 + x % 10;
            x /= 10;
        }
        return rev;
    }

    int minMirrorPairDistance(vector<int>& nums) {
        unordered_map<int, int> lastIndex; // value -> last index
        int mini = INT_MAX;

        for (int i = 0; i < (int)nums.size(); i++) {
            int rev = reverseInt(nums[i]);

            // check if reverse value appeared before
            if (lastIndex.count(rev)) {
                mini = min(mini, i - lastIndex[rev]);
            }

            // update current value's last index
            lastIndex[nums[i]] = i;
        }

        return mini == INT_MAX ? -1 : mini;
    }
};
```

---

### ⏱️ Complexity

- **Time:** `O(n)` average (`unordered_map` operations `O(1)` avg)
- **Space:** `O(n)` 

**Note**: Agar `unordered_map` ko `map` se replace karoge to time complexity `O(n log n)` ho jaayegi.

---





