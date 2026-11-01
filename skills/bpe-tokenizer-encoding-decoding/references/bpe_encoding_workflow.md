## BPE Encoding Workflow: From Text to Tokens

Encoding is the process of converting a raw text string into a sequence of numerical token IDs using a trained Byte Pair Encoding (BPE) tokenizer. This is the inverse of decoding and is typically more complex because it involves iteratively applying the learned merges to the input text.

### The Role of the Merges Dictionary (`merges`)

The `merges` dictionary, generated during BPE training, is the central component for encoding. It stores the history of all merge operations, mapping a `(child1_id, child2_id)` pair to its `new_parent_id`.

**Example `merges` structure:**
```python
merges = {
    (101, 32): 256, # 'e ' -> 256
    (105, 115): 257, # 'is' -> 257
    (256, 257): 258, # 'e is' -> 258 (example of nested merge)
    # ... and so on, in the order they were learned
}
```

Crucially, the order in which these merges were learned (and thus, the order in which they appear in the `merges` dictionary if using Python 3.7+ which preserves insertion order) is vital for encoding. Merges with lower `new_parent_id` values were learned earlier and must be applied before merges with higher `new_parent_id` values.

### Step-by-Step Encoding Process

Given a raw text string (e.g., "This is a sample text."), the encoding process unfolds as follows:

1.  **Initial Tokenization (Bytes):** Convert the input text into its raw UTF-8 byte sequence, represented as a list of integers (0-255). This forms the starting sequence of tokens.
    *   `