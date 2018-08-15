# SYCU - an implementation of SYCL over NVIDIA CUDA/AMD HIP
The goal of the SYCU project is to develop a SYCL 1.2.1 implementation that is built upon NVIDIA CUDA/AMD HIP.
Essentially, SYCU is a SYCL wrapper around CUDA/HIP, i.e. your applications are compiled with nvcc (NVIDIA) or hcc (AMD) like regular CUDA/HIP applications.

## Rationale
Present solutions (OpenCL, SYCL, CUDA, HIP) require choosing either portability or access to latest hardware features.
For example, when writing an application in CUDA, because of vendor support by NVIDIA, latest hardware features of NVIDIA GPUs can be exploited and there are sophisticated debugging/profiling tools available.
However, the application is then inherently unportable and will only run on NVIDIA GPUs.
If portability is more important, OpenCL can be used instead, which is however not well supported by some vendors, and some newer features are not implemented by these vendors. Here, especially NVIDIA stands out as a trouble spot.

SYCU attempts to solve these problems by providing a SYCL interface to CUDA/HIP. This means that
* You program your applications against the SYCL interface, which is an open standard. There are several other SYCL implementations available that allow for an execution of SYCL-compliant code on any OpenCL device, from CPUs to GPUs and FPGAs. In other words, if you don't want to run your code on GPUs from a certain vendor anymore, if it's written in SYCL, you can easily run it on any other device.
* Since SYCU is effectively a CUDA/HIP wrapper, you have full vendor support from NVIDIA (and AMD). Consequently, all CUDA/HIP profiling debugging tools are expected to work with SYCU.
* Since SYCU code relies on compilation with nvcc/hcc, you can easily create optimized code paths for the latest GPUs, and all the latest features that are available in CUDA will also be available to you.

## Current state
SYCU is still in an early stage of development. It can successfully execute some simple SYCL programs; but large parts of the specification are not yet implemented.

Still unimplemented/missing is in particular:
* hierarchical kernel dispatch
* ndrange kernel dispatch
* local memory
* Explicit memory copy functions
* Device library
* Images
* device/platform information queries


## Building SYCU
On nvidia, simply type
```
$ cd <build directory>
$ cmake <sycu source directory>
$ make
```
On AMD, at the moment hcc must be manually specified as cmake compiler.

## Caveats
* Since SYCU uses the vendor compilers nvcc (nvidia) and hcc (AMD) to compile code, all device functions must be marked with `__device__`, as is the case in cuda. This especially affects SYCL kernel lambdas. At the current time, SYCU can hence be thought of as a SYCL dialect. However, portability of SYCU code with other SYCL runtimes can be easily achieved by simply defining `__device__` during compilation. For future versions, it is planned to investigate the possibility of adding the `__device__` attributes automatically using libclang, such that regular sycl code can be compiled as well.
* SYCU uses AMD HIP as backend, which in turn can target CUDA and AMD devices. Due to lack of hardware, unfortunately I cannot test SYCU on AMD at the moment. Bug reports (or better, reports of successes) are greatly appreciated.
* Because SYCU doesn't build on OpenCL, all SYCL OpenCL interoperability features will very likely never be available in SYCU.

## Compiling software with SYCU
* On nvidia, SYCU provides a nvcc compiler wrapper that sets a couple of necessary compilation options for SYCU. This wrapper can be found in `bin/sycucc_nv`. Then just type `sycucc_nv -lsycu <args>`.
* On AMD, hcc must be invoked manually.

## Example
The following code adds two vectors:
```cpp
#include <cassert>
#include <iostream>

#include <CL/sycl.hpp>

#ifndef __SYCU__
// This guarantees compatibility if we are not using SYCU as SYCL implementation.
#define __device__
#endif

using data_type = float;

std::vector<data_type> add(cl::sycl::queue& q,
                           const std::vector<data_type>& a,
                           const std::vector<data_type>& b)
{
  std::vector<data_type> c(a.size());

  assert(a.size() == b.size());
  cl::sycl::range<1> work_items{a.size()};

  {
    cl::sycl::buffer<data_type> buff_a(a.data(), a.size());
    cl::sycl::buffer<data_type> buff_b(b.data(), b.size());
    cl::sycl::buffer<data_type> buff_c(c.data(), c.size());

    q.submit([&](cl::sycl::handler& cgh){
      auto access_a = buff_a.get_access<cl::sycl::access::mode::read>(cgh);
      auto access_b = buff_b.get_access<cl::sycl::access::mode::read>(cgh);
      auto access_c = buff_c.get_access<cl::sycl::access::mode::write>(cgh);

      cgh.parallel_for<class vector_add>(work_items,
                                         [=] __device__ (cl::sycl::id<1> tid) {
        access_c[tid] = access_a[tid] + access_b[tid];
      });
    });
  }
  return c;
}

int main()
{
  cl::sycl::queue q;
  std::vector<data_type> a = {1.f, 2.f, 3.f, 4.f, 5.f};
  std::vector<data_type> b = {-1.f, 2.f, -3.f, 4.f, -5.f};
  auto result = add(q, a, b);

  std::cout << "Result: " << std::endl;
  for(const auto x: result)
    std::cout << x << std::endl;
}

```



