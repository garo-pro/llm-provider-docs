Title: Gathering human feedback

URL Source: https://openai.com/index/gathering-human-feedback

Markdown Content:
![Gathering Human Feedback](https://images.ctfassets.net/kftzwdyauwt9/c8f4669e-ce6a-4c29-83a8c68cbddd/78af9679b59629995232a07059a6d18e/image-66.webp?w=3840&q=90&fm=webp)

RL-Teacher is an open-source implementation of our interface to train AIs via occasional human feedback rather than hand-crafted reward functions. The underlying technique was developed as a step towards safe AI systems, but also applies to reinforcement learning problems with rewards that are hard to specify.

The release contains three main components:

- A [reward predictor(opens in a new window)](https://github.com/nottombrown/rl-teacher/blob/master/rl_teacher/teach.py) that can be plugged into any agent and learns to predict the actions the agent could take that a human would approve of.
- An [example agent(opens in a new window)](https://github.com/nottombrown/rl-teacher/tree/master/agents) that learns via a function specified by a reward predictor. RL-Teacher ships with three pre-integrated algorithms, including[OpenAI Baselines PPO(opens in a new window)](https://blog.openai.com/openai-baselines-ppo/) .
- A [web-app(opens in a new window)](https://github.com/nottombrown/rl-teacher/tree/master/human-feedback-api) that humans can use to give feedback, providing the data used to train the reward predictor.

The entire system consists of less than 1,000 lines of Python code (excluding the agents). After you’ve set up your web server you can launch an experiment by running:

Humans can give feedback via a simple web interface (shown above), which can be run locally (not recommended) or on a separate machine. Full documentation is available on the project’s [GitHub repository(opens in a new window)](https://github.com/nottombrown/rl-teacher). We’re excited to see what AI researchers and engineers do with this technology—please [get in touch](mailto:jack@openai.com?Subject=RL_Teacher) with any experimental results!

## Authors

## Related articles

[View all](https://openai.com/news/)
