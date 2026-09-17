Title: Garcon

URL Source: https://transformer-circuits.pub/2021/garcon

Markdown Content:
You can also [watch a video covering similar content to this piece.](https://www.youtube.com/watch?v=LqvCPmbg5KI&list=PLoyGOS2WIonajhAVqKUgEMNmeq3nEeM51)

This document describes one of the core pieces of infrastructure we've built to enable interpretability research at Anthropic: a tool we call Garçon.

Among other research projects, at Anthropic we work on interpretability — trying to understand what is going on inside of language models as they work. We take a number of approaches, but we spend a lot of time on “[Circuits](https://distill.pub/2020/circuits/)”-style mechanistic interpretability, trying to dig deeply into the actual mechanics of the computation performed by a model.

This kind of work involves “poking at” models in a broad and flexible way. We want to examine individual activations inside arbitrary layers of the model; to perform experiments where we modify or ablate individual components; and otherwise to access and work with the “guts” of the model, and not just its inputs and outputs.

We also desire to scale this work up to the largest models that Anthropic works with. Modern Transformer models can be very large indeed, and large models can be so large that they cannot be efficiently run on a single GPU or even on a single computer with multiple GPUs.

For small models, interpretability work is often performed using Colab or Jupyter notebooks (or similar tools), using features like [PyTorch hooks](https://pytorch.org/docs/stable/notes/modules.html#module-hooks) in order to access intermediate activations. However, when the model scales beyond a single node, there's no obvious way to translate that workflow. Garçon is designed to solve this problem and enable interpretability work on arbitrary models that Anthropic may train. As I'll talk about later, it's turned out to have benefits beyond that original goal.

One typically uses Garçon by launching a new server. Launching a server requires only a name for the server, and a path to the model in our distributed storage; all other details such as model configuration and the number of GPUs are autodetected:

Once a server is launched, you can connect to it, either from an interactive notebook, or from scripts for automated analyses:

(We use the “r” prefix by convention to denote a “remote” model running via garçon).

Servers can also be launched directly from notebooks, using a similar interface:

Servers launched in this way are automatically named based on the model path, which means that re-running that cell in a notebook can detect that the server is already running and reconnect to the existing server. In addition, they are configured to automatically shut down after 1h of inactivity, so researchers don't need to worry as much about cleaning up after experiments.

Once you have a Garçon client connected to a model, the most basic operation available is to run a forward pass, which by default returns logits for every token:

However, the real power of Garçon is in its “hooks” interface, which allows accessing and modifying internal state during the forward pass. Garçon models expose a set of “probe points” (the precise set may vary with model architecture and size), each denoting a named point inside the model where a tensor may be accessed or modified. You can attach a “probe” to any of these points, and then run forward passes (or backwards passes, taking gradients wrt a user-provided loss function) with those probes attached.

The basic interface to probes is that you can provide a “probe function”. Probe functions accept two arguments: a “save context” which can be used to save activations or data for later, and the tensor represented at this particular point in the model. Probe functions can return an updated tensor, which will replace the probed tensor in the computation, or can return None to use the original value (this convention is borrowed from PyTorch hooks).

Thus, two of the simplest patterns we use would be reading out an activation:

def save_tensor(save_ctx, tensor):

save_ctx.tensor = tensor

point_name = 'layers.4.mlp.post_activation'

rmodel.forward(

tokens,

probes=[rmodel.probe_points[point_name].probe(save_tensor)]

)

Or to ablate a given component by setting it to zero (in practice, we might also set it to an appropriate mean activation or do other experiments):

def ablate_neuron(layer, mlp_idx):

def hook(_, tensor):

tensor[:, :, unit] = 0

return tensor

point_name = f'layers.{layer}.mlp.pre_activation'

return rmodel.probe_points[point_name].probe(hook)

rmodel.forward(tokens, probes=[ablate_neuron(4, 123))

You can see from this example that we are able to create new hook functions on the fly and send them to the server. We use the [cloudpickle](https://github.com/cloudpipe/cloudpickle) module to enable sending arbitrary Python code objects to the server.

The save_ctx objects are all saved by the Garçon server (per connection to the server, so multiple users may share a server), and readings can be read out using rmodel.recordings() which returns a dictionary indexed by probe point name. Internally, Garçon takes care of the fact that different probe points may live on different nodes, and does appropriate scatter/gather operations to send input and data around transparently for the user.

The Garçon “probe” interface is flexible but somewhat unwieldy; it introduces state into the server, and it requires a bit of ceremony to access activations — you need to create a probe, run the forward pass, and then separately retrieve the activations. It's natural to ask why we opted for this approach instead of something simpler that (e.g.) let you access arbitrary activations post-facto.

The probes interface was designed to be a “primitive” interface that supported all of our use cases with reasonable efficiency. Specifically, we had the following goals in mind which motivated this design:

One could imagine just capturing all intermediate values, to expose them all to later examination. However, this would be an enormous amount of data for large models, and so it is important to only save activations that are requested.

Sometimes we are not interested in a full tensor, but only in some projection of it — perhaps we only want values for one token position, or perhaps we want some summary statistic such as L2 norm. By enabling arbitrary computation on the server, we can compute this summary server-side. This has two benefits:

We sometimes want to send many, many, input examples through a model, and compute aggregates over activations for all of them. As one example, we collect “dataset examples” for MLP neurons in our models; we want to run many (potentially millions) of examples from the training corpus through the model, and compute which tokens cause the maximal activation for each neuron.

By using the persistent save_ctx object, we can write a hook which keeps this “top 10 activations” in a small data structure local to each MLP layer, and updates it incrementally while we run examples through the model as fast as possible. If we had to return full activations to the user for each example, this sort of computation would be orders of magnitude slower.

In practice, any given experiment may not require the full “probe” feature set. We additionally have a wrapper called easy_garcon which enables a simpler but less powerful interface for expressively analyzing the activations of a single context as it passes through the model. For many experiments this interface is sufficient.

We envision building additional abstraction layers on top of the core probe “assembly language” interface as we learn more about our needs and common patterns.

Garçon servers also expose access to the model’s parameters, using a similar naming scheme to probe points:

For weights one can also download them directly from storage at rest, but it is convenient to expose them via the same interface researchers are already using.

The implementation of Garçon is relatively straightforward, encompassing around 1,000 lines of code (but building heavily on our existing distributed and ML infrastructure).

The server works by running one process per GPU; these processes essentially run in lockstep, using our standard collective communication mechanisms to communicate. One designated process (“rank 0”) takes responsibility for listening on a socket for incoming requests and handling the RPC protocol. It then broadcasts requests to every rank; every rank executes each request in unison, and then (if necessary) a gather operation is performed to send data back to rank 0.

As mentioned, we use [cloudpickle](https://github.com/cloudpipe/cloudpickle) as a serializer to send arbitrary data structures (including Python code objects) between the server and client and between ranks of the server job.

Garçon is designed for flexibility over performance, but we attempt to be reasonably efficient; we believe that [performance is a feature](https://blog.nelhage.com/post/reflections-on-performance/#performance-is-a-feature), and that there are classes of experiments that are enabled by scale and efficiency. In particular, while we use pickles to encode “regular” Python data, we make an effort to send large tensor objects more directly and efficiently. It is not uncommon to manipulate tens of gigabytes of data in an experiment, and it's important not to be too sloppy with performance at that scale.

We've found Garçon very helpful for our research. We now use it for essentially all of our interpretability work, and other teams have also adopted it for ad-hoc exploration of models. Even for models that would fit within a single notebook, we benefit from the consistency of interface that Garçon brings, as well as from performance benefits. Loading even a mid-sized model onto a GPU can be somewhat slow, and with Garçon we can keep long-running servers hosting commonly-used models, which drastically reduces the “time to first plot”[Julia](https://julialang.org/) language community, which is where I learned it. In deep learning this metric is often overlooked, since training large models takes so long anyways, but we've found it a valuable lens for thinking about interpretability tooling.

We have described here the basic design and motivation of Garçon in hopes that it may be useful to other labs interested in doing interpretability work on their models.

While this article focuses on the concrete interface and approach we implemented in Garçon, we think the most valuable contribution is less in the specific details and more in the philosophy and the ways it changes how our interpretability team relates to and interacts with our models and with the rest of the organization. In particular, we find the following key properties really valuable:

If you work at an ML lab and are curious about building something similar, drop us a note! Our implementation is very tied to the particular details of our model code and our infrastructure, but we'd love to chat and share our experiences if it would be helpful.

This article was released along with [A Mathematical Framework for Transformer Circuits](https://transformer-circuits.pub/2021/framework/index.html) to provide context on our infrastructure.
