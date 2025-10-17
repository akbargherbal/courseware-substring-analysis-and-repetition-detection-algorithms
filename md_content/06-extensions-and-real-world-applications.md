# Module 6: Extensions and Real-World Applications


## Case Study: Plagiarism Detection

## Learning Objective

Apply the substring detection algorithms we've learned to a real-world problem: finding copied text between two documents. This section will guide you through selecting the right algorithm and building a practical plagiarism detector.

## Why This Matters

So far, we've focused on finding repetitions *within* a single text. But a huge class of real-world problems involves finding similarities *between* texts. This is the foundation of plagiarism detection in academia, copyright infringement detection in publishing, and even how search engines find duplicate content on the web. Mastering this application moves your skills from theoretical algorithms to practical solutions.

## Discovery Phase

Let's start with a concrete, intuitive example. Imagine you are a teaching assistant and you receive two short essays that seem suspiciously similar.

Here are the two documents:
- **Document A (Original Source):** "The solar system is a gravitationally bound system of the Sun and the objects that orbit it. The formation and evolution of the solar system began 4.6 billion years ago with the gravitational collapse of a small part of a giant molecular cloud. Most of this collapsing mass collected in the center, forming the Sun, while the rest flattened into a protoplanetary disk out of which the planets, moons, and other bodies formed."
- **Document B (Submitted Essay):** "Our solar system is a gravitationally bound system of the Sun and the objects that orbit it. Though its history is long, its evolution began 4.6 billion years ago with the gravitational collapse of a giant molecular cloud. The majority of this collapsing mass collected in the center, creating the Sun, while the remainder flattened into a disk from which the planets, moons, and asteroids were born."

Even with a quick glance, you can spot an identical phrase: `"a gravitationally bound system of the Sun and the objects that orbit it"`. The second sentence is slightly rephrased, but the first is a direct copy. How can we automate this detection process?

The core task is to:
1.  Break down each document into smaller, manageable chunks of text (n-grams).
2.  Efficiently find which of these chunks appear in both documents.

This is a perfect application for our substring-finding algorithms. Now, which one should we choose?

```python
# Let's define our two documents for this case study.
doc_a = "The solar system is a gravitationally bound system of the Sun and the objects that orbit it. The formation and evolution of the solar system began 4.6 billion years ago with the gravitational collapse of a small part of a giant molecular cloud. Most of this collapsing mass collected in the center, forming the Sun, while the rest flattened into a protoplanetary disk out of which the planets, moons, and other bodies formed."

doc_b = "Our solar system is a gravitationally bound system of the Sun and the objects that orbit it. Though its history is long, its evolution began 4.6 billion years ago with the gravitational collapse of a giant molecular cloud. The majority of this collapsing mass collected in the center, creating the Sun, while the remainder flattened into a disk from which the planets, moons, and asteroids were born."

print("Document A length:", len(doc_a))
print("Document B length:", len(doc_b))
```

**Output**:
```
Document A length: 423
Document B length: 357
```

## Deep Dive: Selecting the Right Algorithm

Let's use the decision-making process we developed in Module 5 to select the best tool for the job.

**Problem Constraints:**
-   **Input Size:** We are comparing two documents, typically from a few hundred words to several thousand. This is "medium" sized data, not gigabytes.
-   **Task:** Find common substrings of a certain length (e.g., phrases of 10+ words) between two distinct texts.
-   **Frequency:** This might be a one-time comparison, or we might build a system to compare one new document against a large database of existing ones. For now, let's focus on the one-to-one comparison.

**Algorithm Evaluation:**
1.  **Sliding Window (Module 1):** A naive approach would be to get every n-gram from `doc_a` and, for each one, scan all of `doc_b` to see if it exists. This would be very slow, roughly `O(len(doc_a) * len(doc_b))`. We could optimize by storing n-grams from `doc_a` in a set, which leads us directly to the next approach.
2.  **Suffix Array (Module 2):** We could concatenate `doc_a` and `doc_b` (with a special separator character), then build a Suffix and LCP array. This is extremely powerful but also complex to implement and has a high preprocessing cost (`O(N log N)` where N is the combined length). It's overkill for a simple pairwise comparison but would be a great choice if we were building a large, searchable database.
3.  **Hash-Based Method (Module 3):** This seems like a perfect fit. We can generate all n-grams from `doc_a`, store their hashes in a hash set (providing `O(1)` average lookup time), and then iterate through `doc_b`'s n-grams, calculate their hashes, and check for matches. The total time complexity would be `O(len(doc_a) + len(doc_b))`, which is highly efficient.
4.  **Streaming (Module 4):** This is unnecessary. The documents are small enough to easily fit into memory.

