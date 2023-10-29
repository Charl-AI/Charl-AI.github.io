---
title: Eliminating Dataloading Bottlenecks in PyTorch with Stochastic Caching
subtitle: A zero-effort speedup for dataloading-bottlenecked applications
date: 2023-10-23
word_count: X words ~Y minute read
generate_toc: true
---

If you're training small to medium size vision models and are loading data from an HDD (or worse, over a network), dataloading is probably a bottleneck for you.

Oftentimes, accelerating dataloading requires significant engineering effort. Tricks like [storing data in contiguous raw blocks](https://docs.ffcv.io/working_with_images.html#working-with-image-data-in-ffcv) or [writing custom collate functions](https://github.com/huggingface/pytorch-image-models/blob/main/timm/data/loader.py) can work really nicely, but often require you to tailor the tricks for each dataset you work with. It would be great if we could get the benefits of hand-engineered dataloading pipelines without actually having to do the engineering!

This post walks you through a trick that I call stochastic caching. I'll introduce the concept and build you up from a naive solution to a fully-fledged library. Importantly, implementing stochastic caching with this library requires minimal code changes and works on any dataset or machine. I've released the library on PyPI, under the name `stocaching`. You can find it on [GitHub](https://github.com/Charl-AI/stochastic-caching), or install it with pip:

```bash
pip install stocaching
```

Stocaching is dead-simple (only 1 file!) -- if you don't want to install it with pip, you can just copy-paste it into your projects.

## What is stochastic caching?

We all know that (for multi-epoch training) we should cache small datasets in RAM to speed up dataloading. However, once we get to larger datasets, we often give up on caching completely. This seems like a waste!

If we are training on large, shuffled datasets, we can just cache a subset of the data that fits in RAM. The cached data will then be randomly distributed throughout each batch (as shown in the ASCII figure below). This reduces the I/O workload on the disk and, if we cache a large enough proportion of our data, should move the bottleneck off the dataloading pipeline.

```
┌─────────────────┐┌─────────────────┐┌─────────────────┐
│ Batch 1         ││ Batch 2         ││ Batch 3         │
│  ┌──┬──┬──┬──┐  ││  ┌──┬──┬──┬──┐  ││  ┌──┬──┬──┬──┐  │
│  │┼┼│  │  │┼┼│  ││  │  │┼┼│  │┼┼│  ││  │  │┼┼│┼┼│┼┼│  │ ...
│  │┼┼│  │  │┼┼│  ││  │  │┼┼│  │┼┼│  ││  │  │┼┼│┼┼│┼┼│  │
│  └──┴──┴──┴──┘  ││  └──┴──┴──┴──┘  ││  └──┴──┴──┴──┘  │
└─────────────────┘└─────────────────┘└─────────────────┘
  # = Sample cached in RAM
```

I call this trick stochastic caching. It's conceptually simple and easy to implement. Given this simplicity, it's surprising how effective it can be. Stochastic caching can:

1. Provide dataloading speedups for datasets too large to fit in memory.
2. Be lazy (i.e. only cache samples the first time they're needed).
3. Give speedups that scale linearly with the % of the dataset being cached.

## The naive solution (doesn't work)

What we want is to lazily cache samples in RAM the first time they are read in `__getitem__()` . If we can do this right, the first epoch will still be bottlenecked by dataloading, but future epochs will be faster, as some of the data can be retrieved from RAM instead of disk.

A naive implementation would look something like this:

```python
class NaiveDataset(Dataset):
  def __init__(self) -> None:
    super().__init__()

    ... # other dataset logic

    cache_length = N # number of samples you want to cache
    data_dims = (C, H, W) # shape of data (not including batch)

    self.cache = torch.empty(
      (cache_length, *data_dims), dtype=torch.float32
    )
    self.cached_idxs = set() # keeps track of what has been cached

  def __getitem__(self, idx: int) -> tuple[Tensor, Tensor]:
    if idx >= cache_length:
      x = ... # read image to float32 tensor
    elif idx not in self.cached_idxs:
      x = ... # read image to float32 tensor
      self.cache[idx] = x
      self.cached_idxs.add(idx)
    else:
      x = self.cache[idx] # get float32 image tensor from cache

    x = transforms(x)
    label = ... # load label
    return x, label
```

The key problem with any method that implements caching like this is that the `NaiveDataset` object gets copied for each worker in your dataloader. This means that you'll have to keep several whole copies of your dataset in RAM. Worse still, since each worker sees only a subset of the dataset each epoch, it will take many epochs for each worker to have filled its own cache. Outside of the special case where `num_workers=0`, this method is inefficient in memory and provides only a marginal speed up.

What we need is a method for sharing the cache between all worker processes...

## Caching in shared memory

Fortunately, it is possible to share data between multiple processes in python using `multiprocessing`. In fact, I started developing this project after finding an implementation for shared memory datasets that @[ptrblck](https://github.com/ptrblck/pytorch_misc/blob/master/shared_array.py) shared in 2018.

Adapting this implementation gets us to something like this:

```python
import ctypes
import multiprocessing as mp

import numpy as np
import torch
from torch.utils.data import Dataset

class CacheDataset(Dataset):
  def __init__(self) -> None:
    super().__init__()

    ... # other dataset logic

    cache_length = N # number of samples you want to cache
    data_dims = (C, H, W) # shape of data (not including batch)

    shared_array_base = mp.Array(
      ctypes.c_float, num_samples*np.prod(data_dims)
    )
    shared_array = np.ctypeslib.as_array(shared_array_base.get_obj())
    shared_array = shared_array.reshape(cache_length, *data_dims)
    self.cache = torch.from_numpy(shared_array)
    self.cache *=0

  def __getitem__(self, idx: int) -> tuple[Tensor, Tensor]:
    if idx >= cache_length:
      x = ... # read image to float32 tensor
    # hack to see if cache slot has changed since initialisation
    elif not torch.all(self.cache == 0):
      x = ... # read image to float32 tensor
      self.cache[idx] = x
    else:
      x = self.cache[idx] # get float32 image tensor from cache

    x = transforms(x)
    label = ... # load label
    return img, label
```

This works pretty great! It gets around the data copy problem from the naive solution and will form the backbone of what we will do going forwards. Now, let's extract this idea into a simple library that can be dropped into any codebase for a free speedup.

## The `stocaching` library

So far, we have a cool trick that requires quite a bit of fiddly code and is not massively portable across datasets. If we refactor the shared memory cache into a standalone module, add some convenience features, and handle the edge-cases, we end up with the `stocaching` library, available now on [GitHub](https://github.com/Charl-AI/stochastic-caching) and [PyPI](https://pypi.org/project/stocaching/#description).

The main contribution of `stocaching` is the `SharedCache` class. Using it is super simple, and is best demonstrated through a minimal example:

```python
import torch
from stocaching import SharedCache
from torch.utils.data import Dataset

class MyDataset(Dataset):
    def __init__(self):
        super().__init__()

        ... # set up dataset

        dataset_len = N   # number of samples in the full dataset
        data_dims = (C, H, W)   # data dims (not including batch)

        # initialize the cache
        self.cache = SharedCache(
            size_limit_gib=32,
            dataset_len=dataset_len,
            data_dims=data_dims,
            dtype=torch.uint8,
        )

    def __getitem__(self, idx):
        # retrieve data from cache if it's there
        x = self.cache.get_slot(idx)
        # x will be None if the cache slot was empty or OOB
        if x is None:
            x = ... # load data to uint8 tensor from disk
            self.cache.set_slot(idx, x) # try to cache x
        return x
```

When initialising a `SharedCache` object, we tell it about the size of our dataset and the maximum amount of memory we want the cache to take up. `SharedCache` then calculates how many samples can fit in the cache and allocates space accordingly.

You can interact with `SharedCache` through `__getitem__` and `__setitem__` methods, or even by accessing the underlying PyTorch tensor. For convenience, we expose the `get_slot` and `set_slot` methods, which can gracefully handle cases where they are passed an out-of-bounds index. This design helps to reduce the amount of code necessary for common use cases.

**Importantly, when using `SharedCache`, you don't have to worry about whether the dataset is being fully or partially cached.** Just by changing the `size_limit_gib` parameter, you can run the same code on any machine, getting benefits depending on how much RAM you have. Even caching only 10% of your dataset can still give noticeable speeups when you are very data-bottlenecked (such as when training small models).

## Benchmarks

I've run some basic benchmarks to test the method under single GPU image classification workloads. More details can be found in the [repo](https://github.com/Charl-AI/stochastic-caching/blob/main/benchmark/dataset.py), but the headline results are here:

|          Local HDD          |         Remote data          |
| :-------------------------: | :--------------------------: |
| ![](src/blog/dataloaders/assets/local_sweep.png) | ![](src/blog/dataloaders/assets/remote_sweep.png) |

On both local HDD and network drive workloads, I found that we get speedups which scale linearly with the % of the dataset being cached. While there is a small overhead in the first epoch from filling the cache, this is quickly compensated for by the speedup in the second epoch. Even caching very small percentages of the data (5-10 %) seems beneficial over multiple epochs.

## Tips and tricks

**1. How do I decide how much memory to allocate to the cache?**

As much as you like! The speedup from caching scales linearly with the % of your dataset being cached. When the `SharedCache` object is created, it will print out the calculated size of your dataset and how many samples it decided to cache.

The shared memory is stored in `/dev/shm` (tmpfs), so this is likely the limiting factor for you. We provide a convenience function `get_shm_size()` to check how large it is. Alternatively, check with `df -h`.

Most Unix-like systems have `/dev/shm` pre-set to 50% of your RAM. You can temporarily resize it (e.g. to 128 GiB) by running: `mount -o remount,size=128G /dev/shm`.

**2. How should I organise my transforms/augmentations?**

Generally, you don't want to do any random augmentations before caching because the cache will kill the randomness. It's also a good idea to cache data in the smallest dtype you can to save space. For example, if you are reading images from jpg/png, you can usually cache them in uint8 format instead of float32.

Splitting your transforms/augmentation pipeline into two phases is a good idea. The first phase converts your data to a (possibly resized) uint8 tensor. The output of this phase gets cached. The second phase should do random augmentations, convert to float32, and normalise. This phase happens 'on-line' and the output goes straight into your model.

For a working example of how to do this properly, see the benchmark implementation [here](https://github.com/Charl-AI/stochastic-caching/blob/main/benchmark/dataset.py). 

**What about multi GPU (DDP) training?**

The current implementation *can* work for DDP training, however, it may not be as efficient. Each GPU process would get its own cache and the caches would overlap. I'm still working on the library and haven't focused too much yet on the DDP use case.

It shouldn't be too difficult to extend the library to cover the DDP case. My current thinking is to make a second class, `SharedCacheDDP`, which will use the `multiprocessing.shared_memory.SharedMemory` feature. This feature would allow the cache to be more easily shared across the GPU processes, however, would require a bit more attention from the user.


## Conclusions

GPUs are expensive. If we have a bottleneck in our dataloading pipeline, we are leaving performance and money on the table. 

For dataloading-bottlenecked image classification workloads, my testing has shown this trick to be pretty successful. I'm honestly not too sure why I haven't seen anything similar in deep learning libraries and codebases, but I'm now starting to use it in my research.  

If you have any comments or suggestions (or even want to contribute to the codebase!), feel free to open an issue on the GitHub repo.

