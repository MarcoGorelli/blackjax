# BlackJAX
![CI](https://github.com/blackjax-devs/blackjax/workflows/Run%20tests/badge.svg?branch=master)
[![codecov](https://codecov.io/gh/blackjax-devs/blackjax/branch/master/graph/badge.svg)](https://codecov.io/gh/blackjax-devs/blackjax)


## What is BlackJAX? 

BlackJAX is a library of samplers for [JAX](https://github.com/google/jax) that
works on CPU as well as GPU. 

It is *not* a probabilistic programming library. However it integrates really
well with PPLs as long as they can provide a (potentially unnormalized)
log-probability density function compatible with JAX.

## Who should use BlackJAX?

BlackJAX should appeal to those who:
- Have a logpdf and just need a sampler;
- Need more than a general-purpose sampler;
- Want to sample on GPU;
- Want to build upon robust elementary blocks for their research;
- Are building a probabilistic programming language;
- Want to learn how sampling algorithms work.

## Quickstart

### Installation

BlackJAX is written in pure Python but depends on XLA via JAX. Since the JAX
installation depends on your CUDA version BlackJAX does not list JAX as a
dependency. If you simply want to use JAX on CPU, install it with:

```python
pip install jax jaxlib
```

Follow [these instructions](https://github.com/google/jax#installation) to
install JAX with the relevant hardware acceleration support.

Then install BlackJAX

```bash
pip install blackjax
```

### Example

Let us look at a simple self-contained example sampling with NUTS:

```python
import jax
import jax.numpy as jnp
import jax.scipy.stats as stats
import numpy as np

import blackjax.nuts as nuts

observed = np.random.normal(10, 20, size=1_000)
def potential_fn(loc, scale, observed=observed):
  logpdf = stats.norm.logpdf(observed, loc, scale)
  return -jnp.sum(logpdf)

# Build the kernel
step_size = 1e-3
inverse_mass_matrix = jnp.array([1., 1.])
potential = lambda x: potential_fn(**x)
kernel = nuts.kernel(potential, step_size, inverse_mass_matrix)
kernel = jax.jit(kernel)  # try without to see the speedup

# Initialize the state
initial_position = {"loc": 1., "scale": 2.}
state = nuts.new_state(initial_position, potential)

# Iterate
rng_key = jax.random.PRNGKey(0)
for _ in range(1_000):
    _, rng_key = jax.random.split(rng_key)
    state, _ = kernel(rng_key, state)
```

See [this
notebook](https://github.com/blackjax-devs/blackjax/blob/master/notebooks/Introduction.ipynb) for more examples of how to use the library: how to write inference loops for one or several chains, how to use the Stan warmup, etc.

## Philosophy

### What is BlackJAX?

BlackJAX bridges the gap between "one liner" frameworks and modular, customizable
libraries.

Users can import the library and interact with robut, well-tested and performant
samplers with a few lines of code. These samplers are aimed at PPL developers,
or people who have a logpdf and just need a sampler that works.

But the true strength of BlackJAX lies in its internals and how they can be used
to experiment quickly on existing or new sampling schemes. This lower level
exposes the building blocks of inference algorithms: integrators, proposal,
momentum generators, etc and makes it easy to combine them to build new
algorithms. It provides an opportunity to accelerate research on sampling
algorithms by providing robust, performant and reusable code.

### Why BlackJAX?

Sampling algorithms are too often integrated into PPLs and not decoupled from
the rest of the framework, making them hard to use for people who do not need
the modeling language to build their logpdf. Their implementation is most of
the time monolithic and it is impossible to reuse parts of the algorithm to
build custom kernels. BlackJAX solves both problems.

### How does it work?

BlackJAX allows to build arbitrarily complex algorithms because it is built
around a very general pattern. Everything that takes a state and returns a state
is a transition kernel, and is implemented as:

```python
new_state, info =  kernel(rng_key, state)
```

kernels are stateless functions and all follow the same API; state and
information related to the transition are returned separately. They can thus be
easily composed and exchanged. We specialize these kernels by closure instead of
passing parameters.

## Contributions

### What contributions?

We value the following contributions:
- Bug fixes
- Documentation
- High-level sampling algorithms from any family of algorithms: random walk,
  hamiltonian monte carlo, sequential monte carlo, variational inference,
  inference compilation, etc.
- New building blocks, e.g. new metrics for HMC, integrators, etc.

### How to contribute?

1. Run `pip install -r requirements-dev.txt` to install all the dev
   dependencies.
2. Run `make lint` and `make test` before pushing on the repo; CI should pass if
   these pass locally.


## Acknowledgements

Some details of the NUTS implementation were largely inspired by
[Numpyro](https://github.com/pyro-ppl/numpyro)'s.