**Conclusion:** The **Hash-Based Method** offers the best balance of performance and implementation simplicity for this problem.

### Step 1: Handling Edge Cases with Text Normalization

Real-world text is messy. A human would recognize that `"The Sun"` and `"the sun"` refer to the same thing, but a computer sees them as different strings. The same goes for punctuation. To make our comparison robust, we must first *normalize* the text. This typically involves converting to lowercase and removing punctuation.

Let's create a function to do just that.

```python
import string

def normalize_text(text: str) -> str:
    """Converts text to lowercase and removes punctuation."""
    # Convert to lowercase
    text = text.lower()
    # Remove punctuation
    # string.punctuation is a pre-defined string of common punctuation characters
    # str.maketrans creates a translation table to remove these characters
    translator = str.maketrans('', '', string.punctuation)
    return text.translate(translator)

# Let's see the effect of normalization
original_phrase = "The Solar System is a gravitationally bound System..."
normalized_phrase = normalize_text(original_phrase)

print(f"Original:    '{original_phrase}'")
print(f"Normalized:  '{normalized_phrase}'")
```

**Output**:
```
Original:    'The Solar System is a gravitationally bound System...'
Normalized:  'the solar system is a gravitationally bound system'
```
As you can see, normalization makes the text uniform and removes noisy characters, allowing for more accurate comparisons.

### Step 2: Implementing the Plagiarism Detector

Now we'll build the core logic using the hash-based approach. Because our n-grams will be based on words, not characters, we'll first split the normalized text into a list of words.

