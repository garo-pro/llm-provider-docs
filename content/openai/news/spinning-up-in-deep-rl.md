Title: Spinning Up in Deep RL

URL Source: https://openai.com/index/spinning-up-in-deep-rl

Markdown Content:
![Spinning Up In Deep RL](https://images.ctfassets.net/kftzwdyauwt9/ae9074fb-fa4f-4b4e-afd683021fbb/39c4c4dc9a654c42022d4f86248ceb52/spinning-up-in-deep-rl.jpg?w=3840&q=90&fm=webp)

Illustration: Leandro Castelao

We’re releasing Spinning Up in Deep RL, an educational resource designed to let anyone learn to become a skilled practitioner in deep reinforcement learning. Spinning Up consists of crystal-clear examples of RL code, educational exercises, documentation, and tutorials.

At OpenAI, we believe that deep learning generally—and deep reinforcement learning specifically—will play central roles in the development of powerful AI technology. While there are numerous resources available to let people quickly ramp up in deep learning, deep reinforcement learning is more challenging to break into. We’ve designed Spinning Up to help people learn to use these technologies and to develop intuitions about them.

We were inspired to build Spinning Up through our work with the OpenAI [Scholars(opens in a new window)](https://blog.openai.com/openai-scholars-2019/) and [Fellows(opens in a new window)](https://blog.openai.com/openai-fellows-interns-2019/) initiatives, where we observed that it’s possible for people with little-to-no experience in machine learning to rapidly ramp up as practitioners, if the right guidance and resources are available to them. Spinning Up in Deep RL was built with this need in mind and is integrated into the curriculum for [2019 cohorts(opens in a new window)](https://blog.openai.com/openai-scholars-2019/) of Scholars and Fellows.

We’ve also seen that being competent in RL can help people participate in interdisciplinary research areas like [AI safety(opens in a new window)](https://blog.openai.com/concrete-ai-safety-problems/), which involve a mix of reinforcement learning and other skills. We’ve had so many people ask for guidance in learning RL from scratch, that we’ve decided to formalize the informal advice we’ve been giving.

Spinning Up in Deep RL consists of the following core components:

- A short [introduction(opens in a new window)](https://spinningup.openai.com/en/latest/spinningup/rl_intro.html) to RL terminology, kinds of algorithms, and basic theory.
- An [essay(opens in a new window)](https://spinningup.openai.com/en/latest/spinningup/spinningup.html) about how to grow into an RL research role.
- A curated list of [important papers(opens in a new window)](https://spinningup.openai.com/en/latest/spinningup/keypapers.html) organized by topic.
- A well-documented [code repo(opens in a new window)](https://github.com/openai/spinningup) of short, standalone implementations of: Vanilla Policy Gradient (VPG), Trust Region Policy Optimization (TRPO), Proximal Policy Optimization (PPO), Deep Deterministic Policy Gradient (DDPG), Twin Delayed DDPG (TD3), and Soft Actor-Critic (SAC).
- And a [few exercises(opens in a new window)](https://spinningup.openai.com/en/latest/spinningup/exercises.html) to serve as warm-ups.

We have the following support plan for this project:

- **High-bandwidth software support period** : For the first three weeks following release we’ll move quickly on bug-fixes, installation issues, and resolving errors or ambiguities in the docs. We’ll work hard to streamline the user experience, in order to make it as easy as possible to self-study with Spinning Up.
- **Major review in April, 2019** : Approximately six months after release, we’ll do a serious review of the state of the package based on feedback we receive from the community, and announce any plans for future modification.
- **Public release of internal development** : If we make changes to Spinning Up in Deep RL as we work with our Scholars and Fellows, we’ll push the changes to the public repo and make them immediately available to everyone.

Spinning Up in Deep RL is part of a new education initiative at OpenAI which we’re ‘spinning up’ to ensure we fulfill one of the tenets of the [OpenAI Charter(opens in a new window)](https://blog.openai.com/openai-charter/): “seek to create a global community working together to address AGI’s global challenges”. We hope Spinning Up will allow more people to become familiar with deep reinforcement learning, and use it to help advance safe and broadly beneficial AI.

![Icons for OpenAI Education and OpenAI Spinning Up](https://images.ctfassets.net/kftzwdyauwt9/7eb1f349-a172-44e5-ab8e07acc08f/1205562445d938992c2c85a15572a708/education-spinning-up.png?w=3840&q=90&fm=webp)

**We’re going to host a workshop on Spinning Up in Deep RL at OpenAI San Francisco on February 2nd 2019**. The workshop will consist of 3 hours of lecture material and 5 hours of semi-structured hacking, project-development, and breakout sessions - all supported by members of the technical staff at OpenAI. Ideal attendees have software engineering experience and have tinkered with ML but no formal ML experience is required. If you’re interested in participating please complete our [short application here(opens in a new window)](https://docs.google.com/forms/d/e/1FAIpQLSdWbG3H3JYC2Vp-bC1yeP1SL_DSi6laLNjW1RjvEEzlA1V6rg/viewform). The application will close on December 8th 2018, and acceptances will be sent out on December 17th 2018.

If you want to help us push the limits of AI while communicating with and educating others, then consider applying to [work at OpenAI](https://openai.com/careers/).

We’re also going to work with other organizations to help us educate people using these materials. For our first partnership, we’re working with the [Center for Human-Compatible AI(opens in a new window)](https://humancompatible.ai/) (CHAI) at the University of California at Berkeley to run a workshop on deep RL in early 2019, similar to the planned Spinning Up workshop at OpenAI. We hope this will be the first of many.

![Center for Human-Compatible Artificial Intelligence (CHAI) logo](https://images.ctfassets.net/kftzwdyauwt9/7b09ae65-a629-48b6-0b30ce21c02e/a9094694689009719d706f816abff6c3/chai-logo-1.png?w=3840&q=90&fm=webp)

The best way to get a feel for how deep RL algorithms perform is to just run them. With Spinning Up, that’s as easy as:

At the end of training, you’ll get instructions on how to view data from the experiments and watch videos of your trained agent.

Spinning Up implementations are compatible with Gym environments from the [Classic Control(opens in a new window)](https://gym.openai.com/envs/#classic_control), [Box2D(opens in a new window)](https://gym.openai.com/envs/#box2d), or [MuJoCo(opens in a new window)](https://gym.openai.com/envs/#mujoco) task suites.

We’ve designed the code for Spinning Up with newcomers in mind, making it short, friendly, and as easy to learn from as possible. Our goal was to write minimal implementations to demonstrate how the theory becomes code, avoiding the layers of abstraction and obfuscation typically present in deep RL libraries. We favor clarity over modularity—code reuse between implementations is strictly limited to logging and parallelization utilities. Code is annotated so that you always know what’s going on, and is supported by background material (and pseudocode) on the corresponding readthedocs page.

## Author

## Acknowledgments

Thanks to the many people who contributed to this launch: Alex Ray, Amanda Askell, Ashley Pilipiszyn, Ben Garfinkel, Catherine Olsson, Christy Dennison, Coline Devin, Daniel Zeigler, Dylan Hadfield-Menell, Eric Sigler, Ge Yang, Greg Khan, Ian Atha, Jack Clark, Jonas Rothfuss, Larissa Schiavo, Leandro Castelao, Lilian Weng, Maddie Hall, Matthias Plappert, Miles Brundage, Peter Zokhov & Pieter Abbeel.

## Related articles

[View all](https://openai.com/news/)
