Title: Thinking with images

URL Source: https://openai.com/index/thinking-with-images

Markdown Content:
[__OpenAI o3 and o4-mini__](https://openai.com/index/introducing-o3-and-o4-mini/) are the latest visual reasoning models in our o-series. For the first time, our models can think with images in their chain-of-thought—not just see them.

Similar to our earlier OpenAI o1 model, o3 and o4-mini are trained to think for longer before answering—and use a long internal chain of thought before responding to the user. o3 and o4-mini further extend this capability by thinking with images in their chain-of-thought, which is achieved by transforming user uploaded images with tools, allowing them to crop, zoom in, and rotate, in addition to other simple image processing techniques. More importantly, these capabilities come natively, without relying on separate specialized models.

ChatGPT’s enhanced visual intelligence helps you solve tougher problems by analyzing images more thoroughly, accurately, and reliably than ever before. It can seamlessly combine advanced reasoning with tools like web search and image manipulation—automatically zooming, cropping, flipping, or enhancing your images—to extract insights even from imperfect photos. For example, you can upload a photo of an economics problem set to receive step-by-step explanations, or share a screenshot of a build error to quickly get a root-cause analysis.

This approach enables a new axis for test-time compute scaling that seamlessly blends visual and textual reasoning, as reflected in their state-of-the-art performance across multimodal benchmarks, marking a significant step toward multimodal reasoning.

Thinking with images allows you to interact with ChatGPT more easily. You can ask questions by taking a photo without worrying about the positioning of objects—whether the text is upside down or there are multiple physics problems in one photo. Even if objects are not obvious at first glance, visual reasoning allows the model to zoom in to see more clearly.

*All examples were completed with OpenAI o3.*

Our latest visual reasoning models work in tandem with other tools like Python data analysis, web search, image generation to creatively and effectively solve more complex problems, delivering our first multimodal agentic experience to users.

To highlight visual reasoning improvement versus our previous multimodal models, we tested OpenAI o3 and o4-mini on a diverse set of human exams and ML benchmarks. These new visual reasoning models significantly outperform their predecessors on **all** multimodal tasks we tested.

*All models are evaluated at high ‘reasoning effort’ settings—similar to variants like ‘o4-mini-high’ in ChatGPT.*

In particular, thinking with images—without relying on browsing—leads to significant gains across all perception benchmarks we’ve evaluated. Our models set new state-of-the-art performance in STEM question-answering (MMMU, MathVista), chart reading and reasoning (CharXiv), perception primitives (VLMs are Blind), and visual search (V*). On V*, our visual reasoning approach achieves 95.7% accuracy, largely solving the benchmark.

Thinking with images currently has the following limitations:

- **Excessively long reasoning chains:** Models may perform redundant or unnecessary tool calls and image manipulation steps, resulting in overly long chains of thought.
- **Perception errors:** Models can still make basic perception mistakes. Even when tool calls correctly advance the reasoning process, visual misinterpretations may lead to incorrect final answers.
- **Reliability:** Models may attempt different visual reasoning processes among multiple tries of a problem, some of which can lead to incorrect results.

OpenAI o3 and o4-mini significantly advance state-of-the-art visual reasoning capabilities, representing an important step toward broader multimodal reasoning. These models deliver best-in-class accuracy on visual perception tasks, enabling it to solve questions that were previously out of reach.

We’re continually refining the models’ reasoning capabilities with images to be more concise, less redundant, and more reliable. We’re excited to continue our research in multimodal reasoning, and for people to explore how these improvements can enhance their everyday work.

**April 16 update:** results for o3 on Charxiv-r, Mathvista, and vlmsareblind were updated to reflect a system prompt change that wasn’t present in the original evaluation.

## Authors

## Contributors

Aditya Ramesh, Aidan Clark, Aleksandra Spyra, Alex Tachard Passos, Alexander Kirillov, Ali Kalami, Amy McDonald Sandjideh, Andrei Gheorghe, Andrew Gibiansky, Andrew Tulloch, Angela Baek, Anubha Srivastava, Avital Oliver, Behrooz Ghorbani, Ben Leimberger, Borys Minaiev, Bowen Cheng, Brandon McKinzie, Carpus Chang, Cary Hudson, Casey Chu, Charlotte Cole, Chen Shen, Dan Roberts, Dana Palmie, Daniel Kappler, David Medina, Edmund Wong, Eric Mitchell, Eric Ning, Freddie Sulit, Haiming Bao, Haitang Hu, Hongyu Ren, Hyeonwoo Noh, Jakub Pachocki, James Betker, James Qin, Jamie Kiros, Jason Ai, Jerry Tworek, Jessica Liang, Ji Lin, Jiahui Yu, Jianfeng Wang, Joseph Mo, Kenji Hata, Kevin King, Kristian Georgiev, Kshitij Gupta, Lauren Yang, Li Jing, Lin Yang, Linden Li, Mark Chen, Martin Li, Max Schwarzer, Mia Glaese, Michael Malek, Minnia Feng, Nacho Soto, Nat McAleese, Niko Felix, Peter Faiman, Prafulla Dhariwal, Rajkumar Samuel, Rapha Gontijo Lopes, Ravi Teja Mullapudi, Reiichiro Nakano, Rennie Song, Ricky Xu, Sam Altman, Sean Fitzgerald, Shengjia Zhao, Shengli Hu, Shuchao Bi, Spencer Papay, Szi-chieh Yu, Wenda Zhou, Yang Lu, Yara Khakbaz, Yunxing Dai, Zhishuai Zhang
