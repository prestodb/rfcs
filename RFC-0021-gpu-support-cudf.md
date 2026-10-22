# **RFC-0021 GPU support using cuDF in C++ workers**

### Proposers

* Deepak Majeti, et. al. (IBM)
* Zoltan Arnold Nagy, et. al. (IBM Research Europe)
* Karthikeyan Natarajan, et. al. (NVIDIA)

## Related Issues

* [PR #25094](https://github.com/prestodb/presto/pull/25094): Enable Velox cuDF
* [PR #26156](https://github.com/prestodb/presto/pull/26156): Add support for Velox cuDF options and CudfHiveConnector

## Summary

Enable C++ workers to execute queries on GPUs. 

## Background

There is now a proliferation of GPU hardware primarily due to the demands from AI/ML usecases.
GPU hardware over the years has evolved with advanced I/O capabilities.
New AI adjacent data processing workflows are also being developed.

GPUs provide high compute and memory bandwidth, which can benefit operations such as
joins, aggregations, string processing, etc.


### Goals
* Allow Presto queries to run on a single GPU or multiple GPUs.
* A query will run either on the CPU or a GPU. No hybrid execution.
* Use CPU if a GPU lacks a certain functionality.
* Execution should maximize utilization of available hardware such as NVLink.

## Proposed Implementation

Some of this work has been implemented in [Velox](https://github.com/facebookincubator/velox/tree/main/velox/experimental/cudf).
The current implementation translates the CPU operators to the GPU operators via a DriverAdapter in Velox.

Nvidia's [blog](https://developer.nvidia.com/blog/accelerating-large-scale-data-analytics-with-gpu-native-velox-and-nvidia-cudf/)
has more details on the design and some early results.

The [Extending Velox - GPU Acceleration with cuDF](https://velox-lib.io/blog/extending-velox-with-cudf) blog also covers the current implementation.

On the Presto C++ side, the following registrations and configs have been added.

* CMake build option `PRESTO_ENABLE_CUDF` must be set. https://github.com/prestodb/presto/tree/master/presto-native-execution#nvidia-cudf-gpu-support
* Parquet file-format is supported. cudfHiveConnector is registered.
* S3 and local/linux filesystems are supported.
* cuDF [configs](https://facebookincubator.github.io/velox/configs.html#cudf-specific-configuration-experimental) can be 
specified inside `config.properties` and catalog `.properties` file.

The current work so far shows that GPUs can provide good price-performance. However, to make this support user-friendly and get better price-performance, the following improvements are in progress.

## Work in Progress
* Add GPU plan nodes.
    * Driver adapter runs after the drivers/pipelines are built. Limits the adaptation.
    * Allow efficient fallback to CPU.
* GPU-GPU exchange using UCX (https://github.com/prestodb/presto/tree/ibm-research-preview).
* Topology and hardware detection.
* Metadata queries on CPU only.
* Session parameter to filter workers.
* Optimizer cost model to support GPUs.

## Releases
Presto C++ workers will be released with GPU support.

## Test Plan
Velox CI has a gpu runner sponsored by Meta. We need a similar runner for Presto.
