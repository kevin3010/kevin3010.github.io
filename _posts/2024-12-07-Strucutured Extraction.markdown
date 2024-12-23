---
layout: post
title:  "Structured Extraction with LLM"
categories: openai llm genai
date: 2024-12-07
tags: openai llm genai
---
  
I have been exploring the capabilities of structured generation with OpenAI models for a long time, starting from when function calling was introduced. I believe structured generation is a powerful use case for LLMs. The reason is that if we can extract information already present in data and give it a structured format, we can enable significant automation.

I also came across Andrej Karpathy's idea of an "LLM OS." While I believe this concept is still far-fetched, I feel that structured generation would be one of the crucial steps toward achieving it.

Recently, I have been exploring the capability of smaller LLMs for structured generation. As a proof of concept, I used GPT-4o-mini. While I know GPT-4o-mini is still a powerful model, the purpose of this experiment was to evaluate how similar the outputs of GPT-4o and GPT-4o-mini are when the task is not to generate text but to extract structure from data.

The hypothesis is that if the information is already present in the data, an LLM capable of understanding the language should be able to extract structure from it. With this, we can fine-tune smaller LLMs to perform data extraction tasks. My assumption is that even small models (with parameters around 1 billion or fewer) are capable of extracting structure from data effectively.
  

Here is the snippet of the code that I used to extract the structure from the data.

  
**Schema.py**

```python

class PersonalInfo(BaseModel):
    name: str = Field(..., description="Full name of the candidate.")
    email: str = Field(..., description="Email address of the candidate.")


class EducationItem(BaseModel):
    degree: str = Field(..., description="Degree obtained by the candidate.")
    institution: str = Field(..., description="Name of the institution where the degree was obtained.")
    year: str = Field(..., description="Year of graduation.")  


class ExperienceItem(BaseModel):
    job_title: str = Field(..., description="Title of the job held by the candidate.")
    company: str = Field(..., description="Name of the company where the candidate worked.")
    start_date: str = Field(..., description="Start date of the employment.")
    end_date: str = Field(..., description="End date of the employment.")
    responsibilities: List[str] = Field(..., description="A list of job responsibilities.")


class ProjectItem(BaseModel):
    name: str = Field(..., description="Name of the project.")
    project_description: List[str] = Field(..., description="A list of project description.")
    tools: List[str] = Field(..., description="A list of tools used in the project.")


class CertificationItem(BaseModel):
    name: str = Field(..., description="Name of the certification.")
    issuing_organization: str = Field(..., description="Organization that issued the certification.")
    year: str = Field(..., description="Year when the certification was obtained.")


class Resume(BaseModel):
    personal_info: PersonalInfo = Field(..., description="Contains the personal details of the candidate.")
    education: List[EducationItem] = Field(..., description="Educational qualifications of the candidate.")
    experience: List[ExperienceItem] = Field(..., description="Work experience of the candidate.")
    projects: Optional[List[ProjectItem]] = Field(None, description="Projects done by the candidate.")
    skills: List[str] = Field(..., description="Skills possessed by the candidate.")
    certifications: List[CertificationItem] = Field(..., description="Certifications held by the candidate.")
```

**Raw Resume Data**
```txt
KEVIN JIVANI
kevin30101999@gmail.com ⋄ linkedin/kevin-jivani
PROJECTS
Job Recommendation System

OpenAI, Pinecone, Phi-3, BM25, FastAPI, Python

• Developed a LinkedIn job scraper and deployed it with a CRON job on DigitalOcean to periodically scrape
and process job descriptions.
• Implemented hybrid search combining Vector and Keyword based search using Reciprocal Rank Fusion.
• Employed Pinecone for embedding storage and Voyage AI for generating accurate job and resume embeddings.
• Generated 1000 task-specific synthetic job summaries using distillation method on GPT-4o for training set.
• Utilized OpenAI’s function calling feature to generate structured output and validated JSON using Langchain

EDUCATION
Master of Applied Computer Science, Concordia University (GPA : 4.12 / 4.30) September 2022 - April 2024
• Applied Artificial Intelligence, Image Processing, Introduction to Deep Learning, Distributed System Design
Bachelor of Information Technology, Dharmsinh Desai University

August 2017 - May 2021
```

**GPT-4o-Output**
```json
{
  "personal_info": {
    "name": "Kevin Jivani",
    "email": "kevin30101999@gmail.com"
  },
  "education": [
    {
      "degree": "Master of Applied Computer Science",
      "institution": "Concordia University",
      "year": "2024"
    },
    {
      "degree": "Bachelor of Information Technology",
      "institution": "Dharmsinh Desai University",
      "year": "2021"
    }
  ],
  "projects": [
    {
      "name": "Job Recommendation System",
      "project_description": [
        "Developed a LinkedIn job scraper and deployed it with a CRON job on DigitalOcean to periodically scrape and process job descriptions.",
        "Implemented hybrid search combining Vector and Keyword based search using Reciprocal Rank Fusion.",
        "Employed Pinecone for embedding storage and Voyage AI for generating accurate job and resume embeddings.",
        "Generated 1000 task-specific synthetic job summaries using distillation method on GPT-4o for training set.",
        "Utilized OpenAI’s function calling feature to generate structured output and validated JSON using Langchain"
      ],
      "tools": [
        "OpenAI","Pinecone", "Phi-3", "BM25","FastAPI","Python"
      ]
    }
  ]
}
```

GPT-4o-mini Output
```json
{
  "personal_info": {
    "name": "KEVIN JIVANI",
    "email": "kevin30101999@gmail.com"
  },
  "education": [
    {
      "degree": "Master of Applied Computer Science",
      "institution": "Concordia University",
      "year": "September 2022 - April 2024"
    },
    {
      "degree": "Bachelor of Information Technology",
      "institution": "Dharmsinh Desai University",
      "year": "August 2017 - May 2021"
    }
  ],
  "projects": [
    {
      "name": "Job Recommendation System",
      "project_description": [
        "Developed a LinkedIn job scraper and deployed it with a CRON job on DigitalOcean to periodically scrape and process job descriptions.",
        "Implemented hybrid search combining Vector and Keyword based search using Reciprocal Rank Fusion.",
        "Employed Pinecone for embedding storage and Voyage AI for generating accurate job and resume embeddings.",
        "Generated 1000 task-specific synthetic job summaries using distillation method on GPT-4o for training set.",
        "Utilized OpenAI’s function calling feature to generate structured output and validated JSON using Langchain."
      ],
      "tools": [
        "OpenAI","Pinecone","Phi-3","BM25","FastAPI","Python"
      ]
    }
  ]
}
```

I have trimmed down the resume to improve readability, and it can be observed that the output of both models is very similar, except GPT-4o-mini struggled a bit with the years in the education section. For this experiment, I also experimented with different temperature values.

Another issue I encountered was hallucination. When I provided random text as a resume, the model would fabricate values. Setting the temperature to 0 improved the results, but even then, it would populate some fields while omitting others. I believe these problems can be addressed through fine-tuning.

The next step would be to explore how smaller open-source models like Phi-3.5 and Smollm-1.1b perform on such tasks. I would love to hear some feedback on this.


For more information on the code you can visit the [Github Repository](https://github.com/kevin3010/ResumeParser).
