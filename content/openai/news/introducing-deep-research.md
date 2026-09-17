Title: Introducing deep research

URL Source: https://openai.com/index/introducing-deep-research

Markdown Content:
**February 10, 2026 update:** *You can now connect deep research to any MCP or app and restrict web searches to trusted sites, so you can focus on authenticated, industry-standard sources. You can also now track progress in real-time and interrupt to refine with follow-up prompts or new sources. We've updated the visual experience so it's easier to start, track, and review your research from end to end.*

**July 17, 2025 update:** *Deep research can now go even deeper and broader with access to a visual browser as part of ChatGPT agent. To access these updated capabilities, simply select “agent mode” from the dropdown in the composer and enter your query directly. The original deep research functionality remains available via the “deep research” option in the tools menu.*

*April 24, 2025 update**: We’re significantly increasing how often you can use deep research—Plus, Team, Enterprise, and Edu users now get 25 queries per month, Pro users get 250, and Free users get 5. This is made possible through a new lightweight version of deep research powered by a version of o4-mini, designed to be more cost-efficient while preserving high quality. Once you reach your limit for the full version, your queries will automatically switch to the lightweight version.*

*February 25, 2025 update**: All Plus users can now use deep research.*

*February 5, 2025 update**: Deep research is now available to Pro users in the United Kingdom, Switzerland, and the European Economic Area.*

Today we’re launching deep research in ChatGPT, a new agentic capability that conducts multi-step research on the internet for complex tasks. It accomplishes in tens of minutes what would take a human many hours.

Deep research is OpenAI's next agent that can do work for you independently—you give it a prompt, and ChatGPT will find, analyze, and synthesize hundreds of online sources to create a comprehensive report at the level of a research analyst. Powered by a version of the upcoming OpenAI o3 model that’s optimized for web browsing and data analysis, it leverages reasoning to search, interpret, and analyze massive amounts of text, images, and PDFs on the internet, pivoting as needed in reaction to information it encounters.

The ability to synthesize knowledge is a prerequisite for creating new knowledge. For this reason, deep research marks a significant step toward our broader goal of developing AGI, which we have long envisioned as capable of producing novel scientific research.

Deep research is built for people who do intensive knowledge work in areas like finance, science, policy, and engineering and need thorough, precise, and reliable research. It can be equally useful for discerning shoppers looking for hyper-personalized recommendations on purchases that typically require careful research, like cars, appliances, and furniture. Every output is fully documented, with clear citations and a summary of its thinking, making it easy to reference and verify the information. It is particularly effective at finding niche, non-intuitive information that would require browsing numerous websites. Deep research frees up valuable time by allowing you to offload and expedite complex, time-intensive web research with just one query.

Deep research independently discovers, reasons about, and consolidates insights from across the web. To accomplish this, it was trained on real-world tasks requiring browser and Python tool use, using the same reinforcement learning methods behind OpenAI o1, our first reasoning model. While o1 demonstrates impressive capabilities in coding, math, and other technical domains, many real-world challenges demand extensive context and information gathering from diverse online sources. Deep research builds on these reasoning capabilities to bridge that gap, allowing it to take on the types of problems people face in work and everyday life.

In ChatGPT, select ‘deep research’ in the message composer and enter your query. Tell ChatGPT what you need—whether it’s a competitive analysis on streaming platforms or a personalized report on the best commuter bike. You can attach files or spreadsheets to add context to your question. Once it starts running, a sidebar appears with a summary of the steps taken and sources used.

Deep research may take anywhere from 5 to 30 minutes to complete its work, taking the time needed to dive deep into the web. In the meantime, you can step away or work on other tasks—you’ll get a notification once the research is complete. The final output arrives as a report within the chat – in the next few weeks, we will also be adding embedded images, data visualizations, and other analytic outputs in these reports for additional clarity and context.

Compared to deep research, GPT‑4o is ideal for real-time, multimodal conversations. For **multi-faceted, domain-specific** inquiries where depth and detail are critical, deep research’s ability to conduct extensive exploration and cite each claim is the difference between a quick summary and a well-documented, verified answer that can be usable as a work product.

Deep research was trained using end-to-end reinforcement learning on hard browsing and reasoning tasks across a range of domains. Through that training, it learned to plan and execute a multi-step trajectory to find the data it needs, backtracking and reacting to real-time information where necessary. The model is also able to browse over user uploaded files, plot and iterate on graphs using the python tool, embed both generated graphs and images from websites in its responses, and cite specific sentences or passages from its sources. As a result of this training, it reaches new highs on a number of public evaluations focused on real-world problems.