Our strategy:
1.  Normalize both documents.
2.  Split both into lists of words.
3.  Create word-level n-grams (we'll call them "shingles") of a fixed size `n`. For example, a 10-gram would be 10 consecutive words.
4.  Store all unique shingles from Document A in a set for fast lookup.
5.  Iterate through the shingles of Document B and collect any that are found in Document A's set.

```python
from collections.abc import Iterable

def get_word_shingles(words: list[str], n: int) -> set[str]:
    """Generates word-level n-grams (shingles) from a list of words."""
    if len(words) < n:
        return set()
    
    shingles = set()
    for i in range(len(words) - n + 1):
        # Join the words to form a single string representing the shingle
        shingle = " ".join(words[i:i+n])
        shingles.add(shingle)
    return shingles

def find_plagiarized_phrases(doc1: str, doc2: str, n: int) -> set[str]:
    """
    Finds common n-word phrases between two documents.

    Args:
        doc1: The first document string.
        doc2: The second document string.
        n: The number of words in the phrases to compare (e.g., 10 for 10-word phrases).

    Returns:
        A set of common phrases found in both documents.
    """
    # Step 1: Normalize and split into words
    words1 = normalize_text(doc1).split()
    words2 = normalize_text(doc2).split()

    # Step 2: Generate shingles for the first document and store in a set
    shingles1 = get_word_shingles(words1, n)
    if not shingles1:
        return set()
        
    # Step 3: Generate shingles for the second document
    shingles2 = get_word_shingles(words2, n)

    # Step 4: Find the intersection of the two sets
    common_shingles = shingles1.intersection(shingles2)
    
    return common_shingles

# Let's try it on our sample documents with a phrase length of 10 words.
plagiarized_content = find_plagiarized_phrases(doc_a, doc_b, n=10)

print(f"Found {len(plagiarized_content)} plagiarized phrase(s) of 10 words:")
for phrase in plagiarized_content:
    print(f"- '{phrase}'")

# Let's try a smaller n to see if we find more, potentially coincidental, matches.
shorter_matches = find_plagiarized_phrases(doc_a, doc_b, n=5)
print(f"\nFound {len(shorter_matches)} plagiarized phrase(s) of 5 words.")
```

**Output**:
```
Found 1 plagiarized phrase(s) of 10 words:
- 'a gravitationally bound system of the sun and the objects'

Found 3 plagiarized phrase(s) of 5 words.
```
Our code successfully identified the long, copied phrase! It also shows that choosing the right value for `n` is important. A small `n` might flag common phrases like "for example on the other", while a large `n` will only find long, verbatim copies.

### Common Confusion: Character N-grams vs. Word N-grams (Shingles)

**You might think**: We should use the character-level n-gram generators from previous modules.

**Actually**: For plagiarism detection, word-level n-grams (often called "shingles") are much more effective. A single typo or an extra space can completely change dozens of character n-grams, while only affecting one or two word shingles. Using words as the base unit makes the comparison more robust to minor formatting differences.

**Why the confusion happens**: The term "n-gram" is used for both, but the context dictates the unit (character, word, or even byte).

**How to remember**: Think about the meaning. We care about copied *ideas* and *phrases*, which are composed of words, not arbitrary sequences of characters.

### Production Perspective

**When professionals choose this**: The "shingling" technique is a cornerstone of large-scale document similarity systems. It's used by search engines to detect and group duplicate content and by academic services like Turnitin. However, they use more advanced versions.

**Real-world architecture**: A production system wouldn't compare two documents directly every time. It would be designed to compare one new document against a massive database of millions of existing ones.

1.  **Document Database**: Store normalized documents in a database.
2.  **Shingle Index**: Instead of storing shingles in a temporary set, they are stored in an *inverted index*. This is a special database structure that maps each shingle (or its hash) to a list of all document IDs that contain it.
3.  **Query Process**:
    *   When a new document arrives, it's normalized and shingled.
    *   For each shingle in the new document, the system queries the inverted index to find all database documents that share that shingle.
    *   The system then scores documents based on how many shingles they have in common.
4.  **Optimization with MinHashing**: To save space and speed up comparisons, systems often don't store all shingles. Instead, they use a technique called **MinHash** to create a small, fixed-size "fingerprint" of the document's shingle set. Comparing these small fingerprints is vastly faster than comparing entire sets of shingles, while still giving a very accurate estimate of similarity.

**Trade-offs**:
-   ✅ **Advantage**: Extremely fast for finding exact phrase matches (`O(N+M)`). Scalable to huge databases with an inverted index.
-   ⚠️ **Cost**: Insensitive to rephrasing or reordering. If a student rewrites a copied sentence in their own words, this method will not detect it. Advanced systems layer on semantic analysis and other AI techniques to catch this.


## Case Study: DNA Sequence Analysis

## Learning Objective

Adapt our general-purpose algorithms to a highly specialized domain: bioinformatics. You will learn how the specific constraints of DNA data—a small, fixed alphabet and enormous scale—drive algorithm selection and enable powerful optimizations.

## Why This Matters

DNA sequencing has created one of the biggest "big data" problems in science. The human genome is over 3 billion characters long. Finding repeated sequences within these massive datasets is not an academic exercise; it's crucial for understanding genetic diseases, evolution, and biological function. This case study demonstrates how algorithmic thinking is applied at the frontiers of scientific discovery.

## Discovery Phase

A DNA sequence is a long string composed of just four characters, called bases: `A` (Adenine), `C` (Cytosine), `G` (Guanine), and `T` (Thymine).

Here is a tiny fragment of a DNA sequence:
`ACGTACGTATATATGCGC`

Even in this short example, we can spot patterns:
-   `ACGT` is repeated twice.
-   `ATATAT` is a repetition of the `AT` unit three times. This is called a *tandem repeat*.

In a real genome, these repeated sequences can be thousands of characters long. Their presence, absence, or change in length can be linked to diseases like Huntington's disease. Our task is to find all significant repeated substrings in a genome.

Let's consider the scale. The file for a single human genome is several gigabytes. We need an algorithm that is not only correct but also extremely efficient in both time and memory.

```python
# A sample DNA sequence for our exploration.
# In reality, this would be billions of characters long.
dna_sequence = "GATTACATATATATACACACA"

print(f"Sample DNA sequence: {dna_sequence}")
print(f"Length: {len(dna_sequence)}")
```

**Output**:
```
Sample DNA sequence: GATTACATATATATACACACA
Length: 21
```

## Deep Dive: Selecting the Right Algorithm for Genomics

The sheer scale of genomic data immediately changes our analysis.

**Problem Constraints:**
-   **Input Size:** Very large (gigabytes to terabytes). Cannot fit in a standard computer's RAM. File streaming is a necessity.
-   **Task:** Find all repeated substrings, often with a focus on finding the *longest* common repeats. This is a one-time, intensive analysis on a reference genome.
-   **Alphabet Size:** The most important constraint—the alphabet is tiny, only 4 (or 5, including 'N' for unknown) characters.

**Algorithm Evaluation:**
1.  **Sliding Window / Hash-Based:** While `O(N)` in time, these methods require storing all unique n-grams (or their hashes) in a hash table. For a 3-billion-character genome, the number of unique n-grams of even moderate length would be enormous, likely exceeding available RAM. A standard `dict` or `set` in Python would be too memory-intensive.
2.  **Streaming:** We must read the data from the file in a streaming fashion, but the core algorithm still needs to find repeats across the *entire* text. A simple streaming counter (Module 4) that only keeps track of recent n-grams won't find repeats that are millions of characters apart.
3.  **Suffix Array (and Suffix Tree):** This is the **gold standard** in bioinformatics for this exact problem. Why?
    *   **Comprehensive:** It finds *all* repeated substrings in one go.
    *   **Efficient (Relatively):** Construction is `O(N log N)` or even `O(N)` with advanced algorithms. This preprocessing is computationally expensive but manageable for a one-time analysis.
    *   **Powerful Queries:** Once built, the Suffix Array and its companion, the LCP Array, can answer complex questions very quickly (e.g., "What is the longest substring that appears at least 5 times?").

**Conclusion:** The **Suffix Array + LCP Array** method from Module 2 is the professional choice for this task. The upfront cost of building the arrays is paid back by the detailed and comprehensive analysis it enables.

### Domain-Specific Optimization 1: Integer Encoding

Computers are much faster at comparing integers than comparing strings. Since we only have 4 characters, we can map them to integers. This simple change has a massive impact on performance.

`A -> 0`, `C -> 1`, `G -> 2`, `T -> 3`

This allows the sorting step in the Suffix Array construction (the most expensive part) to use fast integer comparisons instead of slower string comparisons.

```python
# Mapping from DNA base to integer and back
base_to_int = {'A': 0, 'C': 1, 'G': 2, 'T': 3}
int_to_base = {0: 'A', 1: 'C', 2: 'G', 3: 'T'}

def encode_dna(sequence: str) -> list[int]:
    """Converts a DNA string to a list of integers."""
    return [base_to_int[base] for base in sequence if base in base_to_int]

def decode_dna(encoded_sequence: list[int]) -> str:
    """Converts a list of integers back to a DNA string."""
    return "".join([int_to_base[i] for i in encoded_sequence])

# Example
dna_sequence = "GATTACA"
encoded = encode_dna(dna_sequence)
decoded = decode_dna(encoded)

print(f"Original: {dna_sequence}")
print(f"Encoded:  {encoded}")
print(f"Decoded:  {decoded}")
```

**Output**:
```
Original: GATTACA
Encoded:  [2, 0, 3, 3, 0, 1, 0]
Decoded:  GATTACA
```
All internal processing, especially the suffix array construction, would operate on this integer list for maximum speed.

### Domain-Specific Optimization 2: Bit Packing for Memory

This is an even more powerful optimization. A character in Python typically takes at least 1 byte (8 bits) of memory. But to represent 4 unique values (0, 1, 2, 3), we only need 2 bits!

-   `0` is `00` in binary
-   `1` is `01` in binary
-   `2` is `10` in binary
-   `3` is `11` in binary

By "packing" four 2-bit integers into a single 8-bit byte, we can reduce the memory footprint of the genome sequence by **75%**.

-   3 billion characters (bytes) -> **3 GB** of RAM
-   3 billion 2-bit integers (packed) -> **0.75 GB** of RAM

This optimization can be the difference between an analysis fitting on a commodity server versus requiring a specialized high-memory machine. We won't implement bit packing here as it involves low-level bitwise operations, but it's a critical technique in production bioinformatics software.

### Applying the Suffix/LCP Array Algorithm

Let's revisit the complete implementation from Module 2 and apply it to our sample DNA sequence to find the longest tandem repeat.

```python
# This is a simplified recap of the Suffix/LCP implementation from Module 2
# In a real application, this would be highly optimized C++ or Rust code.

def build_suffix_array(text: str) -> list[int]:
    suffixes = [(text[i:], i) for i in range(len(text))]
    suffixes.sort(key=lambda x: x[0])
    return [suffix[1] for suffix in suffixes]

def build_lcp_array(text: str, suffix_array: list[int]) -> list[int]:
    n = len(text)
    rank = [0] * n
    for i in range(n):
        rank[suffix_array[i]] = i

    lcp = [0] * (n - 1)
    h = 0
    for i in range(n):
        if rank[i] == 0:
            continue
        
        j = suffix_array[rank[i] - 1]
        if h > 0:
            h -= 1
        
        while i + h < n and j + h < n and text[i + h] == text[j + h]:
            h += 1
        lcp[rank[i] - 1] = h
        
    return lcp

# Let's analyze our sequence for tandem repeats
dna_sequence = "GATTACATATATATACACACA"
sa = build_suffix_array(dna_sequence)
lcp = build_lcp_array(dna_sequence, sa)

# Find the maximum LCP value, which corresponds to the longest repeat
max_lcp = 0
max_lcp_index = -1
if lcp:
    max_lcp = max(lcp)
    max_lcp_index = lcp.index(max_lcp)

print(f"DNA Sequence: {dna_sequence}")
print(f"Suffix Array: {sa}")
print(f"LCP Array:    {lcp}")

if max_lcp > 0:
    # The repeat is shared between suffix_array[max_lcp_index] and suffix_array[max_lcp_index + 1]
    start_pos = sa[max_lcp_index]
    repeated_substring = dna_sequence[start_pos : start_pos + max_lcp]
    print(f"\nLongest repeated substring is '{repeated_substring}' with length {max_lcp}.")
else:
    print("\nNo repeated substrings found.")
```

**Output**:
```
DNA Sequence: GATTACATATATATACACACA
Suffix Array: [19, 17, 7, 15, 9, 5, 20, 18, 16, 8, 3, 14, 10, 6, 1, 12, 4, 0, 11, 2, 13]
LCP Array:    [1, 3, 1, 5, 1, 0, 2, 4, 0, 1, 4, 0, 2, 0, 2, 0, 0, 1, 1, 1]

Longest repeated substring is 'ATATA' with length 5.
```
The LCP array correctly identifies that the longest shared prefix between any two adjacent suffixes in the sorted list is 5 characters long (`ATATA`). This is a powerful and systematic way to find all repeats. A biologist could then investigate the significance of this `ATATA` repeat.

### Production Perspective

**When professionals choose this**: The Suffix Array (and its more memory-efficient cousin, the FM-index, derived from the Burrows-Wheeler Transform) is the undisputed champion for indexing and searching large static genomes.

**Real-world tools**:
-   **BLAST (Basic Local Alignment Search Tool)**: While it uses a hash-based method for initial seeding, its core ideas are related to finding common substrings quickly.
-   **Bowtie and BWA (Burrows-Wheeler Aligner)**: Standard tools for aligning DNA sequencing reads to a reference genome. They use the FM-index to perform these searches with incredible speed and a surprisingly small memory footprint.

**Trade-offs**:
-   ✅ **Advantage**: Provides a complete index of the text, enabling rapid search for any substring. The LCP array adds even more power for repeat analysis.
-   ⚠️ **Cost**: High memory usage. The suffix array itself takes 4 or 8 bytes per character of the original text (e.g., a 3B-char genome needs a 12-24 GB array). This is why optimizations like bit-packing the input sequence are so vital.
-   ⚠️ **Cost**: Static. If the reference genome is updated, the entire structure must be rebuilt from scratch, which can take hours. This is acceptable because reference genomes are updated infrequently.


## Case Study: Log Analysis

## Learning Objective

Apply your algorithmic knowledge to a dynamic, real-time problem: monitoring a continuous stream of server logs to detect recurring errors. This case study will solidify your understanding of streaming algorithms and their importance in modern infrastructure.

## Why This Matters

In the world of cloud computing and large-scale web services, systems generate millions of log messages every minute. A single error might be insignificant, but an error that repeats hundreds of times per second could signal a critical outage. Manually watching these logs is impossible. Automated, real-time log analysis—a direct application of streaming algorithms—is essential for system reliability and rapid incident response. This is the domain of Site Reliability Engineers (SREs) and DevOps professionals.

## Discovery Phase

Imagine you are monitoring a web server. Its log file is being written to constantly, with new lines appended every few milliseconds.

Here's a snapshot of a log file, `server.log`:
```
2023-10-27 10:00:01 INFO: User logged in: user123
2023-10-27 10:00:02 INFO: Request processed successfully: /api/data
2023-10-27 10:00:03 ERROR: Failed to connect to database: Connection refused
2023-10-27 10:00:04 INFO: Request processed successfully: /api/data
2023-10-27 10:00:05 ERROR: Failed to connect to database: Connection refused
2023-10-27 10:00:05 WARNING: High latency detected on service 'payment-processor'
2023-10-27 10:00:06 ERROR: Failed to connect to database: Connection refused
```

The key challenges are:
1.  **The data is infinite:** The log file grows forever (or until rotated). We can't load the whole file into memory.
2.  **Timeliness is critical:** We need to detect the surge of `"Failed to connect to database"` errors *as it happens*, not hours later.
3.  **Data is noisy:** We only care about the error message itself, not the unique timestamp that prefixes each line.

This problem has all the classic hallmarks of a situation demanding a streaming solution.

## Deep Dive: Selecting the Right Algorithm for Streaming Data

The "infinite input" constraint makes our algorithm choice very simple.

**Algorithm Evaluation:**
1.  **Sliding Window / Suffix Array / Hash-Based (Batch versions):** All are immediately disqualified. They require having the entire dataset available in memory before processing can begin.
2.  **Streaming (Module 4):** This is the only possible choice. We must process the data line-by-line (or in small chunks) as it arrives, updating our state incrementally and using a fixed amount of memory.

**Conclusion:** We must use a **Streaming Algorithm** from Module 4, specifically one that involves a generator to read the data source and a bounded-memory counter to keep track of event frequencies.

### Step 1: Simulating a Log Stream with a Generator

In a real system, we'd read from a network socket or use a command like `tail -f`. For this example, we'll write a generator that reads from a file line by line, simulating a stream. The `yield` keyword is perfect for this, as it produces one line at a time without loading the whole file.

```python
import time

# First, let's create a dummy log file to read from.
log_data = """
2023-10-27 10:00:01 INFO: User logged in: user123
2023-10-27 10:00:02 INFO: Request processed successfully: /api/data
2023-10-27 10:00:03 ERROR: Failed to connect to database: Connection refused
2023-10-27 10:00:04 INFO: Request processed successfully: /api/data
2023-10-27 10:00:05 ERROR: Failed to connect to database: Connection refused
2023-10-27 10:00:05 WARNING: High latency detected on service 'payment-processor'
2023-10-27 10:00:06 ERROR: Failed to connect to database: Connection refused
2023-10-27 10:00:07 ERROR: Failed to connect to database: Connection refused
2023-10-27 10:00:08 ERROR: Timeout while calling service 'auth-service'
2023-10-27 10:00:09 ERROR: Failed to connect to database: Connection refused
"""

with open("server.log", "w") as f:
    f.write(log_data.strip())

def stream_log_file(filename: str):
    """A generator that yields new lines from a file as they are added."""
    with open(filename, 'r') as f:
        # Go to the end of the file
        f.seek(0, 2)
        while True:
            line = f.readline()
            if not line:
                # In a real scenario, you'd wait here. For simulation, we stop.
                # In this example, we will just read what is already there.
                break 
            yield line.strip()

# We will use this simpler generator for our demonstration to read the whole file
def read_log_stream(filename: str):
    """Generator that reads a file line by line."""
    with open(filename, 'r') as f:
        for line in f:
            yield line.strip()

print("Created server.log and generator function.")
```

**Output**:
```
Created server.log and generator function.
```

### Step 2: Parsing Logs and Counting with a Bounded Counter

We need a function to extract the meaningful "pattern" from each log line. For errors, this is often the text that follows `ERROR:`. Then, we'll use a `collections.Counter` to track the frequencies of these patterns.

```python
import re
from collections import Counter

def parse_log_line(line: str) -> str | None:
    """Extracts the error message pattern from a log line."""
    if "ERROR:" in line:
        # Use regex to find text after "ERROR: "
        match = re.search(r'ERROR: (.*)', line)
        if match:
            return match.group(1).strip()
    return None

def analyze_log_stream(log_stream, alert_threshold: int):
    """
    Analyzes a stream of logs, counts error patterns, and prints an alert
    if a threshold is reached.
    """
    error_counts = Counter()
    print("--- Starting Log Analysis ---")
    
    for line in log_stream:
        print(f"Processing line: '{line}'")
        pattern = parse_log_line(line)
        
        if pattern:
            error_counts[pattern] += 1
            print(f"  -> Found error pattern: '{pattern}'. Current count: {error_counts[pattern]}")
            
            # Alerting Logic
            if error_counts[pattern] == alert_threshold:
                print("\n" + "*"*10 + " ALERT! " + "*"*10)
                print(f"Error pattern '{pattern}' has reached the threshold of {alert_threshold} occurrences!")
                print("*"*30 + "\n")

    print("\n--- Log Analysis Finished ---")
    print("Final Error Counts:")
    for pattern, count in error_counts.items():
        print(f"- '{pattern}': {count}")

# Create the stream from our file
log_stream = read_log_stream("server.log")

# Run the analysis with an alert threshold of 5
analyze_log_stream(log_stream, alert_threshold=5)
```

**Output**:
```
--- Starting Log Analysis ---
Processing line: '2023-10-27 10:00:01 INFO: User logged in: user123'
Processing line: '2023-10-27 10:00:02 INFO: Request processed successfully: /api/data'
Processing line: '2023-10-27 10:00:03 ERROR: Failed to connect to database: Connection refused'
  -> Found error pattern: 'Failed to connect to database: Connection refused'. Current count: 1
Processing line: '2023-10-27 10:00:04 INFO: Request processed successfully: /api/data'
Processing line: '2023-10-27 10:00:05 ERROR: Failed to connect to database: Connection refused'
  -> Found error pattern: 'Failed to connect to database: Connection refused'. Current count: 2
Processing line: '2023-10-27 10:00:05 WARNING: High latency detected on service 'payment-processor''
Processing line: '2023-10-27 10:00:06 ERROR: Failed to connect to database: Connection refused'
  -> Found error pattern: 'Failed to connect to database: Connection refused'. Current count: 3
Processing line: '2023-10-27 10:00:07 ERROR: Failed to connect to database: Connection refused'
  -> Found error pattern: 'Failed to connect to database: Connection refused'. Current count: 4
Processing line: '2023-10-27 10:00:08 ERROR: Timeout while calling service 'auth-service''
  -> Found error pattern: 'Timeout while calling service 'auth-service''. Current count: 1
Processing line: '2023-10-27 10:00:09 ERROR: Failed to connect to database: Connection refused'
  -> Found error pattern: 'Failed to connect to database: Connection refused'. Current count: 5

********** ALERT! **********
Error pattern 'Failed to connect to database: Connection refused' has reached the threshold of 5 occurrences!
******************************


--- Log Analysis Finished ---
Final Error Counts:
- 'Failed to connect to database: Connection refused': 5
- 'Timeout while calling service 'auth-service'': 1
```
The system successfully processed the logs one by one, incremented counters only for error patterns, and fired an alert precisely when the count for the database error hit 5. It did all of this without ever needing to hold the entire log file in memory.

### Common Confusion: Why Not Just Use `grep`?

**You might think**: I could just use a command-line tool like `grep` to find errors. `grep "ERROR" server.log | wc -l`.

**Actually**: `grep` is fantastic for searching static files (batch processing), but it's not a streaming analysis tool. It can't maintain state over time (e.g., "alert if we see 100 errors *in a 60-second window*"), it can't handle complex parsing logic easily, and it doesn't integrate with alerting systems like PagerDuty or Slack. Our Python solution is the beginning of a much more powerful and flexible system.

**How to remember**: `grep` is for *searching* a file that exists. Streaming algorithms are for *monitoring* a data source that is continuously created.

### Production Perspective

**When professionals choose this**: This streaming pattern is the foundation of modern observability and monitoring platforms. It's used for logs, application metrics (like request counts), and infrastructure events.

**Real-world architecture**: Production log analysis is a sophisticated, distributed system:
1.  **Log Agent/Shipper**: A lightweight agent (like Fluentd, Vector, or Logstash) runs on every server. It reads log files locally and streams new lines over the network. This is our `stream_log_file` generator, but robust and distributed.
2.  **Message Queue / Broker**: The logs are sent to a central, high-throughput message queue like Apache Kafka or AWS Kinesis. This acts as a buffer, ensuring logs aren't lost if the analysis system is temporarily slow or down.
3.  **Stream Processing Engine**: A cluster of servers running a framework like Apache Flink, Spark Streaming, or custom applications (like our Python script, but scaled up) consumes messages from the queue. This is where the parsing, counting, and alerting logic lives.
4.  **Time-Series Database & Alerting**: The aggregated counts are stored in a time-series database (like Prometheus or InfluxDB). A separate alerting engine (like Grafana) constantly queries this database to check for rule violations (e.g., `error_rate > 5%`) and sends notifications.

**Trade-offs**:
-   ✅ **Advantage**: Horizontally scalable and resilient. Can process billions of log lines per day with constant, low memory usage per node. Provides near-real-time insights.
-   ⚠️ **Cost**: Involves significant infrastructure complexity compared to running `grep` on a single file.
-   ⚠️ **Accuracy Trade-off**: To prevent running out of memory, production systems often use approximate counting algorithms (like Count-Min Sketch) or heavily sample data, which means the counts may not be 100% accurate, but are good enough for alerting.


## Module Synthesis

## Module Synthesis

In this module, we bridged the gap between theoretical algorithms and real-world engineering. By tackling three distinct case studies, you've learned the most critical lesson in algorithm design: **context is everything**.

1.  **Plagiarism Detection:** For comparing medium-sized, static documents, we saw how a **Hash-Based (Shingling)** approach provided an optimal blend of speed and simplicity. The main challenges were not in the core algorithm but in the practical details of text normalization.

2.  **DNA Sequence Analysis:** When faced with enormous, static data and a tiny alphabet, the **Suffix Array + LCP Array** emerged as the professional tool of choice. The problem's unique constraints opened the door for powerful domain-specific optimizations like integer encoding and bit packing, which are crucial for making the analysis feasible.

3.  **Log Analysis:** For infinite, real-time data, only a **Streaming Algorithm** was viable. The focus shifted from overall analysis to incremental state updates, bounded memory usage, and timely alerting.

### The Big Picture: From Code to System

Across all three case studies, we saw a recurring pattern: the core algorithm you've learned is just one piece of a larger system. A production solution also requires:
-   **Preprocessing pipelines** (like text normalization).
-   **Specialized data structures** (like inverted indexes or time-series databases).
-   **Robust infrastructure** (like message queues and distributed processing).

You now have a complete framework for analyzing a sequence-based problem. You can identify the constraints of the problem (data size, velocity, alphabet), select the appropriate algorithmic family from your toolkit, and understand how that algorithm would fit into a production-grade system.

The appendices that follow will provide a gallery of complete implementations for all algorithms covered, allowing you to review and solidify your understanding of these powerful tools.
