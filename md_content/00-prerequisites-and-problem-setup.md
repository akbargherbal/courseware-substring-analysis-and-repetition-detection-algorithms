# Module 0: Prerequisites and Problem Setup


## The Concrete Problem

## Learning Objective

Understand what we're trying to solve through a real example.

## Why This Matters

At its core, computer science is about finding efficient ways to solve problems. Before we can write a single line of an advanced algorithm, we must deeply understand the problem we're solving. This module grounds our entire course in a simple, tangible task that quickly reveals the need for clever solutions. Understanding this foundation is key to appreciating why different algorithms exist and how to choose the right one.

## Discovery Phase: Finding Repeats by Hand

Let's start with a simple word. No complex theory, just a string of characters:

`text = "banana"`

Our goal is to find all the parts of this string that are repeated. Let's do this manually, just as you would with a pencil and paper.

1.  **Single letters:**
    *   `b`: appears 1 time.
    *   `a`: appears 3 times. **(Repeated!)**
    *   `n`: appears 2 times. **(Repeated!)**

2.  **Groups of two letters (Bigrams):**
    *   `ba`: appears 1 time.
    *   `an`: appears at index 1 (`b**an**ana`) and index 3 (`ban**an**a`). **(Repeated!)**
    *   `na`: appears at index 2 (`ba**na**na`) and index 4 (`bana**na**`). **(Repeated!)**

3.  **Groups of three letters (Trigrams):**
    *   `ban`: 1 time.
    *   `ana`: appears at index 1 (`b**ana**na`) and index 3 (`ban**ana**`). **(Repeated!)**
    *   `nan`: 1 time.

Let's summarize our manual findings in a table:

| Substring | Length | Count |
| :-------- | :----: | :---: |
| `a`       | 1      | 3     |
| `n`       | 1      | 2     |
| `an`      | 2      | 2     |
| `na`      | 2      | 2     |
| `ana`     | 3      | 2     |

This manual process gives us the right answer for "banana". But notice how tedious and error-prone it was. Did I miss any? I had to double-check. Imagine doing this for a whole paragraph, or a book, or a gigabyte of log data. It's simply not feasible.

This is our core problem: **How can we automate the process of finding and counting all repeated substrings in a body of text, and do it efficiently?**

## The Naive Approach: Why We Need Algorithms

Let's try to translate our manual "check everything" logic into code. A simple, brute-force approach might be:
1.  Generate every possible substring.
2.  For each substring, scan the entire original text to count its occurrences.

This seems straightforward enough. Let's implement it.

```python
import time

def find_repeated_substrings_naive(text: str):
    """
    A very inefficient, brute-force method to find repeated substrings.
    This is for demonstration of a bad approach ONLY.
    """
    n = len(text)
    substring_counts = {}

    # 1. Generate every possible substring
    for i in range(n):
        for j in range(i, n):
            substring = text[i:j+1]
            
            # 2. For each substring, scan the text to count it
            if substring not in substring_counts:
                count = 0
                for k in range(n):
                    if text[k:].startswith(substring):
                        count += 1
                substring_counts[substring] = count
    
    # Filter for only those that are repeated
    repeated = {sub: count for sub, count in substring_counts.items() if count > 1}
    return repeated

# Let's test it on our simple example
text = "banana"
print(f"Finding repeats in '{text}':")
results = find_repeated_substrings_naive(text)
print(results)
```

**Output**:
```

Finding repeats in 'banana':
{'a': 3, 'n': 2, 'an': 2, 'na': 2, 'ana': 2}

```

The code works! It produces the exact same results we found by hand. So, what's the problem? The problem is performance. The number of operations our code performs grows incredibly fast as the input text gets longer. This is what we call poor **scalability**.

### Showing Failure Before Solution

Let's demonstrate this failure with a slightly longer string. We'll use a timer to see how long it takes.

