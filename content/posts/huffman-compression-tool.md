+++
title = "Writing a Huffman Compressor: Prefix Codes, Bit Packing, and the Header Problem"
date = "2026-05-12"
draft = false
tags = ["python", "compression", "algorithms", "systems-programming", "binary-formats"]
categories = ["engineering"]
description = "A from-scratch Huffman coder in Python. Building the prefix tree, packing variable-length codes into bytes, and the small file-format decisions that decide whether your decoder works at all."
[cover]
  hidden = true
+++

Compression is one of those topics that sits in a peculiar spot: everyone uses it constantly, almost nobody writes one. The algorithms behind `gzip`, `zip`, `zstd` get treated as a black box you pipe bytes through. Huffman coding is the smallest interesting member of that family, and writing one by hand is the cleanest way to surface the parts of compression that aren't really about compression at all: how you pack variable-length codes into fixed-size bytes, how the decoder finds the table, and what "the end of the file" even means when your last byte is half real data and half padding.

I wrote a small one in Python. Two scripts, an encoder and a decoder, with no external dependencies. The compression ratio isn't competitive with anything modern, but every line is visible.

---

## The idea, briefly

Fixed-width encodings (ASCII, UTF-8 for the BMP) spend the same number of bits on every character. That's wasteful when the input is skewed: in English text `e` shows up far more often than `q`, but both pay 8 bits. Huffman coding builds a *prefix code* where frequent characters get short codes and rare ones get long codes, with the constraint that no code is a prefix of another (so the decoder never has to guess where one symbol ends and the next begins).

The construction is the textbook greedy algorithm:

1. Count character frequencies.
2. Make a leaf node for each character, weighted by its frequency.
3. Repeatedly take the two lowest-weight nodes, fuse them under a new internal node whose weight is their sum, and put it back in the pool.
4. When one node is left, it's the root of a binary tree where every leaf is a character. Walk the tree, recording `0` for left and `1` for right; the path to each leaf is its code.

The prefix property falls out of the structure for free: characters only live at leaves, so no character's code can be a prefix of another's.

## The tree

```python
class Node:
    def __init__(self, weight_, letter_=None, left_child_=None, right_child_=None):
        self.weight = weight_
        self.letter = letter_
        self.left_child = left_child_
        self.right_child = right_child_
```

One class for both leaves and internal nodes. Leaves carry a `letter` and no children; internal nodes carry children and no `letter`. That's a deliberate choice. You could split it into `Leaf` and `Internal` subclasses, but for an algorithm that constantly fuses leaves and internal nodes interchangeably the union type is less ceremony.

The build loop is the part that wants a priority queue. I used a sorted list and inserted in place:

```python
while len(ordered_nodes_list) > 1:
    node_zero = ordered_nodes_list.pop(0)
    node_one = ordered_nodes_list.pop(0)
    new_node = Node(weight_=node_zero.weight + node_one.weight,
                    left_child_=node_zero, right_child_=node_one)
    i = 0
    while i < len(ordered_nodes_list) and new_node.weight > ordered_nodes_list[i].weight:
        i += 1
    ordered_nodes_list.insert(i, new_node)
```

This is `O(n²)` over the alphabet: each insert scans, each `pop(0)` shifts. For ASCII text where the alphabet is at most a few hundred symbols it doesn't matter. The "right" structure is `heapq`: pushing the merged node is `O(log n)` instead of linear. Worth knowing the asymptotic cost is there if you ever feed this something with a million-symbol alphabet (Unicode-heavy input, or symbol-level Huffman over words instead of characters).

Walking the tree to build the code table is a textbook DFS:

```python
def visit_node(node, code, table):
    if not node.left_child and not node.right_child:
        table[node.letter] = code
        return
    if node.left_child:  visit_node(node.left_child, code + '0', table)
    if node.right_child: visit_node(node.right_child, code + '1', table)
```

The codes come out as Python strings of `'0'` and `'1'`. That's not how they'll live on disk, but it makes the rest of the encoder trivial to write and inspect. Premature bit-fiddling here would obscure the algorithm.

## Bit packing: the part nobody mentions

Now the actually interesting bit. Huffman codes are variable-length: maybe `e` is `010` and `q` is `1110110`. Disks and memory deal in bytes. So the encoder has to pack a stream of variable-length codes into a stream of 8-bit bytes, and the decoder has to do the inverse.

The shape of what we're about to do: accumulate code bits into a buffer, and every time the buffer has at least 8 bits, peel off the leftmost 8 and write them as one byte. Whatever's left over stays in the buffer for the next character. At the very end, the buffer probably has fewer than 8 bits; we'll handle that as a special case.

