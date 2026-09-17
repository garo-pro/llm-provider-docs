Title: Introducing 4o Image Generation

URL Source: https://openai.com/index/introducing-4o-image-generation

Markdown Content:
# Introducing 4o Image Generation

Unlocking useful and valuable image generation with a natively multimodal model capable of precise, accurate, photorealistic outputs.

## This post introduced 4o image generation.

At OpenAI, we have long believed image generation should be a primary capability of our language models. That’s why we’ve built our most advanced image generator yet into GPT‑4o. The result—image generation that is not only beautiful, but useful.

From the first cave paintings to modern infographics, humans have used visual imagery to communicate, persuade, and analyze—not just to decorate. Today's generative models can conjure surreal, breathtaking scenes, but struggle with the workhorse imagery people use to share and create information. From logos to diagrams, images can convey precise meaning when augmented with symbols that refer to shared language and experience.

GPT‑4o image generation excels at accurately rendering text, precisely following prompts, and leveraging 4o’s inherent knowledge base and chat context—including transforming uploaded images or using them as visual inspiration. These capabilities make it easier to create exactly the image you envision, helping you communicate more effectively through visuals and advancing image generation into a practical tool with precision and power.

We trained our models on the joint distribution of online images and text, learning not just how images relate to language, but how they relate to each other. Combined with aggressive post-training, the resulting model has surprising visual fluency, capable of generating images that are useful, consistent, and context-aware.

A picture is worth a thousand words, but sometimes generating a few words in the right place can elevate the meaning of an image. 4o’s ability to blend precise symbols with imagery turns image generation into a tool for visual communication.

Because image generation is now native to GPT‑4o, you can refine images through natural conversation. GPT‑4o can build upon images and text in chat context, ensuring consistency throughout. For example, if you’re designing a video game character, the character’s appearance remains coherent across multiple iterations as you refine and experiment.

GPT‑4o’s image generation follows detailed prompts with attention to detail. While other systems struggle with ~5-8 objects, GPT‑4o can handle up to 10-20 different objects. The tighter binding of objects to their traits and relations allows for better control.

GPT‑4o can analyze and learn from user-uploaded images, seamlessly integrating their details into its context to inform image generation.

Native image generation enables 4o to link its knowledge between text and images, resulting in a model that feels smarter and more efficient.

Training on images reflecting a vast variety of image styles allows the model to create or transform images convincingly.

A candid paparazzi-style photo of Karl Marx hurriedly walking through the parking lot of the Mall of America, glancing over his shoulder with a startled expression as he tries to avoid being photographed. He’s clutching multiple glossy shopping bags filled with luxury goods. His coat flutters behind him in the wind, and one of the bags is swinging as if he’s mid-stride. Blurred background with cars and a glowing mall entrance to emphasize motion. Flash glare from the camera partially overexposes the image, giving it a chaotic, tabloid feel.

Our model isn’t perfect. We’re aware of multiple limitations at the moment which we will work to address through model improvements after the initial launch.

In line with our Model Spec, we aim to maximize creative freedom by supporting valuable use cases like game development, historical exploration, and education—while maintaining strong safety standards. At the same time, it remains as important as ever to block requests that violate those standards. Below are evaluations of additional risk areas where we’re working to enable safe, high-utility content and support broader creative expression for users.

**Provenance via C2PA and internal reversible search**

**Blocking the bad stuff**

