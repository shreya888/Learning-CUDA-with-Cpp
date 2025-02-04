# Learning CUDA with PyTorch: Notes and Code
This repository documents key concepts, exercises, and insights gained from my exploration of CUDA (Compute Unified Device Architecture) programming, with examples in PyTorch. This folder focuses on understanding and implementing CUDA to enhance code performance in PyTorch. The repository contains commented code examples, conceptual explanations, and a curated list of resources.

## Structure
Code and logs in seperate folders with README.md explaning code.

## Key Learning Points (PyTorch CUDA)
1. **Cuda is Asynchronous**
   * Asynchronous programming is a technique that allows a program to start a long-running task while still being able to respond to other events.
   * CUDA operations, including kernel launches and memory transfers, are executed asynchronously with respect to the host (CPU). This means that after launching a task on the GPU, the CPU can continue executing other instructions without waiting for the GPU to complete its task. This parallelism between CPU and GPU can significantly improve performance in computational workloads by maximizing hardware utilization.
   * Python's built-in `time` module is insufficient for measuring actual GPU execution time. To accurately measure the duration of CUDA kernel execution, `torch.cuda.Event` is used.
   * Placed at specific points in the code, CUDA events are synchronization markers that can be used to track the progress of GPU execution, to accurately measure timing, to ensure that the CPU waits for GPU tasks to complete (synchronize them) and allows for precise timing of asynchronous operations.
   * **Key Insight:** The asynchronous nature of CUDA operations, while beneficial for performance, requires careful handling of synchronization when measuring execution times or ensuring correct data dependencies between operations.

2. **Warmup run**
   * It is standard practice to perform a "warmup run" before benchmarking CUDA kernels.
   * This is due to several initialization costs that may be incurred during the first kernel invocation, such as memory allocation, initialization of CUDA streams, or Just-In-Time (JIT) compilation of the kernel code. Sometimes it’s also referred to as “cooking” a kernel.
   * The JIT compilation can take several seconds for complex kernels, and without a warmup, the initial kernel launch may give inaccurate performance measurements leading to incorrect benchmarking.
   * Practical Implementation: Call each kernel at least once before starting the performance measurements to avoid any bias introduced by initialization.
   * **Analogy**: The concept of a warmup run is akin to clearing the caches or pre-allocating necessary resources, ensuring that the subsequent measurements reflect steady-state performance without the overhead of initialization.
   * **Analogy**: Just as athletes warm up before a race or musicians tune their instruments before a performance, warming up the GPU avoids unnecessary delays caused by setup processes during benchmarking.

3. **Synchronize the code**
   * CUDA operations are inherently asynchronous, meaning that the host (CPU) may continue executing code while GPU kernels are still running. This poses a challenge when accurate timing of operations or data consistency between CPU and GPU is required.
   * To enforce synchronization, one can use the `torch.cuda.synchronize()` function, which blocks the host from advancing until all GPU operations on the current device have completed.
   * This is critical for performance profiling or when data produced by GPU computations must be used on the CPU. Without proper synchronization, results may be incorrect or incomplete due to unfinished GPU tasks.
   * Additionally, CUDA streams, which are sequences of operations executed in order on a GPU, allow for more fine-grained control of task execution. Each stream operates independently, and synchronization can be applied on a per-stream basis using `stream.synchronize()`, providing flexibility in managing concurrent GPU workloads.
   * **Key Insight:** Synchronization is a necessary step to ensure that asynchronous operations on the GPU have completed before proceeding with further computations, especially for tasks involving interdependent operations between the CPU and GPU.

4. **Pytorch Autograd Profiler**
    * Optimization of CUDA code generally begins with identifying performance bottlenecks/hotspots via profiling. PyTorch provides a built-in profiler `torch.autograd.profiler` to monitor and analyze performance, both on the CPU and GPU.
    * This profiler is crucial for identifying the most time-consuming operations, allowing developers to focus optimization efforts where they will have the most significant impact.
    * `torch.autograd engine` keeps a record of execution time of each operator under this line of code - `with torch.autograd.profiler.profile() as prof:`.
    * To profile CUDA operations, provide argument `use_cuda=True` to the profile function. The profiler records the execution time of each operator, including the time spent on kernel launches and GPU execution. The results can be analyzed to identify performance bottlenecks or inefficiencies.
    * **Key Insight:** Profiling is an essential tool in CUDA development. PyTorch's autograd profiler can obtain in-depth insights into the performance of both CPU and GPU operations, allowing for targeted optimizations.

5. **ATen Library (`atten::<some_function_name>`)**
   * ATen (Abstract Tensor) is PyTorch's backend library for tensor operations. It provides the low-level C++ implementation of mathematical operations on tensors.
   * In profiling, ATen functions (e.g., `aten::add`, `aten::matmul`) represent the underlying operations called by higher-level PyTorch functions. These operations eventually translate into CUDA kernel calls for execution on the GPU.
   * The profiler log shows a hierarchical breakdown of operations, starting from high-level calls down to specific CUDA instructions.
  
6. **Understanding the log file (`pytorch_abs.log`)**
   * When interpreting the profiler logs, it’s important to understand the different behaviors of CPU and GPU operations. For this log, it's observed that the CPU run time (in us) is far lesser than GPU (in ms). Some possible reasons for this are below:
      * CPU calls appear faster in the PyTorch Autograd profiler because they are synchronous and execute immediately. The profiler can measure their runtime directly.
      * GPU calls, on the other hand, are asynchronous and involve overhead from kernel launches, memory transfers, and task queuing. This can make GPU function calls appear slower unless you explicitly synchronize the CPU and GPU.
      * For small tasks or operations that don’t fully leverage the GPU’s parallel power, the overhead of launching GPU kernels can outweigh the benefits of parallelism, making the CPU appear faster in these cases.
   * When dealing with large-scale computations, such as matrix multiplications or convolutions in deep learning, GPUs outperform CPUs by several orders of magnitude.
   * In practice, GPUs excel in large-scale parallel computations, and for tasks that are sufficiently complex, such as matrix multiplications or convolutions in deep learning, GPUs outperform CPUs by several orders of magnitude, GPUs outperform CPUs by several orders of magnitude.
   * **Key Takeaways:**
     * For small, frequent operations, CPU execution may appear faster due to lower overhead.
     * For larger workloads, where the GPU’s parallel processing can be fully utilized, the GPU’s performance advantages become evident.
     * Synchronizing CPU and GPU using torch.cuda.synchronize() is crucial when measuring the execution time of CUDA operations.