Here's that as code:

```python
buffer = ''
for char in text:
    buffer += table[char]
    if len(buffer) >= 8:
        data.append(int(buffer[:8], 2))
        buffer = buffer[8:]
```

Using a Python string as a bit buffer is wasteful (`O(n)` slicing) but correct, and for a teaching implementation I'd rather see the bits than wrestle with bit-level integer arithmetic.

The problem comes at the end. The total bit length of the encoded text is almost never a multiple of 8. So the last byte gets padded:

```python
if len(buffer) > 0:
    padded = buffer.zfill(8)
    data.append(int(padded, 2))
    padding = 8 - len(buffer)
    table["padding_size"] = padding
```

`zfill` pads on the left with `0`s. That choice matters: the decoder reads the byte's bits left-to-right and tries to match prefixes, so the *real* bits are the right-hand suffix and the padding is the left-hand prefix. The `padding_size` records how many leading bits of that final byte are garbage.

This is the line where the file format quietly stops being self-describing. Without that field, the decoder cannot tell whether the trailing `0` bits in the last byte are part of a legitimate code or filler. There's no in-band marker for "end of stream" in raw Huffman output. Some implementations sidestep this by adding an explicit EOF symbol with its own code; that's cleaner but uses a few extra bits per file. Recording the padding count is the cheap alternative, and it's the choice every tutorial Huffman compressor I've seen makes implicitly without ever flagging it as a *choice*.

## The header problem

