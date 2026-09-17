Title: Introducing Triton: Open-source GPU programming for neural networks

URL Source: https://openai.com/index/triton

Markdown Content:
# Introducing Triton: Open-source GPU programming for neural networks

![Introducing Triton Open Source Gpu Programming For Neural Networks](https://images.ctfassets.net/kftzwdyauwt9/cdce1ebd-19a2-4848-a08ec8c44e18/55b924fc6628318148b7c5c4902551e7/image-18.webp?w=3840&q=90&fm=webp)

We’re releasing Triton 1.0, an open-source Python-like programming language which enables researchers with no CUDA experience to write highly efficient GPU code—most of the time on par with what an expert would be able to produce.

## Why it matters

Triton makes it possible to reach peak hardware performance with relatively little effort; for example, it can be used to write FP16 matrix multiplication kernels that match the performance of cuBLAS—something that many GPU programmers can’t do—in under 25 lines of code. Our researchers have already used it to produce kernels that are up to 2x more efficient than equivalent Torch implementations, and we’re excited to work with the community to make GPU programming more accessible to everyone.

Novel research ideas in the field of Deep Learning are generally implemented using a combination of native framework operators. While convenient, this approach often requires the creation (and/or movement) of many temporary tensors, which can hurt the performance of neural networks at scale. These issues can be mitigated by writing specialized GPU kernels, but doing so can be surprisingly difficult due to the many intricacies of GPU programming.<sup>, ,</sup>  And, although a variety of systems have recently emerged<sup>,</sup>  to make this process easier, we have found them to be either too verbose, lack flexibility or generate code noticeably slower than our hand-tuned baselines. This has led us to extend and improve Triton<sup>, a recent language and compiler whose original creator now works at OpenAI.</sup>

The architecture of modern GPUs can be roughly divided into three major components—DRAM, SRAM and ALUs—each of which must be considered when optimizing CUDA code:

- Memory transfers from DRAM must be *coalesced* into large transactions to leverage the large bus width of modern memory interfaces.
- Data must be manually stashed to SRAM prior to being re-used, and managed so as to minimize shared memory bank conflicts upon retrieval.
- Computations must be partitioned and scheduled carefully, both across and within Streaming Multiprocessors (SMs), so as to promote instruction/thread-level parallelism and leverage special-purpose ALUs (e.g., tensor cores).

Basic architecture of a GPU.

Reasoning about all these factors can be challenging, even for seasoned CUDA programmers with many years of experience. The purpose of Triton is to fully automate these optimizations, so that developers can better focus on the high-level logic of their parallel code. Triton aims to be broadly applicable, and therefore does not automatically schedule work across SMs -- leaving some important algorithmic considerations (e.g. tiling, inter-SM synchronization) to the discretion of developers.

|  | **CUDA** | **TRITON** | 
| Memory Coalescing | Manual | Automatic | 
| Shared Memory Management | Manual | Automatic | 
| Scheduling (Within SMs) | Manual | Automatic | 
| Scheduling (Across SMs) | Manual | Manual | 

Compiler optimizations in CUDA vs Triton.

Out of all the Domain Specific Languages and JIT-compilers available, Triton is perhaps most similar to Numba: kernels are defined as decorated Python functions, and launched concurrently with different `program_id`’s on a grid of so-called *instances*. However, as shown in the code snippet below, the resemblance stops there: Triton exposes intra-instance parallelism via operations on *blocks*—small arrays whose dimensions are powers of two—rather than a Single Instruction, Multiple Thread (SIMT) <sup>execution model. In doing so, Triton effectively abstracts away all the issues related to concurrency</sup> *within* CUDA thread blocks (e.g., memory coalescing, shared memory synchronization/conflicts, tensor core scheduling).

While this may not be particularly helpful for embarrassingly parallel (i.e., element-wise) computations, it can greatly simplify the development of more complex GPU programs.

Consider for example the case of a fused softmax kernel (below) in which each instance normalizes a different row of the given input tensor *X_∈R_M_×_N*. Standard CUDA implementations of this parallelization strategy can be challenging to write, requiring explicit synchronization between threads as they concurrently reduce the same row of *X*. Most of this complexity goes away with Triton, where each kernel instance loads the row of interest and normalizes it sequentially using NumPy-like primitives.

Note that the Triton JIT treats X and Y as *pointers* rather than tensors; we felt like retaining low-level control of memory accesses was important to address more complex data structures (e.g., block-sparse tensors).

Importantly, this particular implementation of softmax keeps the rows of *X* in SRAM throughout the entire normalization process, which maximizes data reuse when applicable (~<32K columns). This differs from PyTorch’s internal CUDA code, whose use of temporary memory makes it more general but significantly slower (below). The bottom line here is not that Triton is inherently better, but that it simplifies the development of specialized kernels that can be much faster than those found in general-purpose libraries.

The lower performance of the Torch (v1.9) JIT highlights the difficulty of automatic CUDA code generation from sequences of high-level tensor operations.

Being able to write fused kernels for element-wise operations and reductions is important, but not sufficient given the prominence of matrix multiplication tasks in neural networks. As it turns out, Triton also works very well for those, achieving peak performance with just ~25 lines of Python code. On the other hand, implementing something similar in CUDA would take [a lot more effort(opens in a new window)](https://github.com/openai/blocksparse/blob/master/src/matmul_op_gpu.cu) and would even be likely to achieve lower performance.

One important advantage of handwritten matrix multiplication kernels is that they can be customized as desired to accommodate fused transformations of their inputs (e.g., slicing) and outputs (e.g., Leaky ReLU). Without a system like Triton, non-trivial modifications of matrix multiplication kernels would be out-of-reach for developers without exceptional GPU programming expertise.

The good performance of Triton comes from a modular system architecture centered around Triton-IR, an LLVM-based intermediate representation in which multi-dimensional blocks of values are first-class citizens.

The `@triton.jit` decorator works by walking the Abstract Syntax Tree (AST) of the provided Python function so as to generate Triton-IR on-the-fly using a common SSA construction algorithm. <sup>The resulting IR code is then simplified, optimized and automatically parallelized by our compiler backend, before being converted into high-quality LLVM-IR—and eventually PTX—for execution on recent NVIDIA GPUs. CPUs and AMD GPUs are not supported at the moment, but we welcome community contributions aimed at addressing this limitation.</sup>

We have found that the use of blocked program representations via Triton-IR allows our compiler to automatically perform a wide variety of important program optimizations. For example, data can be automatically stashed to shared memory by looking at the operands of computationally intensive block-level operations (e.g., `tl.dot`)—and allocated/synchronized using standard liveness analysis techniques.

On the other hand, Triton programs can be efficiently and automatically parallelized both (1) across SMs by executing different kernel instances concurrently, and (2) within SMs by analyzing the iteration space of each block-level operation and partitioning it adequately across different SIMD units, as shown below.

We intend for Triton to become a community-driven project. Feel free to fork our repository on [GitHub(opens in a new window)](https://github.com/openai/triton)!

*If you’re interested in joining our team and working on Triton & GPU kernels,* *we’re hiring**!*

## References

## Author

## Acknowledgments

Da Yan (HKUST), DeepSpeed (Microsoft), Anthropic

## Related articles

[View all](https://openai.com/news/)
