---
title: An Exponential Moving Average Filter Using Only Addition and Bit Shift
subtitle: Or how I learned to stop worrying and love the bomb disposal robot.
date: 2021-11-14
word_count: 892 words ~3 minute read
---

This post is about a mathematical trick I found for implementing digital signal processing filters in compute-constrained environments. I derived it during my undergrad, during a module on embedded C for microcontrollers. The code can be found [here][ecm-link]. I was pretty new to C programming at the time, so the quality isn't great, however I remain proud of this trick.


<img align="right" width="40%" style="padding: 15px;" src="src/blog/filters/assets/bomb_disposal_robot.jpg">
The project brief was to build the hardware and software for a miniature bomb-disposal robot. The robot had to find an infrared beacon (the bomb), scan it for an RFID (the disarm code), and return home. To find the beacon, I needed to develop a signal classifier to determine whether the robot was looking at the beacon. The data from the IR sensors was noisy, but what I knew was that the signal was relatively stable when not looking at the beacon and very noisy when looking at it. This called for a high-pass filter. But on the 8-bit PIC18F4331 microcontroller I was using, mulitplication and division are expensive operations, and the memory is so limited that holding a moving window with hundreds of numbers is simply infeasible. How can you implement a high-pass filter when you can't use multiplication and don't want to store any data?

## EMA filter without multiplication

The exponential moving average filter is a good start, this filter is defined below, where $S_t$ is the smoothed value, $Y_t$ is the new data and $\alpha$ is the discount factor:

$$S_t = \alpha Y_t + (1 - \alpha)S_{t-1}$$

Although the exponential terminology can sound scary, this is arguably one of the simplest filters possible: simply summing the new data with the running count, multiplied by a discount factor. The 'exponential' part of the name comes from the fact that each time a new bit of data comes in, each item of data already seen gets multiplied by the discount factor again; making the filter equivalent to a weighted moving average filter (with an infinite window) where the weights diminish exponentially over time. This type of filter is typically used for smoothing, which makes sense intuitively, but we should note that it is mathematically an infinite impulse response low-pass filter.

The key insight here is that this filter only needs to store a single value as a running total - moreover, if we are clever about our choice of discount factor, we can avoid multiplication entirely and use only addition and bit shift operations!

Let $\alpha = \frac{1}{2^k}$, we can rewrite this as $\alpha = 1 >> k$, where $>>$ is the bit-shift operator. As long as we choose values of $\alpha$ that can be expressed this way, we can eliminate multiplication. For example, when $\alpha = 0.25$ :


$$S_t = S_{t-1} + (Y_t - S_{t-1}) >> 2$$

Finally, remember that the exponentially weighted moving average is a low-pass filter, so we can create a fast high-pass filter by subtracting the smoothed value from our raw signal:

$$\text{filtered} = Y_n - S_n$$

That's it! When you think of elegant mathematics, you might think of something more complicated. To me though, this is perfect. With only a little understanding of signal processing and algebra, we can make an algorithm that solves a real-world problem. Here's a C implementation of the algorithm, combined with a simple threshold to classify whether the bomb-disposal robot is looking at the beacon:

```c
int classify_data(unsigned int raw_data) {
  static unsigned int smoothed;
  const int THRESHOLD = 100 // arbitrary, found with trial and error

  smoothed = smoothed + ((raw_data - smoothed) >> 2);
  unsigned int filtered = raw_data - smoothed;

  if (filtered >= THRESHOLD) {
    return 1;
  } else {
    return 0;
  }
}
```

Although this is C code, I feel a real Pythonic zen about this trick... Simple is better than complex, Complex is better than complicated...

## Conclusions

Being good at maths makes you better at everything -- especially programming -- just ask the makers of [Quake III][fast-inv-root]. When you code on powerful machines with high level languages, it is easy to hide from maths and convince yourself that it's not an essential part of the job. This is not the case in embedded development. Finding mathematical tricks is essential to making programs run when computing power is limited and the C language lends itself nicely to finding them. There's something special about knowing that the heart of your program contains a small piece of genuis that no-one else will ever know about (unless of course you write about it in your blog).

I am not the first person to develop this trick and it's difficult to determine where the first example of this came from. The method is not too tricky to derive, so I suspect it gets rediscovered all the time. In fact, not only am I not the first person to use this trick, I'm not even the first person to blog about it! For futher reading, I can thoroughly recommend Pieter's Blog, with posts going into more depth on the [exponential moving average algorithm][ema-blog] and [efficient implementations][em-implementation] of it.

[fast-inv-root]: https://en.wikipedia.org/wiki/Fast_inverse_square_root
[ecm-link]: https://github.com/Charl-AI/Bomb-Disposal-Robot
[ema-blog]: https://tttapa.github.io/Pages/Mathematics/Systems-and-Control-Theory/Digital-filters/Exponential%20Moving%20Average/Exponential-Moving-Average.html

[em-implementation]: https://tttapa.github.io/Pages/Mathematics/Systems-and-Control-Theory/Digital-filters/Exponential%20Moving%20Average/C++Implementation.html
