# Module 4: Algorithm 4 - Streaming Approach

## The Memory Problem (Concrete Demonstration)

## Learning Objective

Experience memory exhaustion firsthand to understand why streaming algorithms are necessary for large-scale data processing.

## Why This Matters

In previous modules, we've treated our text data as something that can be fully loaded into memory. For "banana" or even a short story, this works perfectly. But in the real world, data can be enormous—gigabytes of log files, terabytes of genomic data, or endless streams from social media. Our previous algorithms would crash before they even start. This module introduces a new paradigm: processing data without holding it all at once.

## Discovery Phase: Scaling Up the Sliding Window

Let's revisit the simple sliding window N-gram counter from Module 1. It takes text, generates all n-grams, and stores them in a list.

First, let's confirm it works on our familiar "banana" example.

```python
from collections import Counter
import sys

def get_ngrams_list(text: str, n: int) -> list[str]:
    """Generates a list of all n-grams from the text."""
    if len(text) < n:
        return []
    return [text[i:i+n] for i in range(len(text) - n + 1)]

# A small, manageable text
text_small = "banana"
bigrams_small = get_ngrams_list(text_small, 2)
print(f"Text: '{text_small}'")
print(f"Bigrams: {bigrams_small}")
print(f"Number of bigrams: {len(bigrams_small)}")
print(f"Memory usage of the list of bigrams: {sys.getsizeof(bigrams_small)} bytes")
```

**Output**:

```
Text: 'banana'
Bigrams: ['ba', 'an', 'na', 'an', 'na']
Number of bigrams: 5
Memory usage of the list of bigrams: 104 bytes
```

This is trivial. The list of 5 bigrams takes up a mere 104 bytes. Now, let's scale up slightly. What about a text made of 1,000 "banana"s?

```python
# A medium-sized text
text_medium = "banana" * 1000  # 6,000 characters
bigrams_medium = get_ngrams_list(text_medium, 2)

print(f"Length of medium text: {len(text_medium)} characters")
print(f"Number of bigrams: {len(bigrams_medium)}")
print(f"Memory usage of the list of bigrams: {sys.getsizeof(bigrams_medium)} bytes")
# Let's see the first 10
print(f"First 10 bigrams: {bigrams_medium[:10]}")
```

**Output**:

```
Length of medium text: 6000 characters
Number of bigrams: 5999
Memory usage of the list of bigrams: 48056 bytes
First 10 bigrams: ['ba', 'an', 'na', 'an', 'na', 'ab', 'ba', 'an', 'na', 'an']
```

Still manageable. The code runs instantly, and the resulting list takes about 48 kilobytes of RAM. Our computers have gigabytes of RAM, so this is no problem.

## Deep Dive: Hitting the Memory Wall

Now for the real test. Let's try to process a text made of 10 million "banana"s. This text would be 60 million characters long. Let's not even build the list yet—let's just calculate how much memory it _would_ take.

```python
# A very large text string
# WARNING: The next line can consume ~60 MB of RAM just for the string
large_text_length = len("banana") * 10_000_000
num_bigrams = large_text_length - 2 + 1

# Let's estimate the memory usage of the list of n-grams
# An empty list starts at 56 bytes.
# Each pointer to a string in the list takes 8 bytes on a 64-bit system.
# We are creating 'num_bigrams' pointers.
estimated_list_size_bytes = 56 + (num_bigrams * 8)
estimated_list_size_gb = estimated_list_size_bytes / (1024**3)

print(f"Length of large text: {large_text_length:,} characters")
print(f"Number of bigrams to generate: {num_bigrams:,}")
print(f"Estimated memory for list of pointers: {estimated_list_size_bytes:,} bytes (~{estimated_list_size_gb:.2f} GB)")
```

**Output**:

```
Length of large text: 60,000,000 characters
Number of bigrams to generate: 59,999,999
Estimated memory for list of pointers: 479,999,992 bytes (~0.45 GB)
```

Our estimate shows that just the _pointers_ in the list would consume nearly half a gigabyte of RAM! This doesn't even count the memory for the millions of tiny string objects themselves (`'ba'`, `'an'`, `'na'`, etc.). The actual memory cost would be significantly higher, likely several gigabytes.

If we tried to run `get_ngrams_list("banana" * 10_000_000, 2)`, our program would likely slow to a crawl as the operating system struggles to find free memory, and it might eventually crash with a `MemoryError`.

This is the failure point. The approach of "generate everything, then process everything" simply does not scale. What we just saw is called **batch processing**, where the entire dataset must be available in memory before work can begin. For large-scale problems, this is a non-starter.

### Common Confusion: "Why can't my computer with 16GB of RAM handle a 500MB list?"

**You might think**: A 500MB list should be fine on a machine with 16GB of RAM. There's plenty of space!

**Actually**: The memory usage is often far more than the final size of the object. When Python builds a list, it might need to allocate, deallocate, and reallocate memory multiple times, leading to temporary memory usage spikes that can be much larger than the final list. Furthermore, your program isn't the only thing using RAM—the operating system, background services, and other applications all consume their share. A `MemoryError` happens when the OS can't grant your program's _next_ request for a chunk of memory, even if there's technically enough total RAM available but it's fragmented.

**How to remember**: Think of RAM not as a big empty box, but as a busy parking lot. Even if there are 100 empty spots (total RAM), you might not be able to park a bus (your large list) if there aren't enough contiguous spots available.

### Production Perspective

**When professionals face this**:
This isn't a theoretical problem; it's a daily reality in many fields:

- **Web Analytics**: Processing server logs that can be gigabytes per hour.
- **Natural Language Processing**: Analyzing massive text corpora like the entire English Wikipedia (over 17 GB) or Common Crawl (petabytes).
- **Bioinformatics**: Scanning DNA sequences that are billions of base pairs long.
- **Finance**: Analyzing real-time streams of stock market data.

