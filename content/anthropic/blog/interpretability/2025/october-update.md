Title: Circuits Updates – October 2025

URL Source: https://transformer-circuits.pub/2025/october-update

Markdown Content:
We report a number of developing ideas on the Anthropic interpretability team, which might be of interest to researchers working actively in this space. Some of these are emerging strands of research where we expect to publish more on in the coming months. Others are minor points we wish to share, since we're unlikely to ever write a paper about them.

We'd ask you to treat these results like those of a colleague sharing some thoughts or preliminary experiments for a few minutes at a lab meeting, rather than a mature paper.

New Posts

Our [recent paper](https://transformer-circuits.pub/2025/linebreaks/index.html) explored the mechanisms that LLMs develop to perceive low-level visual properties of text, like linebreaking constraints and table formatting. We wondered whether models could also perceive higher-level semantic concepts encoded visually in text. For example, can the model recognize the eyes in an ASCII face? How about eyes rendered in SVG code?    

We found that the same feature that activates over the eyes in an ASCII face also activates for eyes across diverse text-based modalities, including SVG code and prose in various languages. This is not limited to eyes – we found a number of cross-modal features that recognize specific concepts: from small components like mouths and ears within ASCII or SVG faces, to full visual depictions like dogs and cats. These cross-modal features exist in models ranging from Haiku 3.5 to Sonnet 4.5, found in sparse autoencoders trained on a middle layer.

These features depend on the surrounding context within the visual depiction. For instance, an SVG circle element activates “eye” features only when positioned within a larger structure that activates “face” features. Moreover, steering on a subset of these features during generation can modify text-based art in ways that correspond to the feature's semantic meaning, such as turning ASCII frowns to smiles or adding wrinkles to SVG faces. This work provides insight into the internal representations that models use to process and generate text-based visual content.

We began by generating ASCII and SVG smiley faces with Claude, then examining the feature activations of Haiku 3.5. In all cases we removed all comments or descriptions, including those that might identify either the individual body parts or the overall image as a face. One feature we found represented the concept of “eyes across languages and descriptions”, activating on the corresponding shapes in a smiley face illustration in both ASCII and SVG, as well as on prose describing eyes in several languages.

These features' activations depend on the surrounding context. An @ on its own will not activate the “eye” feature unless it is preceded by lines establishing ASCII art. In SVGs, the “eye” feature will only activate if they follow a circle establishing the shape of the face. We find that the activation of this feature is sensitive to a variety of contextual clues such as the character counts of each ASCII line, the color of SVG circles, and the width and height of the parent SVG element.

We then studied a more complex SVG example using features from a more advanced Sonnet 4.5 base model. Given this SVG of a dog, we find features for a host of body parts, many of which also activate on the ASCII face we studied above. We also find several “motor neuron” features, which are distinguished by having top logit effects relating to a specific concept, e.g. a “say smile” feature activates on the tokens that are most often followed by “smile”. That same feature activates on the ASCII face, shown in the bottom right of the figure below.

Like those on the less advanced Haiku 3.5, these features also appeared quite resilient to surface level attribute changes such as color or radius. When we rearrange the 4 lines of code that define the eyes of the dog SVG, we see that only the <circle> moved to the top of the SVG shows reduced activation, likely because at that point, the model doesn’t have enough context to determine that the circle represents an eye. As long as the eye-like shape occurs after the initial definition of the illustration, in this case the first ellipse as the torso, the model starts interpreting shapes as parts of an animal drawing, an LLM version of face pareidolia (the tendency for humans to see meaningful objects/patterns where there is none, like animals in clouds, or faces in your cereal).

This pareidolia effect shows in another feature we found on the same illustration which represents “mouth and lips.” It activates on the most mouth-like elements that follow the definition of the eye.

If we move the definition of the mouth and tail paths away from the 4 circles that define the eyes, the red collar now activates the most as a mouth/lip. Unlike the tail, the activation is high throughout the entire collar definition, and even continues to the bell! Is it because the red rounded rectangle could very easily be a mouth in another illustration? Or is it just about the proximity (in code and in space)? Questions like these are left for future work.

We also wondered if we would see the same type of activations if we used a human-created SVG. Turns out – yes! With this bespoke, hand-drawn dog, we find similar features for “eyes”, “mouth”, “legs”, “head”....

As a bonus, we also inspected features for an SVG of a pelican riding a bicycle, [first popularized](https://github.com/simonw/pelican-bicycle) [by Simon Willison](https://github.com/simonw/pelican-bicycle) as a way to test a model’s artistic capabilities. We find features representing concepts including “bike”, “wheels”, “feet”, “tail”, “eyes”, and “mouth” activating over the corresponding parts of the SVG code.

We’ve shown that features can represent semantically meaningful elements of SVG and ASCII art. But in addition to perception, can features also shape generation of visual depictions? These capabilities are not necessarily equivalent. A perceptual feature that reliably activates in the presence of a “smile” may or may not contain the motor information needed to produce smile-like geometry.

To investigate this, we created a feature steering task. We gave Sonnet 4.5 base model the following prompt:

Human: Make a simple SVG with a similar style to this one. 

<svg width="100" height="100" viewBox="0 0 100 100" xmlns="[http://www.w3.org/2000/svg](http://www.w3.org/2000/svg)">

<circle cx="50" cy="50" r="45" fill="yellow" stroke="black" stroke-width="2"/>

<circle cx="35" cy="40" r="5" fill="black"/>

<circle cx="65" cy="40" r="5" fill="black"/>

<path d="M 30 60 Q 50 80 70 60" fill="none" stroke="black" stroke-width="3" stroke-linecap="round"/>

</svg>

Assistant:

This prompt provides the model with code from the simple SVG face below as a style reference, and asks it to make a similar one.

As in the prior examples, the code contains no explicit comments or labels to hint to the model what it was. Without steering, the model produces the baseline output shown below, which faithfully follows the structure of the example–a centered circular face with simple features–with only minor variations in geometry from the style reference, i.e., the generated smile sits lower on the face.

We then steered the model with different features, and found it could produce meaningful semantic variations. We found these features by looking for activations over plain text sentences related to specific concepts, like “smile”, “eye”, or “cat”. Steering negatively on a “smile” feature resulted in an SVG with a frowning face. This same feature also similarly produced a frown from the ASCII smiley we studied before.

These motor features often reveal a smooth gradient as you increase the steering strength. Halfway between the default smile and the frowning face above, the model generates a straight-mouthed “neutral face” 😐 as a transition.

Steering positively with the “say cat” feature adds ears, whiskers, and a muzzle; steering positively with specific body part features introduces or emphasizes those elements (e.g., a “say wrinkles” feature adds wrinkles to the face). The generated SVGs keep the overall style of the example, such as bold colors and circular geometry, but include new or changed elements to match the feature’s semantic context.

In summary, our findings demonstrate that many features activating for plain text descriptions of concepts also activate for, and can generate, text-based visual depictions of those concepts. We showed this through: (1) using features to recognize entities within text-based visual formats like ASCII and SVG, and (2) steering features to transform visual depictions, e.g., turning smiles into frowns and faces into unicorns.

Several questions emerge from these experiments. First, our experiments focused primarily on concrete entities. Does the model contain features that capture higher, more abstract semantics, like aesthetics or artistic styles, and does steering on these features create meaningful outputs? Second, we observed that motor neurons, which have top logit effects relating to a specific concept, proved more effective for steering than perceptual neurons, which activate on input tokens relating to that concept. Are there more specific properties that characterize features that are valuable for steering, and can we develop methods to automatically identify them? Third, as these features are cross-modal, how can we combine visual output, like SVG, with other parts of our interpretability toolkit like contrastive vectors and steering to study models in a new dimension? Finally, thus far we have not investigated how these features become activated. What computational mechanisms translate low-level text into these higher-level conceptual features, and vice versa? And to what extent does the model have an internal representation where it pictures, plans, and then draws an image?

Edit 11/3/2025: Thanks to Lee Sharkey for making us aware of additional related work on data-point initialization, [Taking features out of superposition with sparse autoencoders more quickly with informed initialization](https://www.lesswrong.com/posts/YJpMgi7HJuHwXTkjk/taking-features-out-of-superposition-with-sparse) by Peigné. Our work extends these learnings by demonstrating the efficacy of this approach in real models.

In much of our interpretability work, we train dictionary learning models to extract dictionaries of latent feature representations. Our choice of initialization conditions for training dictionaries ends up having a meaningful impact on the quality of the model we train.

Prior work has shown that sparse auto-encoder (SAE) initialization has an impact on the features that the model is able to learn ([Paulo & Belrose 2025](<https://arxiv.org/html/2501.16615v2#:~:text=Abstract,-Report%20issue%20for&text=Sparse%20autoencoders%20(SAEs)%20are%20a,truly%20used%E2%80%9D%20by%20the%20model.>)), and we’ve seen in computer vision that training dictionaries constrained to the convex hull of your data can improve results ([Fel et al., 2025](https://arxiv.org/abs/2502.12892)).

In this update, we introduce the Data Point Initialization (DPI) method for training dictionaries in Large Language Models and show it achieves significant improvements. The algorithm is simple and only requires modifying network initialization.

An SAE first encodes the data by taking inputs from the residual stream with dimension 

DPI is a method for initializing our weight matrices with a noisy version of true data points. The algorithm proceeds as follows:

The intuition behind why this is an effective way to seed dictionaries is that model activations are not isotropic, so we initialize the parameters to be in the higher density region of model activations. This might lead to an initial boost in both sparsity and reconstruction. More importantly, this works empirically.

We evaluated several algorithmic variations (sampling in tranches, copying initialization blocks) but found none that consistently outperformed the base DPI method.

We test DPI on SAEs and a type of crosscoder known as a weakly causal crosscoder (WCC) ([Lindsey et al., 2024](https://transformer-circuits.pub/2024/crosscoders/index.html)). DPI improves both sparsity and reconstruction across model sizes. For our largest SAE (524k features), we observed a ~17% reduction in L0 and a ~4% improvement in MSE. For WCCs, improvements were smaller with L0 decreasing by ~1% and MSE decreasing by ~2.3%.
