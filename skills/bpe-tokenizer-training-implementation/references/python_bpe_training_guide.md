## Python BPE Training Guide: Step-by-Step Implementation

This guide provides a practical, step-by-step walkthrough for implementing the Byte Pair Encoding (BPE) training algorithm in Python. You will build the core functions necessary to train a custom tokenizer from scratch, using a text corpus.

### 1. Initial Setup: Preparing Your Text Data

The BPE algorithm begins by treating the input text as a sequence of individual bytes. For Python, this means encoding your string into UTF-8 bytes and then converting those bytes into a list of integers (where each integer represents a byte value from 0 to 255).

```python
# Step 1: Load your text data
# For demonstration, we'll use a simple string. In a real scenario, this would be a large corpus.
text_data = "This is a sample text for BPE. BPE is cool!"

# Encode the text into UTF-8 bytes
byte_sequence = text_data.encode("utf-8")

# Convert the bytes object into a list of integers (our initial tokens)
initial_tokens = list(byte_sequence)

print(f"Original text: '{text_data}'")
print(f"Initial tokens (byte integers): {initial_tokens}")
print(f"Length of initial tokens: {len(initial_tokens)}")
# Example output:
# Original text: 'This is a sample text for BPE. BPE is cool!'
# Initial tokens (byte integers): [84, 104, 105, 115, 32, 105, 115, 32, 97, 32, 115, 97, 109, 112, 108, 101, 32, 116, 101, 120, 116, 32, 102, 111, 114, 32, 66, 80, 69, 46, 32, 66, 80, 69, 32, 105, 115, 32, 99, 111, 111, 108, 33]
# Length of initial tokens: 43
```

### 2. `get_stats` Function: Counting Pair Frequencies

This function is responsible for scanning the current list of tokens and counting how many times each consecutive pair of tokens appears. This is a crucial step in identifying the most frequent pair for merging.

```python
def get_stats(ids):
    """
    Counts the frequency of all consecutive pairs in a list of token IDs.
    Args:
        ids (list): A list of integer token IDs.
    Returns:
        dict: A dictionary mapping (token1, token2) tuples to their frequencies.
    """
    counts = {}
    for i in range(len(ids) - 1):
        pair = (ids[i], ids[i+1])
        counts[pair] = counts.get(pair, 0) + 1
    return counts

# Example usage:
stats = get_stats(initial_tokens)
print("\nPair statistics (top 5):")
# Sort for better visualization
sorted_stats = sorted(stats.items(), key=lambda item: item[1], reverse=True)
for pair, count in sorted_stats[:5]:
    print(f"  {pair}: {count}")

# Find the most frequent pair
if stats:
    most_frequent_pair = max(stats, key=stats.get)
    print(f"Most frequent pair: {most_frequent_pair} (count: {stats[most_frequent_pair]})\n")
else:
    most_frequent_pair = None
    print("No pairs found.")
```

### 3. `merge_pair` Function: Replacing Pairs with New Tokens

Once the most frequent pair is identified, this function takes the current list of tokens and replaces every occurrence of that specific pair with a new, single token ID. This is the core 'compression' step of BPE.

```python
def merge_pair(ids, pair, new_idx):
    """
    Replaces all occurrences of a specific pair of token IDs with a new token ID.
    Args:
        ids (list): The current list of integer token IDs.
        pair (tuple): The (token1, token2) pair to be merged.
        new_idx (int): The new token ID to replace the pair with.
    Returns:
        list: A new list of token IDs with the merges applied.
    """
    new_ids = []
    i = 0
    while i < len(ids):
        # Check if the current position and the next form the target pair
        if i + 1 < len(ids) and (ids[i], ids[i+1]) == pair:
            new_ids.append(new_idx)
            i += 2  # Skip both elements of the merged pair
        else:
            new_ids.append(ids[i])
            i += 1
    return new_ids

# Example usage (assuming most_frequent_pair was found):
if most_frequent_pair:
    # Assign a new ID for the merged token (starting from 256, as 0-255 are byte values)
    new_token_id = 256
    merged_tokens = merge_pair(initial_tokens, most_frequent_pair, new_token_id)
    print(f"Tokens after merging {most_frequent_pair} into {new_token_id}: {merged_tokens}")
    print(f"Length after merge: {len(merged_tokens)}")
```

### 4. The BPE Training Loop: Iterative Merging

This is where `get_stats` and `merge_pair` are combined into an iterative process. You define a target vocabulary size (or number of merges), and the loop repeatedly finds the most frequent pair, merges it, and updates the token sequence.

```python
# Step 4: BPE Training Loop

current_tokens = list(initial_tokens) # Start with a copy of initial bytes

# Define the target vocabulary size. For example, 256 initial bytes + 20 merges = 276 tokens.
TARGET_VOCAB_SIZE = 276
num_merges = TARGET_VOCAB_SIZE - 256

merges = {} # Stores (pair) -> new_id mappings
# Initialize vocabulary with individual bytes (id -> bytes representation)
vocab = {idx: bytes([idx]) for idx in range(256)}

print(f"\nStarting BPE training for {num_merges} merges...")

for i in range(num_merges):
    stats = get_stats(current_tokens)
    if not stats: # No more pairs to merge (e.g., all unique tokens)
        print("No more pairs to merge. Stopping early.")
        break

    # Find the most frequent pair
    most_frequent_pair = max(stats, key=stats.get)
    
    # Assign a new token ID, starting from 256
    new_token_id = 256 + i
    
    # Record the merge
    merges[most_frequent_pair] = new_token_id
    
    # Update the vocabulary for decoding later
    # The new token's bytes are the concatenation of the merged pair's bytes
    vocab[new_token_id] = vocab[most_frequent_pair[0]] + vocab[most_frequent_pair[1]]
    
    # Apply the merge to the current token sequence
    current_tokens = merge_pair(current_tokens, most_frequent_pair, new_token_id)
    
    print(f"Merge {i+1}/{num_merges}: {most_frequent_pair} -> {new_token_id} (count: {stats[most_frequent_pair]}). New sequence length: {len(current_tokens)}")

print("\nBPE training complete.")
print(f"Final token sequence: {current_tokens}")
print(f"Final vocabulary size: {len(vocab)}")
print(f"Total merges performed: {len(merges)}")

# The 'merges' dictionary and 'vocab' dictionary are now your trained tokenizer components.
# You can save these to disk for later use in encoding and decoding.
```

### Conclusion

You have now successfully implemented the core BPE training algorithm. The `merges` dictionary contains the learned subword units and their corresponding new IDs, while the `vocab` dictionary maps all token IDs (from individual bytes to merged subwords) back to their byte representations. These two components are all you need to perform encoding (text to tokens) and decoding (tokens to text) with your custom BPE tokenizer.