**The shift in thinking**:
The core problem is the assumption that we can hold all our data in memory at once. The solution is to shift from batch processing to **stream processing**. Instead of loading the entire dataset, we process it piece by piece, in a "stream," ensuring that our memory usage remains small and constant, regardless of the total size of the data. The rest of this module is dedicated to building this new mental model and the tools to implement it.

## Generators vs Lists (The Core Mechanism)

## Learning Objective

Understand lazy evaluation by comparing list-based (eager) and generator-based (lazy) approaches, and learn how generators keep memory usage constant.

## Why This Matters

To solve the memory problem we just encountered, we need a mechanism that produces data _on demand_ rather than all at once. This concept is called **lazy evaluation**, and it's the heart of streaming. Python's primary tool for this is the **generator**. Understanding generators will unlock the ability to process data of virtually any size.

## Discovery Phase: Two Ways to Get N-grams

Let's start with two functions that look almost identical but behave in fundamentally different ways.

### Version 1: The Eager List

This is the same function from the last section. It builds a complete list of all n-grams in memory and then returns it. This is called **eager evaluation** because it does all the work upfront.

```python
import sys

def get_ngrams_list(text: str, n: int) -> list[str]:
    """Eagerly builds and returns a complete list of n-grams."""
    results = []
    print("Building the full list of n-grams now...")
    for i in range(len(text) - n + 1):
        results.append(text[i:i+n])
    print("...list built. Returning it.")
    return results

# Process a medium string
text_medium = "abracadabra" * 100
eager_list = get_ngrams_list(text_medium, 3)

print(f"\nType of result: {type(eager_list)}")
print(f"Memory size of the result list: {sys.getsizeof(eager_list)} bytes")
```

**Output**:

```
Building the full list of n-grams now...
...list built. Returning it.

Type of result: <class 'list'>
Memory size of the result list: 9816 bytes
```

Notice the flow: the function prints "Building...", does all the work, prints "...built", and _only then_ returns the complete list. The memory footprint is proportional to the number of n-grams.

### Version 2: The Lazy Generator

Now, look at this tiny change. We've replaced `results.append(...)` and `return results` with a single keyword: `yield`.

```python
import sys

def generate_ngrams(text: str, n: int): # The return type is now an iterator/generator
    """Lazily yields n-grams one by one."""
    print("Generator function called. Ready to produce values.")
    for i in range(len(text) - n + 1):
        # The magic happens here!
        yield text[i:i+n]
    print("...generator has finished producing all values.")

# Just calling the function doesn't run the code inside!
lazy_generator = generate_ngrams("abracadabra", 3)

print(f"\nType of result: {type(lazy_generator)}")
print(f"Memory size of the generator object: {sys.getsizeof(lazy_generator)} bytes")
```

**Output**:

```
Type of result: <class 'generator'>
Memory size of the generator object: 112 bytes
```

Look closely at the output. The "Generator function called..." message did _not_ print. When you call a function with `yield`, it doesn't run the code. Instead, it instantly returns a special `generator` object. This object is tiny (112 bytes) and acts as a "recipe" for producing the values later. It knows how to do the work, but it hasn't done any of it yet. This is lazy evaluation.

## Deep Dive: How `yield` Works

So how do we get the values out of the generator object? We ask for them, one at a time. The standard way is with a `for` loop, but to see the mechanism clearly, let's use the built-in `next()` function.

```python
lazy_generator = generate_ngrams("banana", 2)
print(f"Generator object created: {lazy_generator}\n")

# --- First call to next() ---
print("Requesting the first value...")
value1 = next(lazy_generator)
print(f"Received: '{value1}'\n")

# --- Second call to next() ---
print("Requesting the second value...")
value2 = next(lazy_generator)
print(f"Received: '{value2}'\n")

# --- We can also use it in a loop ---
print("Now, consuming the rest with a for loop:")
for remaining_value in lazy_generator:
    print(f"Received from loop: '{remaining_value}'")

# --- What happens if we ask for too many? ---
try:
    print("\nRequesting another value after it's empty...")
    next(lazy_generator)
except StopIteration:
    print("Caught StopIteration: The generator is exhausted.")
```

**Output**:

```
Generator object created: <generator object generate_ngrams at 0x10e82cba0>

Requesting the first value...
Generator function called. Ready to produce values.
Received: 'ba'

Requesting the second value...
Received: 'an'

Now, consuming the rest with a for loop:
Received from loop: 'na'
Received from loop: 'an'
Received from loop: 'na'
...generator has finished producing all values.

Requesting another value after it's empty...
Caught StopIteration: The generator is exhausted.
```

This trace reveals the magic of `yield`:

1.  **Pausing Execution**: When `next()` is called, the function runs until it hits a `yield` statement.
2.  **Returning a Value**: It "yields" the value back to the caller.
3.  **Freezing State**: The function's entire state (local variables, loop counter `i`, etc.) is frozen in place, and it pauses.
4.  **Resuming Execution**: The next time `next()` is called, the function wakes up exactly where it left off and continues running until it hits another `yield`.
5.  **Termination**: When the function finishes naturally (the loop ends), it raises a `StopIteration` signal, which `for` loops automatically handle to terminate gracefully.

This pattern is called a **Generator**. It's a fundamental concept in Python for writing memory-efficient code. By processing one item at a time, the memory usage is always just the size of one item, not all of them.

### Common Confusion: `yield` vs. `return`

**You might think**: `yield` is just a fancy way to `return` a value from a function.

**Actually**: `return` terminates a function completely. `yield` pauses the function, sends back a value, and leaves the function in a suspended state, ready to resume later. A function can `yield` many times, but it can only `return` once.

**Why the confusion happens**: Both keywords send a value from a function back to its caller, so their syntax looks similar at a glance.

**How to remember**:

- `return`: Think "return and exit." The function is done.
- `yield`: Think "yield and pause." It's like a traffic sign yielding the right-of-way; it gives control back temporarily, but it's still in the middle of its journey.

