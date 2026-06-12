# 🪙 Hybrid Byte-Pair Encoding (BPE) Tokenizer Engine

A from-scratch Python implementation of both **Character-Level BPE** and advanced **Byte-Level BPE (BBPE)** tokenizers. This project maps out the fundamental text-to-token transformations powering modern Large Language Models (LLMs)—from early legacy setups to the robust, zero-unknown token systems used by architectures like GPT-4, LLaMA, and Qwen.

---

## 🚀 How the Training Algorithms Work

### 🎭 Variant A: Character-Based BPE

The character pipeline builds vocabulary by treating text as a collection of individual alphabetic characters:

1. **Pre-tokenization & Boundary Marking:** The raw training corpus is split into individual words using whitespace delimiters. To prevent cross-word character stitching during merging, a word boundary token (`</w>`) is appended to the tail of every word tuple.
   * *Example:* `"low lower"` $\rightarrow$ `('l', 'o', 'w', '</w>')`, `('l', 'o', 'w', 'e', 'r', '</w>')`
2. **Base Vocabulary Initialization:** A unique set of all individual characters present across the corpus is compiled to form the immutable base layer.
3. **Iterative Merging:** It slides a tracking window across word tuples, isolates the most frequent adjacent pair (e.g., `'l' + 'o'`), merges them into an atomic token (`'lo'`), and appends this rule chronologically to a `merge_rules` dictionary.

---

### ⚡ Variant B: Byte-Level BPE (BBPE) — *Modern LLM Standard*

The byte-level pipeline drops whitespace assumptions entirely and views text at the bare-metal hardware layer:

1. **Raw Byte Cast:** The entire string corpus is converted into its underlying raw UTF-8 byte stream. 
2. **Base Vocabulary Initialization (The 256 Rule):** Unlike character BPE (which depends on dataset characters), BBPE initializes its base vocabulary with all **256 possible single-byte values (0 to 255)**. 
3. **Whitespace Character Anchoring (`Ġ`):** To preserve spaces without physically breaking strings into lists, native spaces are translated into a visible byte token (commonly represented as **`Ġ`** in GPT models or **`_`** in SentencePiece). Crucially, this token is attached as a prefix to the **front** of the subsequent word.
   * *Example String:* `"apple banana"`
   * *Byte Array State:* `['apple', 'Ġbanana']`
4. **Iterative Merging:** The algorithm counts adjacent byte pairs across the continuous stream. The most common byte pairs are sequentially bound into multi-byte tokens (which gradually grow to represent full syllables, roots, and whole words).

---

## 🧩 The Testing & Inference Pipeline

When processing unseen test strings, both tokenizers run their learned states in reverse:

1. **Initialization:** Incoming text is broken down into its base components (either individual characters + `</w>` markers, or raw 0-255 byte arrays).
2. **Deterministic Execution:** The tokenizer iterates through the compiled `merge_rules` list **in the exact chronological order** they were discovered during training.
3. **Greedy Substitution:** Wherever an adjacent sequence matches a historical merge rule, those tokens are bound together. The process repeats until no further valid merge operations can be applied.

---

## 📊 Structural Trade-Off Analysis

| Feature / Metric | Character-Based BPE | Byte-Level BPE (BBPE) |
| :--- | :--- | :--- |
| **Base Vocab Size** | Variable (Depends on unique training characters) | Fixed at **256** base tokens |
| **Out-Of-Vocabulary (`<UNK>`)** | **High Risk.** Fails on new emojis, unique symbols, or foreign characters. | **Zero Risk.** Every character in existence maps to bytes; `<UNK>` is eliminated. |
| **Whitespace Handling** | Hard space splits. Fragile to tabs, double spaces, and formatting typos. | Space is an active character (`Ġ`). Robustly preserves formatting. |
| **Multilingual Support** | Poor. Highly inefficient for non-whitespace languages (Chinese, Japanese). | Native. Multi-byte languages are naturally compressed by pair merging. |
| **Anchor Positioning** | **Trailing Indicator (`</w>`)** at the end of token sequences. | **Leading Anchor (`Ġ`)** attached to the front of word tokens. |

---

## 🛠️ The Strategic Value of Front Anchoring (`Ġ`)

Moving from trailing word markers (`</w>`) to leading space anchors (`Ġ`) directly optimizes the performance of generative Autoregressive (Left-to-Right) LLMs:

* **Eliminating Empty Steps:** LLMs predict text token-by-token. With front-spaces, when a model finishes a word and decides on the next concept, it outputs the space and the word simultaneously (e.g., `Ġbanana`). If spaces were at the back (`bananaĠ`), the model would have to waste an entire prediction loop just to emit an empty space *before* selecting its next conceptual word.
* **No Ghost Spaces:** When an LLM concludes an execution sequence, a front-space system stops perfectly cleanly at the final character. A back-space system inherently risks leaving an invisible, trailing "ghost space" (` `) at the end of outputs, which corrupts code syntax and text form applications.
* **Punctuation Protection:** A word naturally knows if it needs a space *before* it begins based on the past history, but it cannot guess if a comma, period, or bracket will immediately follow it. Front anchoring protects the right edge of tokens from adding incorrect spaces before punctuation marks.
