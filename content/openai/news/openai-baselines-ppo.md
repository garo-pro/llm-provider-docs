Title: Proximal Policy Optimization

URL Source: https://openai.com/index/openai-baselines-ppo

Markdown Content:
# Proximal Policy Optimization

![Openai Baselines Ppo](https://images.ctfassets.net/kftzwdyauwt9/b4073079-b924-447c-00367a362468/d766c62670a5c92f913c54206060ba54/image_140.png?w=3840&q=90&fm=webp)

Illustration: Ben Barry

We’re releasing a new class of reinforcement learning algorithms, Proximal Policy Optimization (PPO), which perform comparably or better than state-of-the-art approaches while being much simpler to implement and tune. PPO has become the default reinforcement learning algorithm at OpenAI because of its ease of use and good performance.

[Policy gradient methods(opens in a new window)](http://karpathy.github.io/2016/05/31/rl/) are fundamental to recent breakthroughs in using deep neural networks for control, from [video games(opens in a new window)](https://www.nature.com/nature/journal/v518/n7540/full/nature14236.html), to [3D locomotion(opens in a new window)](https://arxiv.org/abs/1506.02438), to [Go(opens in a new window)](https://www.nature.com/nature/journal/v529/n7587/full/nature16961.html). But getting good results via policy gradient methods is challenging because they are sensitive to the choice of stepsize — too small, and progress is hopelessly slow; too large and the signal is overwhelmed by the noise, or one might see catastrophic drops in performance. They also often have very poor sample efficiency, taking millions (or billions) of timesteps to learn simple tasks.

Researchers have sought to eliminate these flaws with approaches like [TRPO(opens in a new window)](https://arxiv.org/abs/1502.05477) and [ACER(opens in a new window)](https://arxiv.org/abs/1611.01224), by constraining or otherwise optimizing the size of a policy update. These methods have their own trade-offs—ACER is far more complicated than PPO, requiring the addition of code for off-policy corrections and a replay buffer, while only doing marginally better than PPO on the Atari benchmark; TRPO—though useful for continuous control tasks—isn’t easily compatible with algorithms that share parameters between a policy and value function or auxiliary losses, like those used to solve problems in Atari and other domains where the visual input is significant.

With supervised learning, we can easily implement the cost function, run gradient descent on it, and be very confident that we’ll get excellent results with relatively little hyperparameter tuning. The route to success in reinforcement learning isn’t as obvious—the algorithms have many moving parts that are hard to debug, and they require substantial effort in tuning in order to get good results. PPO strikes a balance between ease of implementation, sample complexity, and ease of tuning, trying to compute an update at each step that minimizes the cost function while ensuring the deviation from the previous policy is relatively small.

We’ve [previously(opens in a new window)](https://channel9.msdn.com/Events/Neural-Information-Processing-Systems-Conference/Neural-Information-Processing-Systems-Conference-NIPS-2016/Deep-Reinforcement-Learning-Through-Policy-Optimization) detailed a variant of PPO that uses an adaptive [KL(opens in a new window)](https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence) penalty to control the change of the policy at each iteration. The new variant uses a novel objective function not typically found in other algorithms:

- is the policy parameter
- denotes the empirical expectation over timesteps
- is the ratio of the probability under the new and old policies, respectively
- is the estimated advantage at time
- is a hyperparameter, usually 0.1 or 0.2

This objective implements a way to do a Trust Region update which is compatible with Stochastic Gradient Descent, and simplifies the algorithm by removing the KL penalty and need to make adaptive updates. In tests, this algorithm has displayed the best performance on continuous control tasks and almost matches ACER’s performance on Atari, despite being far simpler to implement.

We’ve created interactive agents based on policies trained by PPO—we can [use the keyboard(opens in a new window)](https://github.com/openai/roboschool/blob/master/agent_zoo/demo_keyboard_humanoid1.py) to set new target positions for a robot in an environment within Roboschool; though the input sequences are different from what the agent was trained on, it manages to generalize.

This release of [baselines(opens in a new window)](https://github.com/openai/baselines) includes scalable, parallel implementations of PPO and TRPO which both use MPI for data passing. Both use Python3 and TensorFlow. We’re also adding pre-trained versions of the policies used to train the above robots to the [Roboschool](https://openai.com/index/roboschool/) [agent zoo(opens in a new window)](https://github.com/openai/roboschool/tree/master/agent_zoo).

**Update**: We’re also releasing a GPU-enabled implementation of PPO, called PPO2. This runs approximately 3x faster than the current PPO baseline on Atari. In addition, we’re releasing an implementation of Actor Critic with Experience Replay (ACER), a sample-efficient policy gradient algorithm. ACER makes use of a replay buffer, enabling it to perform more than one gradient update using each piece of sampled experience, as well as a Q-Function approximate trained with the Retrace algorithm.

We’re looking for people to help build and optimize our reinforcement learning algorithm codebase. If you’re excited about RL, benchmarking, thorough experimentation, and open source, please [apply(opens in a new window)](https://jobs.lever.co/openai/5c1b2c12-2d18-42f0-836e-96af2cfca5ef), and mention that you read the baselines PPO post in your application.

## Authors

## Related articles

[View all](https://openai.com/news/)