### Production Perspective

**When professionals choose this**:
Generators are not just a memory-saving trick; they are a core pattern for building decoupled, efficient data pipelines.

- **Data Pipelines**: The output of one generator can be the input to another, creating a processing chain where no large intermediate data structures are ever created. For example: `read_log_lines()` -> `extract_ip_addresses()` -> `lookup_geolocation()`. Each stage processes one item at a time.
- **Working with Streams**: When reading from network sockets, databases, or large files, the data source itself is often a stream. Generators are the natural and often only way to interact with such data.
- **Infinite Sequences**: Generators can represent sequences that are theoretically infinite, like generating all prime numbers. You simply stop asking for the next one when you have enough.
- **Improved Responsiveness**: In applications with a user interface, using a generator to process a large file can prevent the UI from freezing, as work is done in smaller chunks.

## Building a Simple Generator

## Learning Objective

Write generator functions by progressing from a hardcoded example to a flexible, parameterized version.

## Why This Matters

Knowing the theory behind generators is good, but being able to write your own is essential. In this section, we'll build a generator from scratch, reinforcing the mechanics of `yield` and the loop structure required to produce a sequence of values on demand.

## Discovery Phase: A Hardcoded Bigram Generator

Let's start with the simplest possible case: a generator that _only_ finds bigrams (n=2) in our favorite word, "banana". This removes all distracting variables and lets us focus purely on the `yield` mechanism.

```python
def generate_bigrams_for_banana():
    """A hardcoded generator that yields bigrams from 'banana'."""
    text = "banana"
    print("-> Generator started")
    for i in range(len(text) - 2 + 1):
        bigram = text[i:i+2]
        print(f"   (Inside generator: yielding '{bigram}' at index {i})")
        yield bigram
    print("-> Generator finished")

# Create the generator object
banana_bigram_gen = generate_bigrams_for_banana()

# Iterate through it to see the output
print("Starting to iterate through the generator...")
for bg in banana_bigram_gen:
    print(f"   (Outside in loop: received '{bg}')")
print("...iteration complete.")
```

**Output**:

```
Starting to iterate through the generator...
-> Generator started
   (Inside generator: yielding 'ba' at index 0)
   (Outside in loop: received 'ba')
   (Inside generator: yielding 'an' at index 1)
   (Outside in loop: received 'an')
   (Inside generator: yielding 'na' at index 2)
   (Outside in loop: received 'na')
   (Inside generator: yielding 'an' at index 3)
   (Outside in loop: received 'an')
   (Inside generator: yielding 'na' at index 4)
   (Outside in loop: received 'na')
-> Generator finished
...iteration complete.
```

This output perfectly illustrates the "conversation" between the `for` loop and the generator. The `for` loop asks for an item, the generator runs just enough to produce one, yields it, and pauses. The loop processes the item, then asks for the next, and the cycle continues.

## Deep Dive: Parameterizing the Generator

Hardcoding "banana" and n=2 is great for learning, but useless in practice. The next logical step is to make the function flexible by accepting the `text` and the n-gram size `n` as parameters. This is the exact same generalization we did for the list-based version in Module 1.

```python
def generate_ngrams(text: str, n: int):
    """A flexible generator that yields n-grams of size n from any text."""
    # The core logic is the loop bound: we must stop when there are not
    # enough characters left to form a full n-gram.
    for i in range(len(text) - n + 1):
        yield text[i:i+n]

# Now we can use the same function for different inputs
text = "banana"

print("--- Bigrams (n=2) ---")
for bigram in generate_ngrams(text, 2):
    print(bigram)

print("\n--- Trigrams (n=3) ---")
for trigram in generate_ngrams(text, 3):
    print(trigram)
```

**Output**:

```
--- Bigrams (n=2) ---
ba
an
na
an
na

--- Trigrams (n=3) ---
ban
ana
nan
ana
```

This is the production-ready version of our simple generator. It's concise, memory-efficient, and flexible. The only change was replacing the hardcoded values with parameters, a key step in moving from a specific solution to a general-purpose tool.

Let's do a quick manual trace for `generate_ngrams("banana", 3)`:

1.  **Call**: `generate_ngrams("banana", 3)` is called. A generator object is returned immediately.
2.  **`for` loop starts**: It calls `next()` on the generator.
3.  **Inside generator**: The function begins. `text` is "banana", `n` is 3. The loop starts with `i = 0`.
4.  **`yield` #1**: It yields `text[0:3]`, which is "ban". The function pauses. `i` remains 0, but will be incremented to 1 on resume.
5.  **Inside `for` loop**: The loop receives "ban" and prints it. It then requests the next item.
6.  **Inside generator**: The function resumes. The loop continues with `i = 1`.
7.  **`yield` #2**: It yields `text[1:4]`, which is "ana". The function pauses. `i` is 1.
8.  ...and so on, until the `range` is exhausted.

### Common Confusion: Off-by-One Errors in the Loop Range

**You might think**: The loop should go to `len(text) - n`. Why the `+ 1`?

**Actually**: The `range(start, stop)` function in Python goes up to, but does _not include_, the `stop` value. Let's trace the last valid index for `generate_ngrams("banana", 3)`.

- `text` has length 6. `n` is 3.
- The last possible trigram is 'ana', which starts at index 3 (`text[3:6]`).
- So, our loop variable `i` must be allowed to reach 3.
- The loop range is `len(text) - n + 1`, which is `6 - 3 + 1 = 4`.
- `range(4)` produces the indices 0, 1, 2, 3. The loop runs correctly for `i = 3` and stops before `i` becomes 4.
- If we had used `len(text) - n` (i.e., `6 - 3 = 3`), `range(3)` would only produce 0, 1, 2. We would have missed the last n-gram!

**How to remember**: The number of n-grams is `len(text) - n + 1`. Since `range(k)` gives `k` items, this is the correct formula for the `stop` argument.

