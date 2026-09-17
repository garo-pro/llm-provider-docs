Title: Introducing next-generation audio models in the API

URL Source: https://openai.com/index/introducing-our-next-generation-audio-models

Markdown Content:
# Introducing next-generation audio models in the API

A new suite of audio models to power voice agents, now available to developers worldwide.

![OpenAI next-generation audio models blog hero image](https://images.ctfassets.net/kftzwdyauwt9/WynqgNZ4TBki7zVIjJIBA/adedd27d3cc4f68ed1a89fb25d62eb96/Audio_Models_wallpaper_16.9.png?w=3840&q=90&fm=webp)

*Update on August 28, 2025: We announced the general availability of the Realtime API.* *Learn more here**.*

Over the past few months, we’ve invested in advancing the intelligence, capabilities, and usefulness of text-based agents—or systems that independently accomplish tasks on behalf of users—with releases like Operator, Deep Research, Computer-Using Agents, and the Responses API with built-in tools. However, in order for agents to be truly useful, people need to be able to have deeper, more intuitive interactions with agents beyond just text—using natural spoken language to communicate effectively.

Today, we’re launching new speech-to-text and text-to-speech audio models in the API—making it possible to build more powerful, customizable, and intelligent voice agents that offer real value. Our latest speech-to-text models set a new state-of-the-art benchmark, outperforming existing solutions in accuracy and reliability—especially in challenging scenarios involving accents, noisy environments, and varying speech speeds. These improvements increase transcription reliability, making the models especially well-suited for use cases like customer call centers, meeting note transcription, and more.

For the first time, developers can also instruct the text-to-speech model to speak in a specific way—for example, “talk like a sympathetic customer service agent”—unlocking a new level of customization for voice agents. This enables a wide range of tailored applications, from more empathetic and dynamic customer service voices to expressive narration for creative storytelling experiences.

We launched our first audio model in [__2022__](https://openai.com/index/whisper/) and since then, we’ve committed to improving the intelligence, accuracy, and reliability of these models. With these new audio models, developers can build more accurate and robust speech-to-text systems and expressive, characterful text-to-speech voices—all within the API. 

We’re introducing new `gpt-4o-transcribe` and `gpt-4o-mini-transcribe` models with improvements to word error rate and better language recognition and accuracy, compared to the original Whisper models.

`gpt-4o-transcribe` demonstrates improved Word Error Rate (WER) performance over existing Whisper models across multiple established benchmarks, reflecting significant progress in our speech-to-text technology. These advancements stem directly from targeted innovations in reinforcement learning and extensive midtraining with diverse, high-quality audio datasets. 

As a result, these new speech-to-text models can better capture nuances of speech, reduce misrecognitions, and increase transcription reliability, especially in challenging scenarios involving accents, noisy environments, and varying speech speeds. These models are available now in the [__speech-to-text API__(opens in a new window)](https://platform.openai.com/docs/guides/speech-to-text). 

*Word Error Rate (WER) measures the accuracy of speech-recognition models by calculating the percentage of incorrectly transcribed words compared to a reference transcript—lower WER is better and means fewer errors. Our latest speech-to-text models achieve lower WER across benchmarks, including FLEURS (Few-shot Learning Evaluation of Universal Representations of Speech)—a multilingual speech benchmark spanning over 100 languages using manually transcribed audio samples. These results demonstrate stronger transcription accuracy and more robust language coverage. As shown here, our models consistently outperform Whisper v2 and Whisper v3 across all language evaluations.*

*On FLEURS, our models deliver lower WER and strong multilingual performance. Lower WER is better and means fewer errors. As shown here, our models match or outperform other leading models across most major languages.* 

We’re also launching a new `gpt-4o-mini-tts` model with better steerability. For the first time, developers can “instruct” the model not just on what to say but *how* to say it—enabling more customized experiences for use cases ranging from customer service to creative storytelling. The model is available in the [__text-to-speech API__(opens in a new window)](https://platform.openai.com/docs/guides/text-to-speech). Note that these text-to-speech models are limited to artificial, preset voices, which we monitor to ensure they consistently match synthetic presets.

Our new audio models build upon the GPT‑4o and GPT‑4o‑mini architectures and are extensively pretrained on specialized audio-centric datasets, which have been critical in optimizing model performance. This targeted approach provides deeper insight into speech nuances and enables exceptional performance across audio-related tasks.

We’ve enhanced our distillation techniques, enabling knowledge transfer from our largest audio models to smaller, more efficient models. Leveraging advanced self-play methodologies, our distillation datasets effectively capture realistic conversational dynamics, replicating genuine user-assistant interactions. This helps our smaller models deliver excellent conversational quality and responsiveness.

For our speech-to-text models, we’ve integrated a reinforcement learning (RL)-heavy paradigm, pushing transcription accuracy to state-of-the-art levels. This methodology dramatically improves precision and reduces hallucination, making our speech-to-text solutions exceptionally competitive in complex speech recognition scenarios.

These developments represent progress in the field of audio modeling, combining innovative methodologies with practical enhancements for enhanced performance in speech applications.

These new audio models are available to all developers now – more on building with audio [__here__(opens in a new window)](https://platform.openai.com/docs/guides/audio). For developers already building conversational experiences with text-based models, adding our speech-to-text and text-to-speech models is the simplest way to build a voice agent. We’re releasing an integration with the [__Agents SDK__(opens in a new window)](https://openai.github.io/openai-agents-python/voice/quickstart/) that simplifies this development process. For developers looking to build low-latency speech-to-speech experiences, we recommend building with our speech-to-speech models in the Realtime API.

Looking ahead, we plan to continue to invest in improving the intelligence and accuracy of our audio models and exploring ways to allow developers to bring their own custom voices to build even more personalized experiences in ways that align with our safety standards. In addition, we’re [__continuing__](https://openai.com/index/navigating-the-challenges-and-opportunities-of-synthetic-voices/) to engage in conversations with policymakers, researchers, developers, and creatives around the challenges and opportunities synthetic voices can present. We’re excited to see the innovative and creative applications developers will build using these enhanced audio capabilities. We’ll also invest in other modalities—including video—to enable developers to build multimodal agentic experiences.

## Authors

## Research leads

Christina Kim, Junhua Mao, Yi Shen, Yu Zhang

## Contributors

**Research**

Alex Paino, Bowen Cheng, Chengxu Zhuang, Chris Koch, Damian Mrowca, Erik Ritter, Jacob Menick, James Betker, Ji Lin, Jamie Kiros, Jiahui Yu, Liang Zhou, Liyu Chen, Kevin Lu, Madeline Boyd, Michael Lampe, Mike Heaton, Nanxin Chen, Nitish Keskar, Saachi Jain, Sam Toizer, Somay Jain, Tao Xu, Tomer Kaplan, Wei Han, Xiangning Chen, Ye Jia

**Engineering**

Alina Wu, Andres Garcia Garcia, Arshi Bhatnagar, Avital Oliver, Brendan Quinn, Christina Huang, David Fang, Dragos Oprica, Dominik Kundel, Edede Oiwoh, Iaroslav Tverdokhlib, Jiacheng Feng, Jay Chen, Jenia Varavva, Jordan Sitkin, Joseph Florencio, Lien Mamitsuka, Mada Aflak, Manoli Liodakis, Mark Hudnall, Noah MacCallum, Ola Okelola, Peter Bakkum, Rohan Mehta, Romain Huet, Wanning Jiang, Wayne Chang, Yilei Qian

**Product**

Anubha Srivastava, Jackie Shannon, Jeff Harris, Reah Miyara, Xiaolin Hao

## Leadership sponsors

Aidan Clark, Andrew Gibiansky, David Sasaki, Kevin Weil, Liam Fedus, Mark Chen, Nick Ryder, Nick Turley, Olivier Godement, Prafulla Dhariwal, Shengjia Zhao, Shuchao Bi, Sherwin Wu, Sulman Choudhry