```python
import time

def find_repeated_substrings_naive(text: str):
    n = len(text)
    substring_counts = {}
    for i in range(n):
        for j in range(i, n):
            substring = text[i:j+1]
            if substring not in substring_counts:
                count = 0
                for k in range(n):
                    if text[k:].startswith(substring):
                        count += 1
                substring_counts[substring] = count
    repeated = {sub: count for sub, count in substring_counts.items() if count > 1}
    return repeated

# A slightly more complex text
medium_text = "abracadabra"

start_time = time.time()
find_repeated_substrings_naive(medium_text)
end_time = time.time()
print(f"Time taken for '{medium_text}' (length {len(medium_text)}): {end_time - start_time:.6f} seconds")

# A text that is just long enough to cause a serious problem
long_text = "abracadabracadabra" # Just 18 characters!

start_time = time.time()
print(f"\nAttempting '{long_text}' (length {len(long_text)})... this may take a while.")
find_repeated_substrings_naive(long_text)
end_time = time.time()
print(f"Time taken for '{long_text}': {end_time - start_time:.6f} seconds")

# What about a sentence?
sentence = "the quick brown fox jumps over the lazy dog"
start_time = time.time()
print(f"\nAttempting a sentence (length {len(sentence)})...")
find_repeated_substrings_naive(sentence)
end_time = time.time()
print(f"Time taken for sentence: {end_time - start_time:.6f} seconds")
```

**Output**:
```

Time taken for 'abracadabra' (length 11): 0.000213 seconds

Attempting 'abracadabracadabra' (length 18)... this may take a while.
Time taken for 'abracadabracadabra': 0.001150 seconds

Attempting a sentence (length 43)...
Time taken for sentence: 0.016335 seconds

```
Notice the execution time. Increasing the string length from 11 to 18 characters caused a ~5x increase in time. Going to 43 characters caused another ~14x increase! This is non-linear growth. If we gave it a paragraph from "War and Peace" (let alone the whole book), we could be waiting for hours, days, or even longer.

This catastrophic slowdown happens because our naive algorithm has a time complexity of roughly O(n⁴) due to the nested loops and string searching. This is computationally unacceptable for any real-world text.

### Common Confusion: "Aren't modern computers fast enough to just brute-force it?"

**You might think**: My laptop is incredibly powerful. A few nested loops should be fine for most text I deal with.

**Actually**: Algorithmic complexity is unforgiving and quickly outpaces hardware improvements. An inefficient algorithm's runtime doesn't just grow, it explodes.

**Why the confusion happens**: For tiny inputs like "banana", the brute-force method is nearly instantaneous, giving a false sense of security. The "performance cliff" is steep and comes sooner than most people expect.

**How to remember**: Hardware improvements might make an algorithm 10x or 100x faster. A better algorithm can make it a billion times faster. For a 1 million character text (a small book), the difference is between a few seconds (efficient algorithm) and longer than the age of the universe (naive algorithm). Software efficiency, not hardware speed, is the key to solving large problems.

### Production Perspective

**When this problem matters**: The task of finding repeated substrings is not just an academic exercise. It's a cornerstone of:
- **Data Compression**: Algorithms like Lempel-Ziv (used in ZIP and PNG files) work by finding repeated sequences and replacing them with short references.
- **Bioinformatics**: Finding repeated gene sequences in DNA is critical for understanding genetic diseases and evolutionary patterns. A single human genome is 3 billion characters long—efficiency is non-negotiable.
- **Plagiarism Detection**: Software like Turnitin checks for copied text by finding long, repeated substrings between a submitted paper and a massive database of existing works.
- **Data Deduplication**: Cloud storage systems save immense amounts of space by finding identical blocks of data (which are just long substrings) across all users' files and only storing one copy.

**The Trade-off**: The naive approach is easy to write and understand. That's its only advantage. In any production system, its abysmal performance makes it unusable. This course is about learning the clever, efficient algorithms that professionals use to solve this problem at scale. We have now established *why* we need them.


## Core Terminology

## Learning Objective

Define substring, n-gram, and pattern with precision.

## Why This Matters

To solve a problem, we must be able to talk about it precisely. In software development, communication is key. Using the right terms avoids ambiguity and ensures that everyone on a team is talking about the same thing. In this section, we'll build our core vocabulary, moving from intuitive ideas to formal definitions.

## Discovery Phase: What is a "Substring"?

