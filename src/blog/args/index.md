---
title: Configuring Hyperparameters with Decorators
subtitle: A new pattern for hackable research code in Python.
date: 2025-06-29
word_count: 1749 words ~6 minute read
generate_toc: false
---

I think that a serious obstacle to writing good research code is managing configuration. Not just at the interface of our scripts -- that part _is_ tricky, but can be managed with proper use of tools like argparse or hydra -- but in the logic itself. For example, consider the following program sketch:

```python

def guided_prediction(model, t, x, y, omega):
    """Predict with classifier-free guidance."""
    cond = model(t,x,y)
    uncond = model(t,x,None)
    return cond + omega * (cond - uncond)

def generate_image(model, x0, y1, n_steps, omega):
    """Generate using the Euler probability flow ODE solver."""
    x1 = x0
    dt = 1 / n_steps
    t = 0
    for _ in range(n_steps):
        t += dt
        x1 += guided_prediction(model,t,x1,y1,omega)
    return x1

def compute_loss(model, x0, x1, y1, sigma):
    """Flow matching (linear interpolant) loss."""
    t, eps = ... # sample t from U[0,1], eps from N(0,1)
    mu = t * x1 + (1-t) * x0
    ut = x1 - x0
    xt = mu + eps * sigma

    vt = model(t,xt,y1)
    return mse_loss(vt, ut)

def val_loop(ema_std, ..., omega, sigma, n_steps):
    # val_loop relies on all the previous functions so
    # has to be passed all the hyperparamers for all of
    # them, as well as any of its own (e.g. ema_std
    # for the Karras et al. post-hoc EMA trick)
    ...

def train_loop(..., ema_std, omega, sigma, n_steps):
    # the amount of hyperparameters we're passing through
    # is starting to get silly by this point
    ...

def main():
    # config is usually supplied by hydra, argparse etc.
    omega, sigma, n_steps, ema_std = ...
    train_loop(... ema_std, omega, sigma, n_steps)

```

This is pretty close to what you will see in codebases for modern image generation models. Don't worry too much if you don't follow it fully, I just wanted a realistic example to make my point about hyperparameter configuration.

Notice how the parameters that configure the innermost functions (e.g. `omega`, `sigma`, `n_steps`, `ema_std`) must be threaded throughout the program until they bubble up in `main`, where they will either end up hardcoded or supplied via a CLI or config file. This is not so bad in this example, but you can see how it quickly gets annoying when you have to do this 'plumbing' work for many parameters through many layers of function calls. As a shorthand, I will call this style of configuration the 'dependency injection' pattern.

