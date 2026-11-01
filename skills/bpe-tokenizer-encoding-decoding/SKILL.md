---
description: This skill enables the AI agent to perform both encoding (text to tokens)
  and decoding (tokens to text) using a custom-trained Byte Pair Encoding (BPE) tokenizer.
  It covers the practical application of the `merges` dictionary and vocabulary, including
  robust handling of UTF-8 decoding errors.
name: bpe-tokenizer-encoding-decoding
tags:
- tokenizer
- encoding
- decoding
- BPE
- Python
- UTF-8 errors
- merges dictionary
- vocabulary lookup
- text processing
---

# bpe-tokenizer-encoding-decoding

> This skill enables the AI agent to perform both encoding (text to tokens) and decoding (tokens to text) using a custom-trained Byte Pair Encoding (BPE) tokenizer. It covers the practical application of the `merges` dictionary and vocabulary, including robust handling of UTF-8 decoding errors.

## Encoding and Decoding with a Custom BPE Tokenizer

Once a Byte Pair Encoding (BPE) tokenizer has been trained, resulting in a `merges` dictionary and a `vocab` mapping, you can use it to convert raw text into sequences of tokens (encoding) and convert token sequences back into human-readable text (decoding). These are the two fundamental operations for any tokenizer interacting with an LLM.

### Overview

Your goal is to implement robust encoding and decoding functions. Decoding is generally simpler as it involves reversing the merges. Encoding is more complex as it requires iteratively applying the merges in the correct order to the input text.

### Step-by-Step Actions

1.  **Prepare Trained Components:**
    *   **Action:** Ensure you have access to the `merges` dictionary (mapping `(child1, child2) -> parent_id`) and the `vocab` dictionary (mapping `id -> bytes`) generated during BPE training. These are essential for both encoding and decoding.
    *   **Example (from training skill):**
        ```python
        # Example 'merges' and 'vocab' from a hypothetical training run
        merges = {
            (101, 32): 256, # 'e ' -> 256
            (105, 115): 257, # 'is' -> 257
            (256, 257): 258, # 'e is' -> 258 (example of nested merge)
            # ... more merges
        }
        vocab = {
            **{idx: bytes([idx]) for idx in range(256)}, # Initial bytes
            256: b'e ',
            257: b'is',
            258: b'e is',
            # ... more vocab entries
        }
        ```

2.  **Implement Decoding Function (`decode`):**
    *   **Action:** Create a function `decode(ids, vocab)` that takes a list of token IDs and the `vocab` dictionary. It should reconstruct the original byte sequence by looking up each ID in `vocab` and concatenating the resulting bytes. Finally, it decodes the full byte sequence into a Python string using UTF-8, handling potential errors.
    *   **Best Practice:** When concatenating bytes, ensure you are working with `bytes` objects. The `b''.join()` method is efficient for this. Crucially, use `errors='replace'` in `bytes.decode()` to gracefully handle cases where the LLM might generate an invalid UTF-8 byte sequence.
    *   **Reference:** Consult `references/bpe_decoding_workflow.md` for a detailed guide and `references/utf8_error_handling.md` for error management.
    *   **Code Example (Python):**
        ```python
        def decode(ids, vocab):
            """
            Decodes a list of token IDs back into a Python string.
            Args:
                ids (list): A list of integer token IDs.
                vocab (dict): The vocabulary mapping token IDs to their byte representations.
            Returns:
                str: The decoded text string.
            """ 
            # Look up bytes for each ID and concatenate them
            raw_bytes = b"".join(vocab[idx] for idx in ids)
            # Decode bytes to string, handling potential UTF-8 errors
            return raw_bytes.decode("utf-8", errors="replace")

        # Example usage:
        # decoded_text = decode([84, 104, 257, 256, 258], vocab) # Example token sequence
        # print(f"Decoded text: '{decoded_text}'")
        ```