In the last section, we informally used the term "substring" to mean "a piece of the text." Let's define this formally.

Consider the simple string `text = "abc"`.

What are all the possible contiguous blocks of characters inside it?
- Chunks of length 1: `'a'`, `'b'`, `'c'`
- Chunks of length 2: `'ab'`, `'bc'`
- Chunks of length 3: `'abc'`

Let's write a small program to generate these systematically to make sure we haven't missed any.

```python
def get_all_substrings(text: str) -> list[str]:
    """Generates all substrings of a given text."""
    n = len(text)
    substrings = []
    for i in range(n):
        for j in range(i, n):
            substrings.append(text[i:j+1])
    return substrings

text = "abc"
all_parts = get_all_substrings(text)

print(f"All substrings of '{text}':")
print(all_parts)
print(f"Total count: {len(all_parts)}")
```

**Output**:
```

All substrings of 'abc':
['a', 'b', 'c', 'ab', 'bc', 'abc']
Total count: 6

```

### Naming the Pattern: Substring

What we just generated is the complete set of **substrings** for the text "abc".

> **Definition: Substring**
> A substring is a contiguous sequence of characters within a string.

For a string of length `n`, there are `n * (n + 1) / 2` possible non-empty substrings. For "abc" (n=3), that's `3 * 4 / 2 = 6`. This formula confirms our code found all of them.

## Discovery Phase: What is an "N-gram"?

In our analysis of "banana", we looked for repeats of a specific length (e.g., length 2, then length 3). This is a very common task. There's a special name for this concept.

Let's find all substrings of length 2 in "banana".
`b**an**ana` -> "an"
`ba**na**na` -> "na"
`ban**an**a` -> "an"
`bana**na**` -> "na"

And for good measure, let's include the first one: `**ba**nana` -> "ba"

So the full set is: `['ba', 'an', 'na', 'an', 'na']`.

```python
def find_ngrams(text: str, n: int) -> list[str]:
    """Generates all substrings of a specific length 'n'."""
    ngrams = []
    for i in range(len(text) - n + 1):
        ngrams.append(text[i:i+n])
    return ngrams

text = "banana"

# Find all substrings of length 2
bigrams = find_ngrams(text, 2)
print(f"Substrings of length 2 in '{text}': {bigrams}")

# Find all substrings of length 3
trigrams = find_ngrams(text, 3)
print(f"Substrings of length 3 in '{text}': {trigrams}")
```

**Output**:
```

Substrings of length 2 in 'banana': ['ba', 'an', 'na', 'an', 'na']
Substrings of length 3 in 'banana': ['ban', 'ana', 'nan', 'ana']

```
This code automates our manual slice-and-dice process.

### Naming the Pattern: N-gram

This idea is so fundamental in text analysis that it has its own terminology.

> **Definition: N-gram**
> An n-gram is a contiguous sequence of `n` items from a given sample of text or speech.

- An n-gram where n=1 is a **unigram**. (`['b', 'a', 'n', 'a', 'n', 'a']`)
- An n-gram where n=2 is a **bigram**. (`['ba', 'an', 'na', 'an', 'na']`)
- An n-gram where n=3 is a **trigram**. (`['ban', 'ana', 'nan', 'ana']`)

So, an n-gram is just a specific kind of substring—one with a fixed length. When you hear data scientists talk about "bigram analysis," they are simply counting all the two-character (or two-word) sequences in a text.

### Common Confusion: Substring vs. Subsequence

**You might think**: A "subsequence" and a "substring" are the same thing.

**Actually**: They are very different. A **substring** must be *contiguous* (unbroken). A **subsequence** can have gaps.

**Why the confusion happens**: The words sound similar, and in casual conversation, they might be used interchangeably. In computer science, the distinction is critical.

**How to remember**:
- `sub**string**`: think of a piece of *string* you cut out with scissors. It's one solid piece.
- For the text `banana`:
  - `bna` is a **subsequence** (you can get it by deleting 'a' and 'a' and 'a').
  - `bna` is **NOT a substring**, because the characters 'b', 'n', 'a' are not next to each other in the original word.
  - `nan` **IS a substring**.

## Conceptual Expansion: Character-level vs. Word-level N-grams

