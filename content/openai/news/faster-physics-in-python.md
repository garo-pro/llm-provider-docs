Title: Faster physics in Python

URL Source: https://openai.com/index/faster-physics-in-python

Markdown Content:
# Faster physics in Python

We’re open-sourcing a high-performance Python library for robotic simulation using the MuJoCo engine, developed over our past year of robotics research.

![Eight 3D renders of the same robot arm in varying color assortments](https://images.ctfassets.net/kftzwdyauwt9/be57b8bb-027b-4dfb-788f261485a9/089b69fcc2d295db35ebe3435dcd15ed/image-70.webp?w=3840&q=90&fm=webp)

This library is one of our core tools for deep learning [robotics research(opens in a new window)](https://blog.openai.com/robots-that-learn/), which we’ve now released as a major version of [mujoco-py(opens in a new window)](https://github.com/openai/mujoco-py), our Python 3 bindings for MuJoCo. mujoco-py [1.50.1.0(opens in a new window)](https://github.com/openai/mujoco-py/releases) brings a number of new capabilities and significant performance boosts. New features include:

- Efficient handling of parallel simulations
- GPU-accelerated headless 3D rendering
- Direct access to MuJoCo functions and data structures
- Support for all [MuJoCo 1.50 features(opens in a new window)](https://www.mujoco.org/changelist.html) like its improved contact solver

Many methods in trajectory optimization and reinforcement learning (like [LQR(opens in a new window)](https://en.wikipedia.org/wiki/Linear%E2%80%93quadratic_regulator), [PI2(opens in a new window)](http://www.jmlr.org/papers/v11/theodorou10a.html), and [TRPO(opens in a new window)](https://arxiv.org/abs/1502.05477)) benefit from being able to run multiple simulations in parallel. mujoco-py uses data parallelism through [OpenMP(opens in a new window)](http://www.openmp.org/) and direct-access memory management through [Cython(opens in a new window)](http://cython.org/) and [NumPy(opens in a new window)](http://www.numpy.org/) to make batched simulation more efficient.

[Naive usage(opens in a new window)](https://gist.github.com/jonasschneider/d8bacaec035b99c7f303aafa4f67a0f3) of the new version’s [MjSimPool(opens in a new window)](https://github.com/openai/mujoco-py/blob/master/mujoco_py/mjsimpool.pyx) interface shows a 400% speedup over the old, and still about 180% over an optimized and restricted usage pattern using Python’s [multiprocessing package(opens in a new window)](https://docs.python.org/3/library/multiprocessing.html) to gain the same level of parallelism. The majority of the speedup comes from reduced access times to the various MuJoCo data structures. Check out `[examples/simpool.py](https://github.com/openai/mujoco-py/blob/master/examples/simpool.py)` for a tour of MjSimPool.

![Loopdiscogif2](https://images.ctfassets.net/kftzwdyauwt9/42a4f547-89ce-4240-5a7d9a0d067f/61b00638721d7274ab70cc1d16af617c/loopdiscogif2.gif?w=3840&q=90&fm=webp)

We use the [domain randomization(opens in a new window)](https://arxiv.org/abs/1703.06907) technique across many projects at OpenAI. The latest version of mujoco-py supports headless GPU rendering; this yields a speedup of ~40x compared to CPU-based rendering, letting us generate hundreds of frames per second of synthetic image data. In the above (slowed down) animation we use this to vary the textures of one of our robots, which helps it identify its body when we transfer it from the simulator to reality. Check out [examples/disco_fetch.py(opens in a new window)](https://github.com/openai/mujoco-py/blob/master/examples/disco_fetch.py) for an example of randomized texture generation.

The API exposed by mujoco-py is sufficient to enable Virtual Reality interaction without any extra C++ code. We ported MuJoCo’s [C++ VR example(opens in a new window)](https://www.mujoco.org/book/programming.html#saVive) to Python using mujoco-py. If you have an HTC Vive VR setup, you can give try it using [this example(opens in a new window)](https://github.com/openai/mujoco-py/blob/master/examples/mjvive.py) (this support is considered experimental, but we’ve been using it internally for a while).

The simplest way to get started with mujoco-py is with the [MjSim class(opens in a new window)](https://github.com/openai/mujoco-py/blob/master/mujoco_py/mjsim.pyx). It is a wrapper around the simulation model and data, and lets you to easily step the simulation and render images from camera sensors. Here’s a simple example:

For advanced users, we provide a number of lower-level interfaces for accessing the innards of the MuJoCo C structs and functions directly. Refer to the [README(opens in a new window)](https://github.com/openai/mujoco-py/blob/master/README.md) and the [full documentation(opens in a new window)](https://openai.github.io/mujoco-py/build/html/index.html) to learn more.

## Authors

## Related articles

[View all](https://openai.com/news/)