For more on our approach, visit the image generation [__addendum to the GPT‑4o system card__](https://openai.com/index/gpt-4o-image-generation-system-card-addendum/). 

**Using reasoning to power safety**

[work, we’ve trained a reasoning LLM to work directly from human-written and interpretable safety specifications. We used this reasoning LLM during development to help us identify and address ambiguities in our policies. Together with our multimodal advancements and existing safety techniques developed for ChatGPT and Sora, this allows us to](https://openai.com/index/deliberative-alignment/)

__deliberative alignment__
[both input text and output images against our policies.](https://openai.com/index/upgrading-the-moderation-api-with-our-new-multimodal-moderation-model/)

__moderate__
4o image generation rolls out starting today to Plus, Pro, Team, and Free users as the default image generator in ChatGPT, with access coming soon to Enterprise and Edu. It’s also available to use in Sora. For those who hold a special place in their hearts for DALL·E, it can still be accessed through a dedicated DALL·E GPT.

Developers will soon be able to generate images with GPT‑4o via the API, with access rolling out in the next few weeks.

Creating and customizing images is as simple as chatting using GPT‑4o - just describe what you need, including any specifics like aspect ratio, exact colors using hex codes, or a transparent background. Because this model creates more detailed pictures, images take longer to render, often up to one minute.

credit creator: [Alex Duffy](https://every.to/@AlxAi)

credit creator: [Alex Duffy](https://every.to/@AlxAi)

credit creator: [Alex Duffy](https://every.to/@AlxAi)

## Author

## Leadership

**Gabriel Goh:** Image Generation 

**Jackie Shannon:** ChatGPT Product 

**Mengchao Zhong, Wayne Chang:** ChatGPT Engineering 

**Rohan Sahai:** Sora Product and Engineering 

**Brendan Quinn, Tomer Kaftan:** Inference 

**Prafulla Dhariwal:** Multimodal Organization

## Research

**Foundational Research**

Allan Jabri, David Medina, Gabriel Goh, Kenji Hata, Lu Liu, Prafulla Dhariwal

**Core Research**

Aditya Ramesh, Alex Nichol, Casey Chu, Cheng Lu, Dian Ang Yap, Heewoo Jun, James Betker, Jianfeng Wang, Long Ouyang, Li Jing, Wesam Manassra

**Research Contributors**

Aiden Low, Brandon McKinzie, Charlie Nash, Huiwen Chang, Ishaan Gulrajani, Jamie Kiros, Ji Lin, Kshitij Gupta, Yang Song

**Model Behavior**

Laurentia Romaniuk

**Multimodal Organization**

Andrew Gibiansky, Yang Lu

## Data

**Data Leads**

Gildas Chabot, James Park Lennon

**Data**

Arshi Bhatnagar, Dragos Oprica, Rohan Kshirsagar, Spencer Papay, Szi-chieh Yu, Wesam Manassra, Yilei Qian

**Moderators** 

Hazel Byrne, Jennifer Luckenbill, Mariano López

**Human Data Advisors**

Long Ouyang

## Scaling

**Inference Leads**

Brendan Quinn, Tomer Kaftan

**Inference**

Alyssa Huang, Jacob Menick, Nick Stathas, Ruslan Vasilev, Stanley Hsieh

## Applied

**ChatGPT Product Lead**

Jackie Shannon

**ChatGPT Engineering Leads**

Mengchao Zhong, Wayne Chang

**Product Design Lead**

Matt Chan

**Data Science**

Xiaolin Hao

**ChatGPT**

Andrew Sima, Annie Cheng, Benjamin Goh, Boyang Niu, Dian Ang Yap, Duc Tran, Edede Oiwoh, Eric Zhang, Ethan Chang, Jeffrey Dunham, Jay Chen, Kan Wu, Karen Li, Kelly Stirman, Mengyuan Xu, Michelle Qin, Ola Okelola, Pedro Aguilar, Rocky Smith, Rohit Ramchandani, Sara Culver, Sean Fitzgerald, Vlad Fomenko, Wanning Jiang, Wesam Manassra, Xiaolin Hao, Yilei Qian

## Sora

**Sora Product Leads**

Rohan Sahai, Wesam Manassra

**Sora Product and Engineering**

Boyang Niu, David Schnurr, Gilman Tolle, Joe Taylor, Joey Flynn, Mike Starr, Rajeev Nayak, Rohan Sahai, Wesam Manassra

## Safety

**Safety Lead**

Somay Jain

**Safety**

Alex Beutel, Andrea Vallone, Botao Hao, Brendan Quinn, Cameron Raymond, Chong Zhang, David Robinson, Eric Wallace, Filippo Raso, Huiwen Chang, Ian Kivlichan, Irina Kofman, Keren Gu-Lemberg, Kristen Ying, Madelaine Boyd, Meghan Shah, Michael Lampe, Owen Campbell-Moore, Rohan Sahai, Rodrigo Riaza Perez, Sam Toizer, Sandhini Agarwal, Troy Peterson

## Strategy

Adam Cohen, Adam Wells, Ally Bennett, Ashley Pantuliano, Carolina Paz, Claudia Fischer, Declan Grabb, Gaby Sacramone-Lutz, Lauren Jonas, Ryan Beiermeister, Shiao Lee, Tom Stasi, Tyce Walters, Ziad Reslan, Zoe Stoll

## Marketing & Comms

**Comms and Marketing Leads**

Minnia Feng, Natalie Summers, Taya Christianson

**Comms**

Alex Baker-Whitcomb, Ashley Tyra, Bailey Richardson, Gaby Raila, Marselus Cayton, Scott Ethersmith, Souki Mansoor

## Design & Creative

**Leads**

Kendra Rimbach, Veit Moeller

**Design**

Adam Brandon, Adam Koppel, Angela Baek, Cary Hudson, Dana Palmie, Freddie Sulit, Jeffrey Sabin Matsumoto, Leyan Lo, Matt Nichols, Thomas Degry, Vanessa Antonia Schefke, Yara Khakbaz

## Special Thanks

Aditya Ramesh, Aidan Clark, Alex Beutel, Ben Newhouse, Ben Rossen, Che Chang, Greg Brockman, Hannah Wong, Ishaan Singal, Jason Kwon, Jiacheng Feng, Jiahui Yu, Joanne Jang, Johannes Heidecke, Kevin Weil, Mark Chen, Mia Glaese, Nick Turley, Raul Puri, Reiichiro Nakano, Rui Shu, Sam Altman, Shuchao Bi, Vinnie Monaco