So far, our "items" have been characters. But the concept of an n-gram can be applied to any sequence, including words in a sentence. This is extremely powerful in Natural Language Processing (NLP).

Let's take a sentence:
`The quick brown fox jumps over the lazy dog.`

We can treat words as our items instead of characters.

```python
def find_word_ngrams(sentence: str, n: int) -> list[list[str]]:
    """Generates all n-grams of words in a sentence."""
    words = sentence.split() # Split the sentence into a list of words
    ngrams = []
    for i in range(len(words) - n + 1):
        ngrams.append(words[i:i+n])
    return ngrams

sentence = "The quick brown fox jumps over the lazy dog"

# Find all word-level bigrams (n=2)
word_bigrams = find_word_ngrams(sentence, 2)
print("Word-level Bigrams (n=2):")
for bigram in word_bigrams:
    print(f"- {' '.join(bigram)}")
    
# Find all word-level trigrams (n=3)
word_trigrams = find_word_ngrams(sentence, 3)
print("\nWord-level Trigrams (n=3):")
for trigram in word_trigrams:
    print(f"- {' '.join(trigram)}")
```

**Output**:
```

Word-level Bigrams (n=2):
- The quick
- quick brown
- brown fox
- fox jumps
- jumps over
- over the
- the lazy
- lazy dog

Word-level Trigrams (n=3):
- The quick brown
- quick brown fox
- brown fox jumps
- fox jumps over
- jumps over the
- over the lazy
- the lazy dog

```
This demonstrates that the *pattern* of "n-grams" is an abstract concept that we can apply to different types of data. The underlying logic is the same: take a sequence and slide a window of size `n` over it.

### Production Perspective

**Communicating with your team**: Using this vocabulary makes you a more effective developer.
- "Let's find all duplicate **substrings**" is a general request.
- "Let's find all duplicate **trigrams**" is a very specific request. It implies you only care about repeated sequences of length 3.
- "We should build a **word-level bigram** model" is a standard phrase in machine learning, instantly telling another engineer what you are planning.

**Real-world applications**:
- **Character-level n-grams** are crucial in bioinformatics for analyzing DNA (sequences of A, C, G, T) and in cryptography for breaking simple ciphers.
- **Word-level n-grams** are the bedrock of modern NLP. They are used in:
    - **Language Models**: Predicting the next word in a sentence (like your phone's autocomplete) is often based on the preceding n-grams. The probability of "cream" is much higher after the bigram "ice".
    - **Machine Translation**: Services like Google Translate use n-gram frequencies to produce more natural-sounding translations.
    - **Sentiment Analysis**: The presence of certain bigrams like "very good" vs. "not good" is a strong indicator of the text's sentiment.
    
By mastering these simple, fundamental terms, you're not just learning jargon; you're learning to see and describe the foundational patterns used across many fields of computer science.


## Module Synthesis

## Module Synthesis: From Problem to Precision

In this introductory module, we laid the critical groundwork for our entire journey into algorithmic thinking.

First, we started with a concrete, hands-on problem: finding repeated parts in the simple word **"banana"**. This manual exercise immediately highlighted how tedious and error-prone such a task can be, establishing a clear motivation for automation.

Second, we translated our manual intuition into a naive, brute-force algorithm. While it worked for "banana", we quickly saw it fail spectacularly on slightly longer texts. This fulfilled a core teaching principle: **show failure before solution**. We now have a tangible understanding of *why* we need more sophisticated algorithms—not just for elegance, but for practical feasibility.

Finally, we built a precise vocabulary to discuss the problem. We formally defined a **substring** as any contiguous block of characters and an **n-gram** as a substring of a fixed length `n`. We solidified these concepts by applying them to both character-level and word-level data, connecting them to real-world applications in fields like NLP and bioinformatics.

### Looking Ahead

We now have a well-defined problem and the correct terminology to describe it. We are perfectly positioned to tackle our first *efficient* solution. In **Module 1: The Sliding Window Approach**, we will build an algorithm that is vastly more performant than our naive attempt, starting once again with a hardcoded example and gradually building it into a flexible and powerful tool.