3.  **Implement Encoding Function (`encode`):**
    *   **Action:** Create a function `encode(text, merges)` that takes a raw text string and the `merges` dictionary. It should first convert the text into initial UTF-8 byte tokens. Then, it iteratively applies the merges in the *same order they were learned* during training until no more merges are possible for the current sequence.
    *   **Pitfall:** The order of merges matters. You must apply merges from the earliest created (lowest `new_id`) to the latest. This means iterating through the `merges` dictionary in the order of its keys (which Python 3.7+ guarantees is insertion order).
    *   **Best Practice:** Reuse the `get_stats` and `merge_pair` functions from the training phase. The encoding loop will repeatedly find the most eligible pair (the one with the lowest `new_id` in `merges`) and apply that merge.
    *   **Reference:** Refer to `references/bpe_encoding_workflow.md` for a comprehensive guide.
    *   **Code Example (Python):**
        ```python
        # Assume get_stats and merge_pair functions are available from training
        # Assume 'merges' dictionary is available

        def get_stats(ids):
            counts = {}
            for i in range(len(ids) - 1):
                pair = (ids[i], ids[i+1])
                counts[pair] = counts.get(pair, 0) + 1
            return counts

        def merge_pair(ids, pair, new_idx):
            new_ids = []
            i = 0
            while i < len(ids):
                if i + 1 < len(ids) and (ids[i], ids[i+1]) == pair:
                    new_ids.append(new_idx)
                    i += 2
                else:
                    new_ids.append(ids[i])
                    i += 1
            return new_ids

        def encode(text, merges):
            """
            Encodes a text string into a list of token IDs using the trained merges.
            Args:
                text (str): The input text string.
                merges (dict): The dictionary mapping (child1, child2) pairs to their new token ID.
            Returns:
                list: A list of integer token IDs.
            """
            # 1. Initial tokenization: convert text to list of byte integers
            tokens = list(text.encode("utf-8"))

            # 2. Iteratively apply merges
            # Create a reverse lookup for merges to find the new_id for a pair
            # This assumes merges are ordered by new_id, which they are if built sequentially
            # For encoding, we need to know which pair maps to which new_id
            # The 'merges' dict already provides this: (pair) -> new_id

            while True:
                # Get current pair statistics
                stats = get_stats(tokens)
                if not stats: # No pairs left to merge
                    break

                # Find the most eligible pair to merge: the one that was merged earliest (lowest new_id)
                # We iterate through all possible pairs in 'tokens' and check if they are in 'merges'
                # We want the pair from 'tokens' that has the *smallest* new_id in 'merges'
                best_pair = None
                min_merge_id = float('inf')

                for pair_in_tokens in stats.keys():
                    if pair_in_tokens in merges: # Is this pair a known merge?
                        merge_id = merges[pair_in_tokens]
                        if merge_id < min_merge_id:
                            min_merge_id = merge_id
                            best_pair = pair_in_tokens
                
                if best_pair is None: # No more mergeable pairs found in current tokens
                    break

                # Apply the merge
                tokens = merge_pair(tokens, best_pair, merges[best_pair])
            
            return tokens

        # Example usage:
        # text_to_encode = "This is a sample text for BPE."
        # encoded_ids = encode(text_to_encode, merges)
        # print(f"Encoded IDs: {encoded_ids}")
        ```

### Validation

*   **Round-trip Test:** Encode a known string, then decode the resulting token IDs. The decoded string should be identical to the original input string (barring any `errors='replace'` substitutions during decoding).
    ```python
    original_text = "Hello, world! This is a test with some é and 😀."
    encoded_ids = encode(original_text, merges)
    decoded_text = decode(encoded_ids, vocab)
    print(f"Original: '{original_text}'")
    print(f"Decoded:  '{decoded_text}'")
    assert original_text == decoded_text # Should pass if no invalid UTF-8 was generated
    ```
*   **Token Count Comparison:** Compare the token count of your custom BPE tokenizer with a known tokenizer (e.g., `tiktoken` for GPT-2/GPT-4) for the same text. While not identical, they should show similar compression trends (e.g., GPT-4 tokenizer uses fewer tokens than GPT-2 for Python code).

### Best Practices and Pitfalls

*   **Consistency:** Always use the exact `merges` dictionary and `vocab` generated during training for both encoding and decoding. Inconsistencies will lead to incorrect results.
*   **Order of Merges (Encoding):** The encoding process *must* apply merges in the same order they were learned (i.e., by the `new_id` assigned during training). This ensures that larger, more complex subwords are formed correctly from their constituent parts.
*   **UTF-8 Error Handling:** The `errors='replace'` parameter in `bytes.decode()` is a critical safety net. Without it, an LLM generating an invalid byte sequence could crash your decoding process. The replacement character signals an issue in the LLM's output.
*   **Efficiency:** For production-grade tokenizers, the `encode` function can be optimized beyond the simple `while True` loop with `get_stats` in each iteration. More advanced implementations might use data structures like tries or suffix arrays for faster lookup and merging of pairs.

By mastering this skill, you can effectively translate between raw text and token sequences, enabling your AI agent to prepare data for LLMs and interpret their outputs reliably.

## 📺 Source Videos

- [Let's Build the GPT Tokenizer](https://www.youtube.com/watch?v=zduSFxRajkE) — Andrej Karpathy

**Difficulty**: Intermediate

## Prerequisites

- llm-tokenization-fundamentals
- bpe-tokenizer-training-implementation
- Python programming proficiency