To illustrate how frustrating the dependency-injection style can be, imagine you recently read [this paper](https://arxiv.org/abs/2502.07849) and wanted to implement power-law guidance. The change itself is only two lines:

```python

def guided_prediction(model, t, x, y, alpha, omega):
    """Predict with power-law classifier-free guidance."""
    cond = model(t,x,y)
    uncond = model(t,x,None)
    diff = cond - uncond
    scale = omega * l2_norm(diff) ** alpha
    return cond + scale * diff

```

However, after making this change, we now have to add `alpha` to the function signature of `generate_image`, `val_loop` and `train_loop`. This is tedious and shouldn't be this hard, given that this is a pretty common type of change to make in research code. The galling part about this is that these functions arguably have no need to know about alpha. All they do is pass it through to `guided_prediction` where it is needed.

## Config God-objects

The plumbing work that comes with the dependency injection pattern pushes many machine learning researchers turn to a pattern that I personally hate -- the config 'God-object'. For example:

```python

def guided_prediction(model, t, x, y, config):
    """Predict with classifier-free guidance."""
    omega = config["omega"]
    cond = model(t,x,y)
    uncond = model(t,x,None)
    return cond + omega * (cond - uncond)

def generate_image(model, x0, y1, config):
    """Generate using the Euler probability flow ODE solver."""
    n_steps = config["n_steps"]
    x1 = x0
    dt = 1 / n_steps
    t = 0
    for _ in range(n_steps):
        t += dt
        x1 += guided_prediction(model,t,x1,y1,config)
    return x1

def compute_loss(model, x0, x1, y1, config):
    """Flow matching (linear interpolant) loss."""
    sigma = config["sigma"]
    t, eps = ... # sample t from U[0,1], eps from N(0,1)
    mu = t * x1 + (1-t) * x0
    ut = x1 - x0
    xt = mu + eps * sigma

    vt = model(t,xt,y1)
    return mse_loss(vt, ut)

... # val_loop, train_loop, main continue in the same way

```

This pattern reduces the plumbing work by allowing us to get away with only passing around one config object (usually a dictionary containing the superset of all configuration needed by every part of the program). Of course, this is terrible. While it may be convenient for the person writing the code, as a reader, it essentially makes all function signatures useless. We now can't know what hyperparamers a function actually depends on unless we read the entire function body.

For instance, in the example above, how are we supposed to know that the `generate_image` function needs `config` to have `n_steps` and `omega` keys, but that it will ignore the `sigma` key? We're also at risk of name clashes if two functions accidentally use the same name to configure themselves. I see the God-object pattern as essentially a global variable in disguise.

At this point, a common suggestion is to refactor the logic into a class where we set the hyperparameters in the constructor. At first glance, this seems like an improvement:

```python
class DiffusionTrainer:
    def __init__(self, omega, sigma, n_steps, ema_std):
        self.omega = omega
        self.sigma = sigma
        self.n_steps = n_steps
        self.ema_std = ema_std
        ...

    def guided_prediction(self, model, t, x, y):
        ...

    def generate_image(self, model, x0, y1):
        ...

    ... # other methods like compute_loss, val_loop etc.
```

However, while this gives us attribute access (e.g., `self.omega`) instead of dictionary lookups, it doesn't solve the core problem. Look at the signature for `generate_image`. Just as with the dictionary-based God-object, it is impossible to know from the signature that this method secretly depends on `self.n_steps` and `self.omega`. To understand the function's true dependencies, we are still forced to read its entire body. The class-based refactor only provides a thin veneer of encapsulation -- all we've done is replace the `config` dictionary with `self` as the God-object.

While my disdain for the God-object style is hard to conceal, I must admit, it _does_ do a decent job at solving the plumbing problem. I think the popularity of this pattern in machine learning research code comes down to the fact that, as well as optimising our code for performance and readability, we are also optimising it for _hackability_. Since we often want to make aggressive changes to deeply nested parts of the code, it makes sense that researchers often opt for the flexibility and power of the God-object pattern over the readability of the dependency injection pattern. This raises an interesting question: can we design a third paradigm that gives us the advantages of both patterns, without the drawbacks?

## Decorator-based configuration

One of the interesting things about config in research code is that most of the parameters are _hyperparameters_. That is to say, while they may change across runs, within each experiment, they are fixed. If this is true (i.e. we are willing to assume that configuration parameters need not change throughout the lifetime of the program) then a new possiblity opens up. What if, after resolving the configuration at the start of the program, we simply replace all configurable functions with partial versions of themselves where their hyperparamers are fixed?

Our third paradigm does exactly this by using python decorators. It's best explained through an example:

```python

from ornamentalist import configure, setup_config, Configurable

@configure()
def guided_prediction(model, t, x, y, omega = Configurable):
    """Predict with classifier-free guidance."""
    cond = model(t,x,y)
    uncond = model(t,x,None)
    return cond + omega * (cond - uncond)

@configure()
def generate_image(model, x0, y1, n_steps = Configurable):
    """Generate using the Euler probability flow ODE solver."""
    x1 = x0
    dt = 1 / n_steps
    t = 0
    for _ in range(n_steps):
        t += dt
        # no need to pass omega to guided_prediction here!
        x1 += guided_prediction(model,t,x1,y1)
    return x1

@configure()
def compute_loss(model, x0, x1, y1, sigma = Configurable):
    """Flow matching (linear interpolant) loss."""
    t, eps = ... # sample t from U[0,1], eps from N(0,1)
    mu = t * x1 + (1-t) * x0
    ut = x1 - x0
    xt = mu + eps * sigma

    vt = model(t,xt,y1)
    return mse_loss(vt, ut)

@configure()
def val_loop(ema_std = Configurable, ...,):
    ...

def train_loop(...):
    # notice how train_loop doesn't need to be passed
    # the hyperparameters for the preceding functions...
    # we've successfully stopped the hyperparameter
    # proliferation issue from earlier!
    ...

def main():
    # config is usually supplied by hydra, argparse etc.
    config = {
        "guided_prediction": {"omega": ...},
        "generate_image": {"n_steps": ...},
        "compute_loss": {"sigma": ...},
        "val_loop": {"ema_std": ...},
    }
    setup_config(config)
    train_loop(...)

```

The idea behind this pattern is that the function signatures still locally declare all the function's dependencies, so are still as meaningful as possible. However, we also get the benefits of not needing to plumb parameters around (in fact, there is even less plumbing than in the God-object pattern!). You can think of the `@configure` decorator as replacing the functions with `partial` versions of themselves, where the parameters are fixed by the values given at configuration-time.

In my opinion, this pattern is close to the best of both worlds. We get the readability of the dependency-injection paradigm, and the hackability of the God-object paradigm. Changing this code to implement power-law guidance requires far less work. All we have to do is add `alpha` to the config dict, with no changes required to the functions that call `guided_prediction`:

```python
@configure()
def guided_prediction( model, t, x, y,
    alpha = Configurable, omega = Configurable
):
    """Predict with power-law classifier-free guidance."""
    cond = model(t,x,y)
    uncond = model(t,x,None)
    diff = cond - uncond
    scale = omega * l2_norm(diff) ** alpha
    return cond + scale * diff

def main():
    config = {
        "guided_prediction": {"omega": ..., "alpha": ...},
        ...
    }
    setup_config(config)
    train_loop(...)
```

Importantly, this is not just a concept post! I've implemented decorator-based configuration as a library and released it on [GitHub](https://github.com/Charl-AI/ornamentalist) and [PyPI](https://pypi.org/project/ornamentalist/) under the name `ornamentalist`. If you're reading this and are interested, please try it and reach out with your thoughts :).

PS. I'm still riding high after the first review of this blog post came in...

> This [blog post] is a masterful piece of technical writing.
>
> \- Gemini (after some prompting).

I don't care if the system prompt tells it to say that. As a late-stage PhD student, I take what compliments I can get at this point.
