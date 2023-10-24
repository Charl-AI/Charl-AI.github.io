---
title: Eliminating Dataloading Bottlenecks in PyTorch with Stochastic Caching
subtitle: Getting the most out of scrappy setups.
date: 2023-10-23
word_count: X words ~Y minute read
generate_toc: true
---

GPUs are expensive. If we have a bottleneck in our dataloading pipeline (i.e. training is not compute-bound), we are leaving performance and money on the table. The trouble is, modern GPUs are fast. Unless you're training large models, loading imaging data from an HDD or over a network tends to result in dataloading bottlenecks.

While it would be great if we could have all our data on NVMe SSDs, or in super high bandwidth data stores, in practice, many of us have cheap and scrappy setups. For example, in our lab, a lot of our prototyping is run on a loose assortment of GPU workstations, with data loaded from a server over a network. This is extremely data-bottlenecked. In fact, when many people are reading data from the same server at once, training noticeably slows down.

This post walks you through a caching trick for improving network dataloading for deep learning training. The trick is not new (in fact, it seems to have originated with the legendary [@ptrblck in 2018](https://github.com/ptrblck/pytorch_misc/blob/master/shared_array.py)), however it seems underutilised, and is especially useful for people with imperfect/cheap/scrappy setups. If you have a well-funded industrial lab, have all your data on a local NVMe SSD, or are training mostly on DGX clusters, it's probably safe for you to ignore this post.

I refactored the main contribution of this post into a dead-simple library that you can install with pip (or just copy-paste into your code). You can find the GitHub page [here](), or try it out with:

```bash
pip install stochaching
```

## The naive solution (doesn't work)

What we want is to lazily cache images in RAM the first time they are read in `__getitem__()` . If we can do this right, the first epoch will still be bottlenecked by dataloading, but future epochs will be fast, as they simply need to retrieve the images from the cache in RAM.

A naive implementation would look something like this:

```python
class NaiveDataset(Dataset):
  def __init__(self) -> None:
    super().__init__()

    ... # other dataset logic

    self.cache = [] # will store the cached imgs
    self.cached_idxs = set() # keeps track of what has been cached

  def __getitem__(self, idx: int) -> tuple[Tensor, Tensor]:
    if idx not in self.cached_idxs:
      img = ... # read image into a float32 tensor
      self.cache.append(img)
      self.cached_idxs.add(idx)
    else:
      img = self.cache[idx]

    img = transforms(img)
    label = ... # logic for loading the label
    return img, label
```

The key problem with any method that implements caching like this is that the `NaiveDataset` object gets copied for each worker in your dataloader. This means that you'll have to keep several whole copies of your dataset in RAM. Worse still, since each worker sees only a subset of the dataset each epoch, it will take many epochs for each worker to have filled its own cache. Outside of the special case where `num_workers=0`, this method is inefficient in memory and provides only a marginal speed up.

What we need is a method for sharing the cache between all worker's copies of the `Dataset`...

## @ptrblck's solution

Fortunately, it is possible to share data between multiple objects in python -- @ptrblck shared [an implementation](https://github.com/ptrblck/pytorch_misc/blob/master/shared_array.py) for doing this in 2018.

The implementation looks something like this:

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

    img_shape = ... # (C, H, W)
    num_samples = ... # length of dataset

    shared_array_base = mp.Array(
      ctypes.c_float, num_samples*np.prod(img_shape)
    )
    shared_array = np.ctypeslib.as_array(shared_array_base.get_obj())
    shared_array = shared_array.reshape(num_samples, *img_shape)
    self.shared_array = torch.from_numpy(shared_array)
    self.use_cache = False

  def set_use_cache(self, use_cache: bool):
      self.use_cache = use_cache

  def __getitem__(self, idx: int) -> tuple[Tensor, Tensor]:
    if not self.use_cache:
      print('Filling cache for index {}'.format(idx))
      img = ... # read image into a float32 tensor
      self.shared_array[idx] = img
    img = self.shared_array[idx]

    img = transforms(img)
    label = ... # logic for reading the label
    return img, label

# train for one epoch, call ds.set_use_cache(True),
# then continue training
```

This works pretty great! It gets around the data copy problem from the naive solution and will form the backbone of what we will do going forwards. Now, let's focus on expanding the method to deal with datasets too large to fit in the cache.

## Stochastic caching

The maximum size of the shared array that we can store is equal to the size of `/dev/shm/`, you can find this with `df -h` (and you can temporarily increase it with `mount -o remount,size=100G /dev/shm`). If our dataset is larger than this, what should we do?

One natural option is to specify the size of the cache we want, and cache only a subset of images that fits into that size. For simplicity of implementation, we can simply calculate how many images will fit (N), then cache the first N images by `idx`. When training with a shuffled dataset, the cached images will be dispersed throughout each batch. As long as we're caching a high enough proportion of images, we should be able to shift the bottleneck off the dataloading. I call this approach **stochastic caching**.

We can calculate how many images to cache like so:

```python
max_cache_size_gib = ... # set to the size of /dev/shm

# 32 bit imgs (float32), 8 bits per byte
ds_size = np.prod(img_shape) * num_samples * 32 / 8
if ds_size > max_cache_size_gib * 1e9:
  imgs_to_cache = (
    int(max_cache_size_gib * 1e9
    / (np.prod(img_shape) * 32 / 8))
  )
  print(
    f"Dataset ({ds_size / 1e9:.2f}GiB) larger than"
    + f" the cache limit ({max_cache_size_gib}GiB),"
    + f"Lazily caching {imgs_to_cache} / {num_samples} images."
  )
else:
  imgs_to_cache = num_samples
  print(f"Lazily caching full dataset"
    + f" ({ds_size / 1e9:.2f}GiB) in RAM.")
```

The above snippet will go in `__init__()`. From here, the implementation is basically the same as before, with a couple of changes to the `__getitem__()` method:

```python
def __getitem__(self, idx: int) -> tuple[Tensor, Tensor]:
  # never cache images out of bounds of the cache array
  # i.e. only cache first N imgs by idx
  if idx >= len(self.shared_array):
    img = ... # read image into a float32 tensor
  elif not self.use_cache:
    print('Filling cache for index {}'.format(idx))
    img = ... # read image into a float32 tensor
    self.shared_array[idx] = img
  else:
      img = self.shared_array[idx]

  img = transforms(img)
  label = ... # logic for reading the label
  return img, label
```

Now we're cooking with gas! Let's go a bit further now. We'll add some optimisations and convenience features to make it a bit more usable.

## Optimisations for convenience and efficiency

So far, we have had to set a `self.use_cache` flag after the first epoch to signal whether we are reading from the cache or not. This is kind of ugly. A simple hack to avoid this is to initialise our shared array to all zeros. In `__getitem__()`, we can now check `all(self.shared_array[idx] == 0)` to see if we've stored anything in that cache slot yet.

Also notice that so far, we have been caching `float32` tensors. This is kind of unnecessary, since the images probably started in `uint8` format (like `png` or `jpg`) anyway. We can cut our memory usage by 3x by simply caching in `uint8` format, then transforming to `float32` tensors later.

This actually leads us onto an important point about transformations. We should not perform any augmentations (i.e. random transformations) before caching. We would unintentionally be killing the randomness and defeating the entire point of the augmentations. When caching a dataset, it makes sense to split your pipeline into two parts: a 'transformations' stage, the output of which will be cached; and an 'augmentations' stage, which will be performed on-the-fly. In general, the transformations stage should map PIL images to `uint8` tensors, doing any expensive operations like `Resize`. The augmentations stage should map `uint8` tensors to normalised `float32` tensors, doing any stochastic operations like `RandomRotation`.

An implementation that includes these optimisations will look like this (for the transformations, we demonstrate usage of the 'v2' API of torchvision.transforms):

```python
from PIL import Image
import ctypes
import multiprocessing as mp

import numpy as np
import torch
from torch.utils.data import Dataset
from torchvision.transforms import v2


def get_transforms() -> Callable[[Image.Image], Tensor]:
  transform_list = [
    v2.ToImageTensor(),
    v2.ToDtype(torch.uint8),
    v2.Resize(256, antialias=True),
  ]

  return v2.Compose(transform_list)


def get_train_augmentations() -> Callable[[Tensor], Tensor]:
  aug_list = [
    v2.RandomResizedCrop(224, antialias=True),
    v2.RandAugment(),
    v2.ToDtype(torch.float32),
    v2.Normalize(mean=IMAGENET_MEAN, std=IMAGENET_STD),
  ]
  return v2.Compose(aug_list)


class StochasticCacheDataset(Dataset):
  def __init__(self) -> None:
    super().__init__()

    ... # other dataset logic
    self.transforms = get_transforms()
    self.augmentations = get_train_augmentations()

    img_shape = ... # (C, H, W)
    num_samples = ... # length of dataset

    max_cache_size_gib = ... # set to the size of /dev/shm

    # 8 bit imgs (uint8), 8 bits per byte
    ds_size = np.prod(img_shape) * num_samples * 8 / 8
    if ds_size > max_cache_size_gib * 1e9:
      imgs_to_cache = (
        int(max_cache_size_gib * 1e9
        / (np.prod(img_shape) * 8 / 8))
      )
      print(
        f"Dataset ({ds_size / 1e9:.2f}GiB) larger than"
        + f" the cache limit ({max_cache_size_gib}GiB),"
        + f"Lazily caching {imgs_to_cache} / {num_samples} images."
      )
    else:
      imgs_to_cache = num_samples
      print(f"Lazily caching full dataset"
        + f" ({ds_size / 1e9:.2f}GiB) in RAM.")

    shared_array_base = mp.Array(
      ctypes.c_uint8, imgs_to_cache*np.prod(img_shape)
    )
    shared_array = np.ctypeslib.as_array(
      shared_array_base.get_obj()
    )
    shared_array = shared_array.reshape(imgs_to_cache, *img_shape)
    self.shared_array = torch.from_numpy(shared_array)

    # initialise all to 0 (we check this later to see
    # if anything has been cached in each slot)
    self.shared_array *= 0

  def __getitem__(self, idx: int) -> tuple[Tensor, Tensor]:
    # never cache images out of bounds of the cache array
    if idx >= len(self.shared_array):
      img = ... # read image into PIL uint8 img
      img = self.transforms(img) # to torch uint8 tensor

    # check if image has been cached yet
    elif all(self.shared_array[idx] == 0):
      print('Filling cache for index {}'.format(idx))
      img = ... # read image into a PIL uint8 img
      img = self.transforms(img) # to torch uint8 tensor
      self.shared_array[idx] = img

    else:
        img = self.shared_array[idx]

    img = augmentations(img) # to float32 tensor
    label = ... # logic for reading the label
    return img, label
```

Okay, now we have something that works quite nicely in the real-world! The only problem is that the code is quite long and fiddly. It would be nice if we could refactor this into some reusable components...

## The `stochaching` library

## Conclusions

For our particular use case (loading data from a slow network), this trick has been pretty successful. I'm honestly not sure why it hasn't caught on in deep learning libraries and codebases. I suspect it's because most people building deep learning libraries are working at well-funded labs and have optimised the libraries for _their_ workflows. They probably never needed to consider anything as janky as reading data off slow network, so never thought to implement these tricks. For those of us who are less fortunate, we need to develop our own workarounds -- code optimised for someone else's workflow may not be optimal for yours.