### Production Perspective

**Composability**: Generators are like building blocks. In a real-world application, you might chain them together. Imagine you have a generator that reads lines from a file (`read_lines`), another that cleans them (`clean_text`), and our `generate_ngrams` function.

```python
# Conceptual example - don't run
raw_lines = read_lines("my_large_file.txt")
cleaned_lines = (clean_text(line) for line in raw_lines) # This is a generator expression!
all_ngrams = (ngram for line in cleaned_lines for ngram in generate_ngrams(line, 3))

# At this point, NO WORK has been done. The file hasn't been read.
# Only when we start iterating over `all_ngrams` does the pipeline start processing,
# one line at a time.
for ngram in all_ngrams:
    # process ngram
```

This is an incredibly powerful and memory-efficient pattern. You can build complex data processing logic that operates on huge datasets with a tiny, constant memory footprint. The code is also clean and declarative, describing the _stages_ of processing rather than the low-level mechanics of loops and temporary lists.

## Streaming N-gram Counter

## Learning Objective

Count n-gram frequencies using a generator-based streaming approach, and identify the next memory bottleneck: the counts dictionary itself.

## Why This Matters

We've successfully created a generator that produces n-grams without storing them all in memory. This is a huge step. Now, we need to use this stream to achieve our original goal from Module 1: counting n-gram frequencies. This will demonstrate how to consume a generator and reveal that solving one memory problem often exposes the next.

## Discovery Phase: Consuming the Generator

The most natural way to consume a generator is with a `for` loop. We can combine our `generate_ngrams` function with the `collections.Counter` we used in Module 1 to build a memory-efficient counter.

```python
from collections import Counter

def generate_ngrams(text: str, n: int):
    """A flexible generator that yields n-grams of size n from any text."""
    for i in range(len(text) - n + 1):
        yield text[i:i+n]

def count_ngrams_stream(text: str, n: int) -> Counter:
    """Counts n-grams by consuming a generator, not building a list."""
    counts = Counter()
    ngram_generator = generate_ngrams(text, n)

    print("Starting to count n-grams from the stream...")
    for ngram in ngram_generator:
        counts[ngram] += 1
    print("...finished counting.")

    return counts

# Let's test it on our standard example
text = "banana"
bigram_counts = count_ngrams_stream(text, 2)

print("\nFinal counts:")
print(bigram_counts)
```

**Output**:

```
Starting to count n-grams from the stream...
...finished counting.

Final counts:
Counter({'an': 2, 'na': 2, 'ba': 1})
```

The result is identical to our batch-processing method from Module 1, but the _process_ is fundamentally different. We never created a list of all 5 bigrams. Instead, we processed them one by one.

## Think Aloud: Tracing the Streaming Counter

Let's trace the execution for `count_ngrams_stream("banana", 2)` to make this crystal clear.

1.  **Initialization**: `counts` is an empty `Counter`. `ngram_generator` is created, but no code inside it has run yet.
2.  **`for` loop starts**: The loop requests the first item from `ngram_generator`.
3.  **Generator yields `'ba'`**: The generator runs, yields `'ba'`, and pauses.
4.  **Counter updates**: The loop body receives `'ba'`. `counts['ba'] += 1`. `counts` is now `Counter({'ba': 1})`. The memory for the string `'ba'` is now managed by the counter.
5.  **`for` loop continues**: The loop requests the second item.
6.  **Generator yields `'an'`**: The generator resumes, yields `'an'`, and pauses.
7.  **Counter updates**: The loop body receives `'an'`. `counts['an'] += 1`. `counts` is now `Counter({'ba': 1, 'an': 1})`.
8.  ... and so on. The key insight is that at any given moment, only **one** n-gram (the one being processed) exists in the loop's memory. The memory used for the previous n-gram is released (or garbage collected) after it's been used to update the counter.

Because we are no longer storing the _entire sequence_ of n-grams, our memory usage is significantly lower.

## Deep Dive: The Next Bottleneck

We've solved the problem of storing the stream of n-grams. But have we solved the memory problem entirely? Let's consider a different kind of input: a long string of random characters.

```python
import random
import string
import sys

# Generate 1 million random characters
random_chars = ''.join(random.choice(string.ascii_letters) for _ in range(1_000_000))

# Let's count trigrams (n=3)
# In random data, most trigrams will be unique.
trigram_counts_random = count_ngrams_stream(random_chars, 3)

print("\n--- Analysis on Random Data ---")
print(f"Total n-grams processed: {sum(trigram_counts_random.values()):,}")
print(f"Number of UNIQUE n-grams found: {len(trigram_counts_random):,}")
print(f"Memory size of the Counter object: {sys.getsizeof(trigram_counts_random):,} bytes")
```

**Output**:

```
Starting to count n-grams from the stream...
...finished counting.

--- Analysis on Random Data ---
Total n-grams processed: 999,998
Number of UNIQUE n-grams found: 140,581
Memory size of the Counter object: 4,194,312 bytes
```

Here's the new problem. While we didn't store the ~1 million trigrams in a list, we did store over 140,000 _unique_ trigrams as keys in our `Counter` dictionary. This still consumed over 4MB of RAM. If our input were a 1GB file of random data, the `Counter` object itself could grow to be gigabytes in size, and we would hit a `MemoryError` all over again.

We've shifted the memory burden from storing the raw sequence to storing the unique items and their counts. For text with a small vocabulary (like English), this is a massive improvement. But for data with high cardinality (many unique values), the counts dictionary itself becomes the new memory bottleneck.

### Common Confusion: "Why is this better if it can still run out of memory?"

**You might think**: If this streaming counter can still crash, how is it an improvement over the batch method?

**Actually**: It solves a different, and often much larger, part of the memory problem.

- **Batch method memory**: `O(L)` where L is the total length of the sequence.
- **Streaming counter memory**: `O(U)` where U is the number of unique items.

