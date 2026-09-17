Title: OpenAI Microscope

URL Source: https://openai.com/index/microscope

Markdown Content:
![Microscope](https://images.ctfassets.net/kftzwdyauwt9/a22e34f3-7274-4e42-f3e2a7086b3a/44c449c83ac1c4d927b7a1088833ad78/image_129.png?w=3840&q=90&fm=webp)

Illustration: Ben Barry

We’re introducing [OpenAI Microscope(opens in a new window)](https://microscope.openai.com/), a collection of visualizations of every significant layer and neuron of eight vision “model organisms” which are often studied in interpretability. Microscope makes it easier to analyze the features that form inside these neural networks, and we hope it will help the research community as we move towards understanding these complicated systems.

The abilities of modern neural networks are the result of the interactions of thousands of neurons (sometimes tens of thousands or more!). In order to understand their behavior, we’d like to be able to quickly and easily investigate these neurons interactions in detail, and share those observations. This is especially true in collaborative environments. For instance, one researcher might speculate:

InceptionV1 [4c:447(opens in a new window)](https://microscope.openai.com/models/inceptionv1/mixed4c_0/447) is a car detector which is built from a wheel detector ([4b:373(opens in a new window)](https://microscope.openai.com/models/inceptionv1/mixed4b_0/373)) and a window detector ([4b:237(opens in a new window)](https://microscope.openai.com/models/inceptionv1/mixed4b_0/237)).


When someone makes a claim like this, it’s useful if others can quickly explore those neurons, evaluating the claim and discovering new things. This is the goal of the OpenAI Microscope.

![Visualizations of models as stacked layers](https://images.ctfassets.net/kftzwdyauwt9/1a544545-fc75-48de-12d36267c100/89b7f5eb277bde769f8824a5eaafe7cd/models.jpg?w=3840&q=90&fm=webp)

![Feature Vis](https://images.ctfassets.net/kftzwdyauwt9/0678e6ec-c8c9-46a2-ef0f2416f032/1538a84a9e9781315f5546a1c980c4d3/feature-vis.jpg?w=3840&q=90&fm=webp)

Microscope systematically visualizes every neuron in several commonly studied vision models, and makes all of those neurons linkable. We hope this will support the interpretability community in several ways:

1. Although these models and visualizations are already open source (we help maintain the [lucid library(opens in a new window)](https://github.com/tensorflow/lucid/) , which is used to generate all the visualizations in Microscope) visualizing neurons is tedious. Microscope changes the feedback loop of exploring neurons from minutes to seconds. This quick feedback loop has been essential for us in discovering unexpected features like high-low frequency detectors in the ongoing[circuits project(opens in a new window)](https://distill.pub/2020/circuits/zoom-in/) .
2. Making models and neurons linkable allows immediate scrutiny and further exploration of research making claims about those neurons. It also removes potential confusion about which model and neuron is being discussed (which of the five versions of InceptionV1 are we talking about again?). This is really helpful for collaboration, especially when researchers are at different institutions.
3. One of the wonderful things about interpretability as an area of ML is how accessible it is. Compared to many other areas, it requires comparatively little access to compute. But systematically visualizing neural networks can still take hundreds of GPU hours. We hope that, by sharing our visualizations, we can help keep interpretability highly accessible.

Just as biologists often focus on the study of a few “model organisms,” Microscope focuses on exploring a small number of models in detail. Our initial release includes nine frequently studied vision models, along with several visualization techniques we’ve found particularly useful in studying them. We plan to expand to other models and techniques in the coming months.

We’re excited to see how the community will use Microscope, and we encourage you to reuse these assets. In particular, we think it has a lot of potential in supporting the [Circuits collaboration(opens in a new window)](https://distill.pub/2020/circuits/zoom-in/)—a project to reverse engineer neural networks by analyzing individual neurons and their connections—or similar work.

## Main contributors

## Contributors

## Related articles

[View all](https://openai.com/news/)
