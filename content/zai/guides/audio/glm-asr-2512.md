> ## Documentation Index
> Fetch the complete documentation index at: https://docs.z.ai/llms.txt
> Use this file to discover all available pages before exploring further.

# GLM-ASR-2512

## &#x20;  Overview

GLM-ASR-2512 is Z.AI's next-generation speech recognition model, enabling real-time conversion of speech into high-quality text. Whether for daily conversations, meeting minutes, work documents, or scenarios involving specialized terminology, it delivers precise recognition and conversion, significantly boosting input and recording efficiency. The model maintains industry-leading recognition performance across diverse scenarios and accents, achieving a Character Error Rate (CER) of just 0.0717. This delivers a fast and reliable voice input experience.

> **Input Modality**
>
> Audio / File

> **Output Modality**
>
> Text

> **Upload Restrictions**
>
> * Audio duration ≤ 30 seconds
> * File size ≤ 25 MB

## &#x20;  Usage

> **Real-time Meeting Minutes**
>
> Transcribe online meetings instantly, automatically organizing structured summaries to significantly boost efficiency.

> **Customer Service Quality Assurance & Ticket Management**
>
> High-precision transcription of support calls enhances QA efficiency and enables multi-scenario analysis.

> **Live Video Captioning**
>
> Provides real-time synchronized subtitles for news broadcasts, educational courses, or video conferences with low latency and high accuracy.

> **Office Document Input**
>
> Rapidly generate work documents, emails, and proposal drafts via voice input, dramatically accelerating content creation.

> **Multilingual Communication & Translation**
>
> Supports cross-language speech comprehension for cross-border exchanges, online meetings, and educational settings.

> **Medical Record Entry**
>
> Instantly recognizes extensive medical terminology, enabling doctors to dictate patient histories for swift electronic medical record generation.

## &#x20;  Resources

* [API Documentation](/api-reference/audio/audio-transcriptions): Learn how to call the API.

## &#x20;  Introducing GLM-ASR-2512

### 1. Product Advantages

* Precise Recognition: In the latest competitive evaluation, GLM-ASR-2512 achieved a Character Error Rate (CER) of just 0.0717, reaching internationally leading standards and matching the world's top speech recognition models.
* Efficient Custom Dictionary: The model enables users to swiftly import specialized vocabulary, project codes (e.g., AutoGLM, Zhipu AI Input Method), and uncommon names/locations through simple configuration. Add once in settings to eliminate repetitive editing hassles.
* Complex Scenario Advantages: Whether handling mixed Chinese-English expressions, command-based text, industry-specific terminology, long sentences, or colloquial speech, GLM-ASR-2512 consistently delivers high-quality transcriptions with overall performance significantly outperforming competitors.

### 2. Supported Languages

GLM-ASR-2512 excels in multilingual and dialect processing, accurately transcribing major global languages and regional speech:

* Chinese: Supports Mandarin, along with major dialects including Sichuanese, Cantonese, Min Nan, and Wu
* English: Supports multiple accents such as American and British
* Other supported languages: Dozens of globally used languages including French, German, Japanese, Korean, Spanish, Arabic, and more

## &#x20;   Quick Start

The following is a full sample code to help you onboard `GLM-ASR-2512` with ease.

**cURL**

**Basic Call**

```bash theme={null}
curl --request POST \
    --url https://api.z.ai/api/paas/v4/audio/transcriptions \
    --header 'Authorization: Bearer API_Key' \
    --header 'Content-Type: multipart/form-data' \
    --form model=glm-asr-2512 \
    --form stream=false \
    --form file=@example-file
```

**Streaming Call**

```bash theme={null}
curl --request POST \
    --url https://api.z.ai/api/paas/v4/audio/transcriptions \
    --header 'Authorization: Bearer API_Key' \
    --header 'Content-Type: multipart/form-data' \
    --form model=glm-asr-2512 \
    --form stream=true \
    --form file=@example-file
```


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.