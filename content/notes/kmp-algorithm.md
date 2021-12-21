---
title: Knuth–Morris–Pratt algorithm
tags: [
    "string-searching",
	"string-matching",
	"algorithm",
	"notes",
]
date: 2021-12-21
draft: True
---

# Explanation

The Knuth-Morris-Pratt (KMP) algorithm is used to find the occurrences of a string, of length K, in another, longer string, of length N, with O(N) complexity.

The naive approach works by iterating over both strings one character at a time. 
If the current character in the longer string matches the first character in the short string then we continue with the next character until we either match all characters or we find a mismatching character. In the second case we would then start again from the character after the current starting character.

For example:

```
S = ABCDDCBA
W = CBA

  ABCDDCBA
0 C
1  C
2   CD
3    C
4     C
5      CBA
```

The KMP improves over this naive approach by using a pre-computed table to avoid rechecking the matches for already matched characters. This pre-computation has complexity O(k).

This pre-computed table is called the **Longest Proper Prefix which is also Suffix (LPS) array**. 

A proper prefix/suffix of a string is a prefix/suffix that is different from the string itself.

For example:

```
S = "ACA"

Prefixes = "A", "AC", "ACA" 
Proper Prefixes = "A", "AC"

Suffixes = "A", "CA", "ACA" 
Proper Suffixes = "A", "CA"

Longest Proper Suffix which is a Suffix = "A" 
```

The LPS array of some string contains at position *i* the length of such a prefix for the sub-string from *0* till *i*. 

The following animation illustrates the construction of the LPS array:

![KMP LPS Array Construction](kmp_lps_array.gif)
*Image Taken From https://towardsdatascience.com/pattern-search-with-the-knuth-morris-pratt-kmp-algorithm-8562407dba5b*

# Simplified Explanation

# Analogy