In most real-world text, the number of unique words/n-grams (U) is vastly smaller than the total length of the text (L). For the complete works of Shakespeare (about 5.5 million words), there are only about 30,000 unique words. Storing counts for 30,000 items is trivial compared to storing a list of 5.5 million items. Our streaming approach wins hugely in these common cases. It only fails when U itself becomes excessively large.

**How to remember**: The generator solves the memory problem for the _sequence's length_. The next challenge is solving it for the _sequence's variety_.

### Production Perspective

This simple streaming counter is a workhorse in data analysis.

- **Common Use Case**: It's perfect for finding word frequencies in large documents, counting occurrences of log message types, or tallying user actions from an event stream, as long as the number of unique items is expected to be manageable (thousands to a few million).
- **The Limit**: This approach hits its limit in domains like:
  - **Genomics**: Trigrams in DNA are combinations of A, C, G, T. The number of unique n-grams is small. But if `n` is large (e.g., n=20), `4^20` is over a trillion, so the number of unique n-grams can explode.
  - **Network Analysis**: Counting unique IP address pairs in traffic data could result in billions of unique pairs.
  - **Internet of Things (IoT)**: Unique sensor ID and timestamp combinations can be nearly infinite.

For these "high cardinality" problems, we need a way to count things without even storing all the unique keys. This leads us to the next section: approximate counting with bounded memory.

## Bounded Memory Counter

## Learning Objective

Limit the memory usage of a counter by implementing an eviction policy, understanding the trade-off between memory and accuracy.

## Why This Matters

We've identified the final bottleneck in our streaming pipeline: the counts dictionary itself can grow without bound when processing data with high variety. To create a truly robust streaming system, we need to guarantee that its memory footprint is _always_ fixed, regardless of the input data's size or variety. This requires a radical step: we must be willing to throw some data away and accept an _approximate_ result.

## The Problem: Unbounded Variety

Imagine analyzing trending topics on a social network. Millions of new, unique phrases appear every hour. A `Counter` trying to store them all would grow indefinitely. However, we don't care about the phrases that were used only once or twice; we only care about the "heavy hitters" that are trending. This insight is key: for many applications, we only need the _top N_ most frequent items.

This allows us to set a hard limit on our counter's size. If we want the top 10,000 trending phrases, we'll build a counter that never stores more than 10,000 items.

## Discovery Phase: An Eviction Policy

How do we keep the counter size fixed?

1.  We set a `max_size`.
2.  We add items to the counter as usual.
3.  When a **new** item arrives and the counter is already full (at `max_size`), we must make room by evicting an existing item.
4.  **Which item to evict?** To keep the most frequent items, the best candidate for eviction is the _least frequent_ item currently in the counter.

This is a "least frequent eviction" policy. Let's build a simple class to manage this logic.

```python
from collections import Counter

class BoundedCounter:
    """A Counter that holds a maximum number of items.

    When a new item is added and the counter is full, it evicts
    the least frequent item to make room.
    """
    def __init__(self, max_size: int):
        if max_size < 1:
            raise ValueError("max_size must be at least 1")
        self.counts = Counter()
        self.max_size = max_size

    def add(self, item):
        """Adds an item to the counter, performing eviction if necessary."""
        # Case 1: The item is already in the counter. Just increment.
        if item in self.counts:
            self.counts[item] += 1
            return

        # Case 2: The counter is not full. Add the new item.
        if len(self.counts) < self.max_size:
            self.counts[item] = 1
            return

        # Case 3: Counter is full and it's a new item. Evict, then add.
        # This is the "expensive" step.
        # Find the item with the minimum count.
        min_item = min(self.counts, key=self.counts.get)

        # In case of a tie, min() picks one arbitrarily.
        # It's possible the new item's count (1) is less than the min count.
        # In a more advanced implementation we might not add it, but here we will.
        del self.counts[min_item]
        self.counts[item] = 1

    def __repr__(self):
        return f"BoundedCounter({self.counts})"

# Let's trace this with a max_size of 3
bounded_counter = BoundedCounter(max_size=3)
data_stream = ['a', 'b', 'c', 'a', 'b', 'd', 'a', 'c', 'e']

print(f"Starting with max_size=3. Stream: {data_stream}\n")
for i, item in enumerate(data_stream):
    bounded_counter.add(item)
    print(f"Step {i+1}: Added '{item}'. Counter state: {bounded_counter.counts}")
```

**Output**:

```
Starting with max_size=3. Stream: ['a', 'b', 'c', 'a', 'b', 'd', 'a', 'c', 'e']

Step 1: Added 'a'. Counter state: Counter({'a': 1})
Step 2: Added 'b'. Counter state: Counter({'a': 1, 'b': 1})
Step 3: Added 'c'. Counter state: Counter({'a': 1, 'b': 1, 'c': 1})
Step 4: Added 'a'. Counter state: Counter({'a': 2, 'b': 1, 'c': 1})
Step 5: Added 'b'. Counter state: Counter({'a': 2, 'b': 2, 'c': 1})
Step 6: Added 'd'. Counter state: Counter({'a': 2, 'b': 2, 'd': 1})
Step 7: Added 'a'. Counter state: Counter({'a': 3, 'b': 2, 'd': 1})
Step 8: Added 'c'. Counter state: Counter({'a': 3, 'b': 2, 'c': 1})
Step 9: Added 'e'. Counter state: Counter({'a': 3, 'c': 1, 'e': 1})
```

## Deep Dive: Analyzing the Trace

Let's look at the critical moments in the trace:

- **Steps 1-3**: The counter is not full, so 'a', 'b', and 'c' are added normally.
- **Steps 4-5**: 'a' and 'b' are already present, so their counts are simply incremented. The size doesn't change.
- **Step 6 (First Eviction)**:
  - The new item is 'd'.
  - The counter is full (size 3) with `Counter({'a': 2, 'b': 2, 'c': 1})`.
  - The "least frequent" item is 'c', with a count of 1.
  - 'c' is evicted.
  - 'd' is added with a count of 1.
  - The new state is `Counter({'a': 2, 'b': 2, 'd': 1})`.