On [__Humanity’s Last Exam__(opens in a new window)](https://lastexam.ai/), a recently released evaluation that tests AI across a broad range of subjects on expert-level questions, the model powering deep research scores a new high at 26.6% accuracy. This test consists of over 3,000 multiple choice and short answer questions across more than 100 subjects from linguistics to rocket science, classics to ecology. Compared to OpenAI o1, the largest gains appeared in chemistry, humanities and social sciences, and mathematics. The model powering deep research showcased a human-like approach by effectively seeking out specialized information when necessary.

| Model | Accuracy (%) | 
|---|---|
| GPT-4o | 3.3 | 
| Grok-2 | 3.8 | 
| Claude 3.5 Sonnet | 4.3 | 
| Gemini Thinking | 6.2 | 
| OpenAI o1 | 9.1 | 
| DeepSeek-R1* | 9.4 | 
| OpenAI o3-mini (medium)* | 10.5 | 
| OpenAI o3-mini (high)* | 13.0 | 
| OpenAI deep research** | 26.6 | 

On [GAIA(opens in a new window)](https://openreview.net/forum?id=fibxvahvs3)<sup>, a public benchmark that evaluates AI on real-world questions, the model powering deep research reaches a new state of the art (SOTA), topping the external</sup> [leaderboard(opens in a new window)](https://huggingface.co/spaces/gaia-benchmark/leaderboard). Encompassing questions across three levels of difficulty, successful completion of these tasks requires abilities including reasoning, multi-modal fluency, web browsing, and tool-use proficiency.

| GAIA |  |  |  |  | 
|---|---|---|---|---|
|  | Level 1 | Level 2 | Level 3 | Avg. | 
| [Previous SOTA(opens in a new window)](https://huggingface.co/spaces/gaia-benchmark/leaderboard) | 67.92 | 67.44 | 42.31 | 63.64 | 
| Deep Research (pass@1) | 74.29 | 69.06 | 47.6 | 67.36 | 
| Deep Research (cons@64) | 78.66 | 73.21 | 58.03 | 72.57 | 

### GAIA task examples

In an internal evaluation of expert-level tasks across a range of areas, deep research was rated by domain experts to have automated multiple hours of difficult, manual investigation.

##### Pass Rate vs Max Tool Calls

##### Expert-level task examples

##### Pass Rate on Expert-Level Tasks by Estimated Economic Value

##### Pass Rate on Expert-Level Tasks by Estimated Hours

Deep research unlocks significant new capabilities, but it’s still early and has limitations. It can sometimes hallucinate facts in responses or make incorrect inferences, though at a notably lower rate than existing ChatGPT models, according to internal evaluations. It may struggle with distinguishing authoritative information from rumors, and currently shows weakness in confidence calibration, often failing to convey uncertainty accurately. At launch, there may be minor formatting errors in reports and citations, and tasks may take longer to kick off. We expect all these issues to quickly improve with more usage and time.

Deep research in ChatGPT is currently very compute intensive. The longer it takes to research a query, the more inference compute is required. We are starting with a version optimized for **Pro users** today, with up to 100 queries per month. **Plus and Team users** will get access next, followed by Enterprise. We are still working on bringing access to users in the United Kingdom, Switzerland, and the European Economic Area. 

All paid users will soon get significantly higher rate limits when we release a faster, more cost-effective version of deep research powered by a smaller model that still provides high quality results.

In the coming weeks and months, we’ll be working on the technical infrastructure, closely monitoring the current release, and conducting even more rigorous testing. This aligns with our principle of iterative deployment. If all safety checks continue to meet our release standards, we anticipate releasing deep research to Plus users in about a month.

Deep research is available today on ChatGPT web, and will be rolled out to mobile and desktop apps within the month. Currently, deep research can access the open web and any uploaded files. In the future, you’ll be able to connect to more specialized data sources—expanding its access to subscription-based or internal resources—to make its output even more robust and personalized.

Looking further ahead, we envision agentic experiences coming together in ChatGPT for asynchronous, real-world research and execution. The combination of deep research, which can perform asynchronous online investigation, and Operator, which can take real-world action, will enable ChatGPT to carry out increasingly sophisticated tasks for you.

*February 3, 2025 addendum**: We conducted rigorous safety testing, preparedness evaluations, and governance reviews on the early version of o3 that powers deep research, identifying it as* *Medium*(opens in a new window) *risk. We also ran additional safety testing to better understand incremental risks associated with deep research's ability to browse the web, and we have added new mitigations. We will continue to thoroughly test and closely monitor the current limited release. We will share our safety insights and safeguards for deep research in a system card when we widen access to Plus users.*

## Footnotes

## Authors

## Research Leads

Isa Fulford, Zhiqing Sun

## Foundational Contributors

Alex Tachard Passos, Alexandra Barr, Allison Tam, Charlotte Cole, Hyung Won Chung, Jason Wei, Jon Blackman, Scott Mayer McKinney, Valerie Qi

## Core Contributors

**Research**

Elaine Ya Le, Eric Mitchell, Eric Wallace, Hyung Won Chung, Ignasi Clavera, Leo Liu, Lorenz Kuhn, Louis Feuvrier, Max Schwarzer, Saachi Jain, Scottie Yan, Shunyu Yao, Vitchyr Pong

**Deployment**

Carpus Chang, Harry Zhao, Joseph Trasatti, Joshua Dickens, Matt Kaufer, Mike Trpcic, Minnia Feng, Neel Ajjarapu, Peter Vidani, Sean Fitzgerald

## Contributors

**Research**

Ahmed El-Kishky, AJ Ostrow, Alexander Wei, Andrei Gheorghe, Andrew Kondrich, Andrey Mishchenko, Anuj Nair, Behrooz Ghorbani, Brydon Eastman, Chak Li, Foivos Tsimpourlas, Francis Song, Giambattista Parascandolo,Gildas Chabot, Hessam Bagherinezhad, Haitang Hu, Hongyu Ren, Henry Aspegren, Hunter Lightman, Ilya Kostrikov, Ilge Akkaya, James Lennon, Jean Harb, Jonathan Ward, Kai Chen, Katy Shi, Kevin Liu, Kevin Yu, Manuka Stratta, Marvin Zhang, Mengyuan Yan, Mostafa Rohaninejad, Noam Brown, Phoebe Thacker, Raz Goan, Reah Miyara, Spencer Papay, Taylor Gordon, Wenda Zhou, Wenlei Xie, Yash Patil, Yann Dubois, Youlong Cheng, Yushi Wang, Wyatt Thompson

+ all the contributors to o3.

**Safety Systems**

Adam Kalai, Alex Beutel, Andrea Vallone, Andy Applebaum, David Robinson, Elizabeth Proehl, Evan Mays, Grace Zhao, Irina Kofman, Jason Phang, Joaquin Quinonero Candela, Joel Parish, Kevin Liu, Kristen Ying, Lama Ahmad, Leon Maksin, Leyton Ho, Meghan Shah, Michele Wang, Miles Wang, Phillip Guo, Olivia Watkins, Owen Campbell-Moore, Patrick Chao, Sam Toizer, Samuel Miserendino, Sandhini Agarwal, Tejal Patwardhan, Tina Sriskandarajah, Troy Peterson, Yaodong Yu, Yunyun Wang

**Deployment**

Adam Koppel, Adam Wells, Adele Li, Andy Applebaum, Andrey Malevich, Andrew Duberstein, Andrew Howell, Anton Tananaev, Ashley Tyra, Brandon Walkin, Bryan Ashley, Cary Bassin, Cary Hudson, Cory Decareaux, Cristina Scheau, Derek Chen, Dibya Bhattacharjee, Drea Lopez, Eric Antonow, Eric Burke, Filippo Raso, Fotis Chantzis, Freddie Sulit, Harris Cohen, Heather Whitney, Jay Dixit, Jeffrey Han, Jen Robinson, Jessica Shieh, Joel Parish, Kan Wu, Kevin Gladstone, Kshitij Wadhwa, Leo Vandriel, Leyton Ho, Liang Chen, Madeline Christian, Mamie Rheingold, Matt Jones, Michelle Fradin, Mike McClay, Mingxuan Wang, Nacho Soto, Niko Felix, Patrick Delaney, Paul McMillan, Philip Pronin, Rodrigo Riaza Perez, Samuel Miserendino, Scott Ethersmith, Steven Baldwin, Thomas Dimson, Tomo Hiratsuka, Yaming Lin, Yara Khakbaz, Yining Chen

## Leadership

Akshay Nathan, Greg Brockman, Hannah Wong, Jakub Pachocki, Jerry Tworek, Johannes Heidecke, Josh Tobin, Liam Fedus, Mark Chen, Mia Glaese, Nick Turley, Sam Altman, Wojciech Zaremba
