# Character-Based Byte-Pair Encoding (BPE) Tokenizer

A from-scratch Python implementation of a character-level Byte-Pair Encoding (BPE) tokenizer. This project demonstrates the foundational sub-word tokenization mechanics utilized in modern Large Language Models (LLMs) to construct an optimal vocabulary from a text corpus.

---

## 🚀 How the Training Algorithm Works

The tokenizer pipeline automates vocabulary construction via an iterative frequency-extraction loop over the training data:

### 1. Pre-tokenization & Boundary Marking
The raw training corpus is split into individual words using whitespace delimiters. To maintain structural integrity and prevent cross-word character stitching, an explicit word boundary token (`</w>`) is appended to the tail of every word tuple.
* **Input String:** `"low lower"`
* **Processed State:** `('l', 'o', 'w', '</w>')`, `('l', 'o', 'w', 'e', 'r', '</w>')`

### 2. Base Vocabulary Initialization
A unique set of all individual characters present across the corpus is compiled. This forms the immutable base layer of the token vocabulary.

### 3. Iterative Pair Extraction & Merging
The algorithm enters a loop for a user-defined number of iterations (`num_merges`):
1. **Adjacent Pair Counting:** It slides a tracking window across all word tuples to count the raw frequencies of adjacent tokens, weighted by the occurrence frequency of the parent word.
2. **Maximum Selection:** The character pair with the highest overall frequency count is isolated.
3. **Global Merge:** The isolated pair is combined into a new, atomic multi-character token (e.g., `'l' + 'o' -> 'lo'`).
4. **State Update:** The new token is permanently added to the token vocabulary set, and the exact transformation rule is pushed to a sequential `merge_rules` dictionary.

---

## 🧩 The Testing Pipeline (Inference)

When encountering unseen test data, the tokenizer operates in reverse using the states saved during training:
1. The test text is split into base characters and appended with `</w>`.
2. The tokenizer reads the compiled `merge_rules` dictionary **in the exact chronological order** they were discovered during training.
3. If an input matches a known merge rule, those characters are bound together. This loop continues until no further valid merge rules can be applied to the sequence.

---

## 📊 Trade-Off Analysis

While character-based BPE provides a massive efficiency leap over traditional word-level structures, it features inherent operational limitations:

### 🌟 Advantages
* **Sub-word Efficiency:** Balances vocabulary size against sequence tracking lengths by keeping frequent full words whole while representing rare words by their structural components.
* **Explicit Word Anchoring:** The `</w>` delimiter ensures that prefixes, suffixes, and core roots are analyzed within their true morphological contexts.

### ⚠️ Disadvantages
* **The Unknown Token Fallback (`<UNK>`):** Because the base vocabulary is anchored to the characters present in the *training* set, running the tokenizer on test data containing unique symbols, foreign characters, or unseen emojis will fail, forcing a fallback to `<UNK>` symbols.
* **Rigid Whitespace Dependency:** The initial step relies heavily on space splitting. This makes it poorly optimized for languages without natural whitespace boundaries (e.g., Mandarin, Japanese) and susceptible to formatting anomalies.

> *Note: Advanced variants like Byte-Level BPE (BBPE) or SentencePiece resolve these core formatting issues by utilizing native byte-level vocabularies and treating spaces as regular structural characters.*