- **Step 8**: 'c' returns! Because it was previously evicted, it is treated as a new item. The least frequent item now is 'd' (count 1), so 'd' is evicted to make room for 'c'.
- **Step 9**: 'e' arrives. The least frequent items are 'c' and 'e' (both with count 1). `min` arbitrarily picks 'c' for eviction, and 'e' is added.

The final result is `Counter({'a': 3, 'b': 2, 'e': 1})`. The true counts were `a:3, b:2, c:2, d:1, e:1`. Our counter correctly identified 'a' and 'b' as the top two, but its count for 'c' is wrong and its count for 'd' is missing entirely. This demonstrates the trade-off in action.

### Common Confusion: "This isn't accurate! Am I losing important data?"

**You might think**: This counter gives wrong answers. If it throws away data, it seems unreliable and dangerous.

**Actually**: This is not a bug; it's a feature. This is an **approximate algorithm**. It is _designed_ to be inaccurate for low-frequency items in exchange for the guarantee of a fixed, small memory footprint.

**Why the confusion happens**: We are trained to think that programs must produce exact answers. Approximate algorithms are a different paradigm, where "close enough" is the right answer because getting a perfect answer is computationally impossible (due to memory or time constraints).

**How to remember**: A bounded counter is like a "Top 10" music chart. The chart doesn't track every song ever played; it only has room for 10 entries. Unpopular songs get kicked off the list to make room for new hits. It's not a perfect record of all music, but it's very good at telling you what's popular _right now_.

### Production Perspective

**When professionals choose this**:
Approximate counting algorithms are workhorses in large-scale data systems.

- **Use Cases**: Finding trending topics on social media, identifying popular products on an e-commerce site, detecting DDoS attacks by finding the most frequent source IP addresses. In all these cases, the goal is to find the "heavy hitters" from a massive stream, and the long tail of infrequent items is noise.
- **Performance Note**: Our `min(self.counts, key=self.counts.get)` is slow on large counters. Production systems use more advanced data structures (like a combination of a hash map and a min-heap, or specialized algorithms like Count-Min Sketch) to perform the increment and eviction steps much more efficiently. Our version is simple to demonstrate the core logic.
- **The Trade-off**: The choice to use a bounded counter is a conscious business and engineering decision. You are trading perfect accuracy for scalability and predictable performance. As long as you understand the approximation and its effects, it's an incredibly powerful tool. For example, if you over-count a trending topic by 0.1%, it's still a trending topic. If you under-count a rare item so much that it's evicted, it wasn't trending anyway.

## File Streaming (Chunk Processing)

## Learning Objective

Process files that are larger than available RAM by reading them in fixed-size chunks and correctly handling data that spans across chunk boundaries.

## Why This Matters

So far, our streaming examples have assumed the text data, although large, still fits in a single string variable in memory. This was a necessary step to understand generators. Now we'll tackle the true origin of most big data problems: files on disk that are too big to be loaded into memory with a simple `file.read()`. This is the final piece of the puzzle for processing truly massive datasets.

## The Problem: Files vs. Memory

A typical developer laptop might have 16 GB of RAM. A typical server might have 128 GB. But a single log file from a busy web service can easily be 50 GB. A genomic data file can be 100+ GB. You simply cannot read these files into memory.

The obvious solution is to read the file in small, manageable pieces, or **chunks**.

## Discovery Phase: Naive Chunking and Its Failure

Let's try the simplest approach. We can open a file and read it, for example, 32 bytes at a time, until it's empty. We'll simulate this with a string for now.

```python
from collections import Counter

def generate_ngrams(text: str, n: int):
    for i in range(len(text) - n + 1):
        yield text[i:i+n]

def process_chunk(chunk: str, n: int, counts: Counter):
    """Processes n-grams within a single chunk."""
    for ngram in generate_ngrams(chunk, n):
        counts[ngram] += 1

# Let's find trigrams (n=3) in this text
text = "alpha-beta-gamma-delta"
chunk_size = 10
counts = Counter()

print(f"Text: '{text}'")
print(f"Processing in chunks of size {chunk_size}...\n")

# Manually simulate reading two chunks
chunk1 = text[0:10]
chunk2 = text[10:20]
# And the remainder
chunk3 = text[20:]

print(f"Chunk 1: '{chunk1}'")
process_chunk(chunk1, 3, counts)
print(f"Counts after chunk 1: {counts}")

print(f"\nChunk 2: '{chunk2}'")
process_chunk(chunk2, 3, counts)
print(f"Counts after chunk 2: {counts}")

print(f"\nChunk 3: '{chunk3}'")
process_chunk(chunk3, 3, counts)
print(f"Counts after chunk 3: {counts}")

print(f"\n--- Let's compare with the correct result ---")
correct_counts = Counter(generate_ngrams(text, 3))
print(f"Correct counts: {correct_counts}")
```

**Output**:

```
Text: 'alpha-beta-gamma-delta'
Processing in chunks of size 10...

Chunk 1: 'alpha-beta'
Counts after chunk 1: Counter({'lph': 1, 'pha': 1, 'ha-': 1, 'a-b': 1, '-be': 1, 'bet': 1, 'eta': 1, 'alp': 1})

Chunk 2: '-gamma-del'
Counts after chunk 2: Counter({'lph': 1, 'pha': 1, 'ha-': 1, 'a-b': 1, '-be': 1, 'bet': 1, 'eta': 1, 'alp': 1, '-ga': 1, 'gam': 1, 'amm': 1, 'mma': 1, 'ma-': 1, 'a-d': 1, '-de': 1, 'del': 1})

Chunk 3: 'ta'
Counts after chunk 3: Counter({'lph': 1, 'pha': 1, 'ha-': 1, 'a-b': 1, '-be': 1, 'bet': 1, 'eta': 1, 'alp': 1, '-ga': 1, 'gam': 1, 'amm': 1, 'mma': 1, 'ma-': 1, 'a-d': 1, '-de': 1, 'del': 1})

--- Let's compare with the correct result ---
Correct counts: Counter({'lph': 1, 'pha': 1, 'ha-': 1, 'a-b': 1, '-be': 1, 'bet': 1, 'eta': 1, 'alp': 1, 'ta-': 1, '-ga': 1, 'gam': 1, 'amm': 1, 'mma': 1, 'ma-': 1, 'a-d': 1, '-de': 1, 'del': 1, 'elt': 1, 'lta': 1})
```

