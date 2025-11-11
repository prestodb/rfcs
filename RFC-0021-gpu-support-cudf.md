# **RFC-0021 GPU support using cuDF in C++ workers**

Proposers

* Deepak Majeti, Zoltan Arnold Nagy, et. al. (IBM)
* Karthikeyan Natarajan, et. al. (NVIDIA)

## [Related Issues]

* [PR #25094](https://github.com/prestodb/presto/pull/25094): Enable Velox cuDF
* [PR #26156](https://github.com/prestodb/presto/pull/26156): Add support for Velox cuDF options and CudfHiveConnector

## Summary

Enable C++ workers to execute queries on GPUs. 

## Background

There is now a proliferation of GPU hardware primarily due to the demands from AI/ML usecases.
We are now seeing improved GPU hardware with advanced I/O capabilities.
New AI adjacent data processing workflows have also evolved recently.

GPUs provide high compute and memory bandwidth, which can benefit operations such as
joins, aggregations, string processing, etc.

Nvidia's [blog](https://developer.nvidia.com/blog/accelerating-large-scale-data-analytics-with-gpu-native-velox-and-nvidia-cudf/)
has more details.


### Goals
* Allow Presto queries to run on a GPU.

## Proposed Implementation

Most of the work has been already implemented in [Velox](https://github.com/facebookincubator/velox/tree/main/velox/experimental/cudf).

* Parquet file-format is supported. cudfHiveConnector is registered.
* S3 and local/linux filesystems are supported.
* CMake build option `PRESTO_ENABLE_CUDF` must be set.
* cuDF [configs](https://facebookincubator.github.io/velox/configs.html#cudf-specific-configuration-experimental) can be 
specified inside `config.properties` and catalog `.properties` file.

## Releases
Presto C++ workers will be released with GPU support.

### Future Work
* Optimizer plan changes to benefit GPUs.
* Scheduler changes:
    * Metadata queries on CPU only.
    * Multi-GPU support.
* Session parameter to enable GPU execution.
* GPU-GPU exchange support.

## Test Plan
NVIDIA wants to donate GPU compute that we can use in Presto CI.