The decoder needs the code table. If you don't ship it alongside the data, the compressed file is just noise... the codes are generated freshly per input, so there's no shared table to fall back on. (You *can* use a fixed canonical table tuned for, say, English text. That's how some real compressors trade off table size against generality. For a single-file tool, per-file tables are simpler and tighter.)

So the file layout is: a header containing the table, followed by the packed bit data. The header has to be self-delimiting: the decoder needs to know exactly where it ends and the data begins. The simplest way: prefix the header with its length.

```python
serialized_header = json.dumps(table)
header_bytes = serialized_header.encode("utf-8")
header_size = len(header_bytes).to_bytes(8, sys.byteorder)
header = bytearray(header_bytes)
header[:0] = header_size

with open(f"compressed_{output_filename}", "wb") as f:
    f.write(header + data)
```

Eight bytes of length, then that many bytes of UTF-8 JSON, then the packed data. The decoder reverses it:

```python
header_bytes_size = infile.read(8)
header_size = int.from_bytes(header_bytes_size, sys.byteorder)
header = infile.read(header_size).decode("utf-8")
table = json.loads(header)
```

Two things to flag here.

**JSON for the header is overkill but useful.** It costs space: every code is stored as an ASCII string of `'0'`/`'1'`, every character key is a JSON-quoted string, there's structural overhead per entry, and it inflates the smallest files past their original size. A tighter format would write each (character, code-length, code) triple in a packed binary form. But JSON is human-inspectable, which during development is worth more than the bytes. You can `head -c 2000 compressed_x.txt | tail -c 1000` and read your own header. That diagnostic loop closes fast.

**`sys.byteorder` is a footgun.** I used the host's native endianness for the length field, which means a file compressed on a little-endian machine cannot be decompressed on a big-endian one. For a real format you pick an endianness in the spec (network byte order (big-endian) is the conventional choice) and stick with it forever. Most binary file formats that survive contact with reality have made this mistake at least once. Pick early.

## Decoding

Decoding is the reverse walk. Read header, invert the table so codes map to characters, then pull bits from the data section and accumulate them until the running prefix matches an entry. The loop below does three things at once, so it's worth pulling them apart before reading the code.

**Thing one: read one byte at a time, unpack it into a string of 8 bit-characters.** That's `format(chunk[0], "08b")`: turn the integer into its binary representation, zero-padded to width 8.

**Thing two: detect whether the byte we just read is the *last* byte of the file.** We need to know this because the last byte has padding bits glued to its front that aren't real data. The trick: tentatively read the *next* byte. If that read returns empty, the previous byte was the last one. If it returned something, rewind one byte (`seek(-1, 1)`: seek by -1 relative to the current position) so the next loop iteration sees it. When we're on the last byte, we slice off `padding_size` leading bits before walking it. That's the mirror image of `zfill`: padding went on the left at encode time, comes off the left now.

**Thing three: for each bit, append it to a running prefix and check if that prefix is a code.** If it is, emit the character and reset the prefix. If it isn't, keep accumulating.

Now the code:

```python
inverted_table = {v: k for k, v in table.items()}
bit_in_progress = ""
while chunk := infile.read(1):
    bits = format(chunk[0], "08b")
    nxt = infile.read(1)
    if nxt == b"":
        bits = bits[padding_size:]
    else:
        infile.seek(-1, 1)
    for bit in bits:
        bit_in_progress += bit
        if inverted_table.get(bit_in_progress):
            outfile.write(inverted_table[bit_in_progress])
            bit_in_progress = ""
```

One correctness trap in this code worth naming: walking the tree vs. matching strings. The decoder above matches the running bit string against a dict of all codes. The textbook decoder instead walks the original tree node-by-node, taking left or right per bit and emitting a character when it hits a leaf. The dict version does more work per bit (string concatenation, hash lookup) but avoids keeping the tree around. For code clarity the dict version wins; for performance, the tree walk is dramatically better, since each bit becomes a single pointer-chase instead of an `O(k)` string append plus an `O(k)` hash.

## A side door into language models

There's a connection here that's easy to miss until you've written the encoder yourself. Huffman assigns each symbol a code of length roughly `-log2(p)` bits, where `p` is its frequency. That formula isn't a Huffman fact, it's Shannon's: the information content of an event with probability `p` is `-log2(p)` bits, and any encoder that beats that on average is impossible. Huffman's contribution is being a constructive, integer-length approximation to that bound.

A worked example makes this less abstract. Say the letter `e` appears with frequency 0.125 in your text. Then `-log2(0.125) = 3`, so a perfect encoder spends 3 bits per `e`. A rarer letter at frequency 0.001 costs `-log2(0.001) ≈ 9.97` bits. Huffman has to round those to whole numbers (it can't spend 9.97 bits), which is exactly where it loses ground to better encoders. The total compressed size is just the sum: for each symbol, multiply its count by its code length, add them up.

A language model is, mechanically, a thing that outputs a probability distribution over the next token given the prefix so far. Plug those probabilities into an arithmetic coder (Huffman's continuous-length cousin, which can spend fractional bits per symbol) and you get a compressor whose code length per token is `-log2(p_model(token | context))`. The better the model's predictions, the shorter the encoded file. This isn't a metaphor: DeepMind's "Language Modeling Is Compression" makes the equivalence literal and shows large LMs beating `gzip` and PNG on text and even on image bytes, using the model purely as a probability source for arithmetic coding.

Once you've watched the bit packer in this post turn frequencies into codes, the LM-as-compressor framing stops being mystical. Frequency tables are a zeroth-order model: `p(char)`. Modern compressors (`zstd`, the PPM family) use higher-order context models: `p(char | last few chars)`. An LM is the same idea pushed to its logical end: `p(token | entire prefix)`, parameterized by a neural net. The encoder/decoder structure is unchanged; only the probability source got smarter. Conversely, every bit of compression ratio a model achieves over a uniform baseline is, in a precise sense, a measurement of how much structure the model has actually learned.

## What this project actually teaches

The compression algorithm itself is a single page of code. The interesting hours go into the boundary work: choosing a header format, deciding how to mark padding, picking an endianness, threading EOF through the decoder. Those decisions are what separate "I implemented Huffman" from "I produced a file format another program can read." They generalize: every binary protocol you'll ever write (a wire format, a save file, a cache layer) demands the same set of decisions, in the same order.

Two more things worth taking away.

**Variable-length encodings need explicit termination.** Either an end-of-stream symbol baked into the alphabet, or out-of-band metadata (a length, a padding count). There's no third option, and you have to pick before you write the encoder, because the decoder's structure follows from it. I picked padding count; an EOF symbol would have meant a slightly larger tree and a simpler decoder loop. Both are legitimate.

**Self-describing formats are paid for in overhead, and the overhead is worth it on small projects.** Shipping the table inside every file is wasteful when files are small and the alphabet is large. Real compressors amortize this with tricks: canonical Huffman codes (which let the table be reconstructed from just the lengths), shared dictionaries, or block-level retables. Skipping all of that and embedding a literal JSON dict makes the file inspectable, the bug surface tiny, and the decoder a few dozen lines. The price is that compressing a 100-byte file produces a larger file than the original. For a learning tool that's the right tradeoff; for `zstd` it obviously isn't.

The reason to write one of these is the same reason to write a JSON parser, or `wc`, or any other "already solved" problem from scratch: you find out which parts of the stack you'd been treating as one indivisible thing actually decompose into separable decisions, each with its own failure modes. Huffman is small enough that you can hold the entire pipeline (frequency table, tree, code book, bit packer, header, decoder) in your head at once. Most of the production formats you'll ever read are recognizably the same shape, scaled up.