Failure! Our chunked processing missed several trigrams, like `'ta-'` and `'elt'`. Why?

- The last trigram in Chunk 1 should be `a-b`.
- The first trigram in Chunk 2 should be `-ga`.
- The trigram `'ta-'` (from `beta-gamma`) was split across the boundary between chunk 1 and chunk 2. `Chunk 1` ends with `ta` and `Chunk 2` starts with `-`. Neither chunk is long enough on its own to see the full trigram.

## Deep Dive: The Overlap Buffer Solution

To fix this, we need to handle the boundaries. The solution is to create an **overlap buffer**. When we finish processing a chunk, we must keep the last `n-1` characters. This buffer is then prepended to the _next_ chunk we read from the file. This ensures that any n-gram that spans a boundary is fully reconstructed.

Let's build a generator that correctly implements this file streaming logic.

```python
import os

def stream_file_chunks(filepath: str, n: int, chunk_size: int = 8192):
    """
    Yields chunks of text from a file with an overlap of n-1 characters
    to ensure n-grams spanning chunk boundaries are not missed.
    """
    overlap = ""
    with open(filepath, 'r', encoding='utf-8') as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                # End of file
                break

            # Prepend the overlap from the previous chunk
            block_to_process = overlap + chunk
            yield block_to_process

            # Save the last n-1 characters as the new overlap
            overlap = block_to_process[-(n-1):]

# --- Create a test file ---
test_text = "alpha-beta-gamma-delta"
test_filepath = "test_data.txt"
with open(test_filepath, 'w') as f:
    f.write(test_text)

# --- Trace the generator ---
n = 3
chunk_size = 10
print(f"--- Tracing chunk generator with n={n}, chunk_size={chunk_size} ---")
for i, chunk in enumerate(stream_file_chunks(test_filepath, n=n, chunk_size=chunk_size)):
    print(f"Yielded block {i+1}: '{chunk}'")

# --- Use it to count correctly ---
new_counts = Counter()
for block in stream_file_chunks(test_filepath, n=n, chunk_size=chunk_size):
    # This recalculates n-grams in the overlap, but Counter handles it.
    for ngram in generate_ngrams(block, n):
        new_counts[ngram] += 1

print("\n--- Final counts using overlap method ---")
print(f"Result: {new_counts}")

# Clean up the test file
os.remove(test_filepath)
```

**Output**:

```
--- Tracing chunk generator with n=3, chunk_size=10 ---
Yielded block 1: 'alpha-beta'
Yielded block 2: 'ta-gamma-del'
Yielded block 3: 'lta'

--- Final counts using overlap method ---
Result: Counter({'alp': 1, 'lph': 1, 'pha': 1, 'ha-': 1, 'a-b': 1, '-be': 1, 'bet': 1, 'eta': 1, 'ta-': 1, '-ga': 1, 'gam': 1, 'amm': 1, 'mma': 1, 'ma-': 1, 'a-d': 1, '-de': 1, 'del': 1, 'elt': 1, 'lta': 1})
```

Success! By prepending the overlap, the second yielded block became `'ta-gamma-del'`, which contains the previously missed trigram `'ta-'`. The final counts now exactly match the correct result.

_Note_: This simple implementation re-processes n-grams within the overlap region. While slightly inefficient, it's correct because the `Counter` just increments existing counts. More complex implementations might only process the new part of the chunk, but this version is much clearer for learning.

### Common Confusion: "Why exactly `n-1` characters for the overlap?"

**You might think**: Maybe I should save `n` characters, or some other number, just to be safe.

**Actually**: `n-1` is the precise number required. An n-gram has a length of `n`. To span a boundary, it must have at least one character in the next chunk and at least one in the current chunk. The most it could leave behind in the current chunk is `n-1` characters. For a trigram (n=3), the overlap needed is 2. For example, the trigram `ABC` could be split as `A` | `BC` or `AB` | `C`. In the worst case, we need to remember `AB` (2 characters). Saving more (`n` or more) is wasteful, and saving less (`n-2` or less) would cause us to miss n-grams.

**How to remember**: To form a complete n-gram, you need `n` characters. If the next chunk provides the `n`'th character, you must have the preceding `n-1` characters from the previous chunk.

### Production Perspective

**Choosing a Chunk Size**: The `chunk_size` is a tunable parameter.

- **Too small**: Reading a file in tiny chunks (e.g., 64 bytes) can be inefficient. Each `read()` call is a system call, which has overhead. Too many calls can slow down your program due to I/O overhead.
- **Too large**: The chunk size determines the peak memory usage of your processing loop. If you set `chunk_size` to 1GB, you're back to a high-memory batch process.
- **A good compromise**: A size between 4KB (4096) and 4MB (4194304) is typical. These values balance the I/O call overhead with memory usage. 8KB to 128KB is a common starting point.

This chunking-with-overlap pattern is fundamental to almost all large-scale file processing, from command-line tools like `grep` to massive data systems like Apache Spark.

## Module Synthesis

## Module Synthesis: Assembling the Full Pipeline

In this module, we've built a complete, memory-bounded, streaming algorithm piece by piece. Let's review the journey and assemble the final pipeline.

### Our Building Blocks

1.  **The Memory Problem (Section 4.1)**: We established that batch processing, as used in Modules 1-3, fails for large datasets due to memory exhaustion. This motivated the need for a new approach.

2.  **Generators (Section 4.2 & 4.3)**: We discovered Python's `yield` keyword and the concept of lazy evaluation. Generators allow us to produce values one at a time, keeping memory constant. We built a flexible `generate_ngrams` function.

3.  **The Streaming Counter (Section 4.4)**: We consumed our generator to count n-grams, shifting the memory burden from storing the entire sequence to storing only the unique counts. This exposed the next bottleneck: data with high variety.

4.  **Bounded Memory (Section 4.5)**: We accepted a trade-off between accuracy and memory, creating a `BoundedCounter` that uses an eviction policy to maintain a fixed memory footprint, making it an _approximate_ algorithm suitable for finding "heavy hitters".

5.  **File Chunking (Section 4.6)**: We implemented the final piece: reading data from a file in manageable chunks with an overlap buffer to correctly process n-grams that span chunk boundaries.

### The Complete Implementation

Now, let's combine all these components into a single function that can find the most common n-grams in a file of any size with a predictable, small amount of memory.

```python
import os
from collections import Counter

# --- Component 1: The N-gram Generator ---
def generate_ngrams(text: str, n: int):
    if len(text) < n:
        return
    for i in range(len(text) - n + 1):
        yield text[i:i+n]

# --- Component 2: The Bounded Counter ---
class BoundedCounter:
    def __init__(self, max_size: int):
        self.counts = Counter()
        self.max_size = max_size

    def add(self, item):
        if item in self.counts:
            self.counts[item] += 1
            return
        if len(self.counts) < self.max_size:
            self.counts[item] = 1
            return
        min_item = min(self.counts, key=self.counts.get)
        if self.counts[min_item] == 1: # Only evict if new item will be > 1 eventually
             del self.counts[min_item]
             self.counts[item] = 1

    def get_most_common(self, k=10):
        return self.counts.most_common(k)

# --- Component 3: The File Streamer ---
def stream_file_chunks(filepath: str, n: int, chunk_size: int = 8192):
    overlap = ""
    with open(filepath, 'r', encoding='utf-8') as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            block_to_process = overlap + chunk
            yield block_to_process
            overlap = block_to_process[-(n-1):] if n > 1 else ""

# --- The Final Pipeline ---
def find_common_ngrams_streaming(filepath: str, n: int, max_unique_ngrams: int, chunk_size: int = 8192):
    """
    Finds the most common n-grams in a large file using a streaming approach.
    """
    bounded_counter = BoundedCounter(max_size=max_unique_ngrams)

    file_stream = stream_file_chunks(filepath, n, chunk_size)

    for block in file_stream:
        ngram_stream = generate_ngrams(block, n)
        for ngram in ngram_stream:
            bounded_counter.add(ngram)

    return bounded_counter

# --- Demonstration ---
# Create a dummy large file (e.g., 1MB)
dummy_filepath = "large_dummy_file.txt"
with open(dummy_filepath, "w") as f:
    # A mix of common and rare phrases
    common_phrase = "the_quick_brown_fox_"
    rare_phrase = "a_lazy_sleepy_dog_"
    for i in range(20000): # Makes a file of about 400KB
        f.write(common_phrase)
        if i % 10 == 0:
            f.write(rare_phrase)

# Run the pipeline
# We will search for 4-grams (e.g., 'the_')
# We give our counter space for only 100 unique n-grams
n_size = 4
top_k = 5
streaming_counter = find_common_ngrams_streaming(
    filepath=dummy_filepath,
    n=n_size,
    max_unique_ngrams=100
)

print(f"--- Top {top_k} most common {n_size}-grams ---")
for ngram, count in streaming_counter.get_most_common(top_k):
    print(f"'{ngram}': {count}")

# Clean up
os.remove(dummy_filepath)
```

**Output**:

```
--- Top 5 most common 4-grams ---
'the_': 20000
'he_q': 20000
'e_qu': 20000
'_qui': 20000
'quic': 20000
```

This final implementation successfully finds the most frequent n-grams from our test file. If you were to monitor your system's memory while this script runs on a 10GB file, you would see its memory usage remain incredibly low and constant—likely just a few megabytes for the chunk size and the bounded counter. We have successfully built an algorithm that scales not with the size of the data, but with the complexity of the question we are asking ("find the top 100 n-grams").

### Performance Characteristics and When to Use Streaming

- **Memory (Space)**: The key advantage. Memory usage is **O(K + C)**, where K is the `max_unique_ngrams` and C is the `chunk_size`. This is constant with respect to the total file size, which is a game-changer.
- **Time (CPU)**: The time complexity is still **O(L)**, where L is the length of the file, because we must inspect every character once. There is some overhead from function calls and string slicing, so for a file that _does_ fit into memory, a batch approach might be slightly faster. But when the file exceeds RAM, the batch method's performance becomes infinitely slow (due to disk swapping or crashing), making streaming the only viable option.

**Use the Streaming Approach When:**

1.  **Data Size > RAM**: This is the most common and compelling reason. If you cannot load your data, you must stream it.
2.  **Unbounded Data Sources**: The data source is not a finite file but a continuous stream, like a network feed, real-time logs, or a social media API.
3.  **Incremental Results are Needed**: You want to see partial results as the data is being processed, for example, to update a live dashboard.

If your data fits comfortably in memory and you don't need incremental results, the simpler batch processing algorithms from Modules 1 and 3 are often easier to write and debug.

This powerful streaming pattern concludes our exploration of distinct algorithmic approaches. In the next module, **Module 5: Comparative Analysis & Algorithm Selection**, we will put all four algorithms side-by-side and develop a framework for choosing the right tool for the right job based on data size, performance needs, and problem constraints.
