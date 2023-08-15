---
title: 'Azure OpenAI on Enterprise Data with Security'
date: 11-08-2023
permalink: /posts/2023/08/Azure_OpenAI_on_Enterprise_Data_with_Security/
tags:
  - OpenAI Security
  - Azure
  - Authenitcation & Authorisation
---

Azure OpenAI Service provides powerful large language models (LLM) which can be used in many use cases (searching, knowledge mining, summarisation, classification, and sentiment analysis) and bring massive value to businesses by looking at both structured and unstructured data. 
Organisations are beginning to explore and, in fact, opt in to Azure OpenAI to uncover largely untapped insights effectively on their unstructured data, such as Team Chat, Slack conversion, Teams Video / audio calls, Internal documents, messages, emails, and images that are stored across different ecosystems, e.g., SharePoint, collaboration tools (JIRA, Confluence), internal sites, and data stores.

As the organisation wants to monetize the information but faces security challenges to protect the sensitive data. They want these capabilities, but personas who have access to this info should be restricted based on their user profile. Ex HR functions want employees to search and chat with data about HR-related queries and policies; however, their access should be restricted based on their profile, and they should not have to access other' confidential info such as personal details, pay, and grade. Similarly, organisations have multiple business units, which require separation of access with a lot of restrictions on documents or files based on their business units. This requires appropriate security control (fine-grained access control) in order to protect the confidentiality of the data based on the persona’s profile or access level within the organisation.
Authentication & Authorisation (AA) come into play in order to provide an appropriate level of security controls.
Here we will focus on possible approaches on the Azure cloud platform to implement Authentication & Authorisation (AA) to control access. 

Below, high-level reference design provide an overview of an application.

![alt text](/images/Azure OpenAI Reference Security Architecture.jpg "Azure OpenAI Security")


Flow
======

**App login (1):** User login with credentials to access OpenAI-enabled applications that are integrated with AAD (Azure Active Directory).

**AAD (2):** User credentials are validated and authenticated, and AAD issues a time-bounded authentication token.

**Application (3):** An application (web or chat app) validates AAD tokens and forwards or processes the request to specific services (use case). Ex: Cognitive Search or similar search tools with vector stores will be used for embedding search. AI Document Intelligence [(formerly Azure Form Recognizer)](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/overview?view=doc-intel-3.1.0) or any open source tool used to crack the document to extract entities for summarisation. Similarly, the Azure Video Indexer, Translator, or Whisper API is used to extract transcriptions of the audio and video files.

Authorisation - RBAC (Role Based Access Control):  Now, enforcing fine-grained access at the application level enables control access irrespective of the underlying service or tools used. But it requires building bespoke code to manage access based on the user profile. Though it is not an ideal place, it provides some level of control.

**Stage (4):**
Here, use case specific services come into play and enforce fine-grained access based on the user profile. Azure provides RBAC (Role Based Access Control) across some of these services. In stages (a–e), any one of the services or combination of services can be used, considering the use case and its complexity, so it’s possible to employ RBAC in any one of the services.

**Cog Search (4-a):**
Cognitive search indexes files, documents, and data in order to provide search services. Cognitive search comes with filters to deny access to documents by trimming results based on user profiles. This can be done as part of the indexing with associated groups (the user should be part of the group). 
[Approach details](https://learn.microsoft.com/en-us/azure/search/search-security-trimming-for-azure-search-with-aad).

Unfortunately, the above approach at present supports only whole documents and doesn’t support part of the documents or row- or column-level records.To overcome this gap, the following alternative approaches are available:

1. Build a separate index for each of the user groups based on their profile or group, but it is not scalable if you have too many user groups.
2. Build custom RBAC mechanisms in the application as explained in Stage 3. 

**Vector Store (4-b):**
Embeddings (the converted binary value of the textual information with semantic context) that are stored in the Vector store. The search tool can be used on top of the vector store to perform semantic search to find similar meanings to the search query. Cognitive Search provides Vector Store (currently in private preview) or other stores such as Redis that offer similar capabilities. [Redis store provide ACL](https://redis.com/blog/rediscover-redis-security-with-redis-enterprise-6/) 
<https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-configure-role-based-access-control>

**AI Document Intelligence (4-c):** Azure AI document intelligence handles the document cracking by extracting the entities and content from the documents, which can be used for summarisation or classification. Unfortunately, at present, Azure AI Document Intelligence doesn’t support RBAC. However, the RBAC approach can be implemented with any combination of services that support RBAC capability (ex. AI Document Intelligence + Redis Store + Cognitive Search).

**Azure Functions (4-d):**
If the use case demands to implement Azure functions calling bespoke code in the serverless world, such as orchestrating events, an RBAC approach is possible at the Azure functions level to control access to data. <https://learn.microsoft.com/en-us/azure/architecture/serverless-quest/functions-app-security#set-up-azure-role-based-access-control-azure-rbac>

**Plugins / Connectors (4-e):** Plugins are tools designed specifically for OpenAI LL Models with safety as a core principle that help access up-to-date information, run computations, or use third-party services. Plugins can be custom code to integrate with data sources as well as OpenAI to provide seamless interaction of the data with LL Models. Plugins can be embed RBAC at the code level to control the access of the data sources.

Similarly, connectors to integrate data sources to retrieve data for OpenAI API calls. So one approach is to consider the access controls at the connector level. Another approach would be to have access control at the data source level, i.e., DB, files, or events, based on the user's privileges. But the downside of this approach is account management overhead, as each individual user or group account should be created rather than using a service account.

**Orchestration (5):** Applications act as orchestrators of services to manage the OpenAI interaction, such as maintaining "in-context" background with grounding data as well as performing identity authentication before calling OpenAI APIs. Here, wrapper tools like LangChain or Semantic Kernel can be used to manage interaction with OpenAI.

Also, it enables applications to log or audit the prompting and associated events to track the OpenAI model interaction through API Management (be mindful of throughput implications).

**OpenAI - Authentication (6):** To secure the OpenAI API interaction, there are 2 approaches: 
1. API keys 
2. Azure Active Directory (AAD)
It is recommended to use AAD authentication using either managed identity or service principal to grant access with the Azure Services user RBAC role as [specified](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal)

AAD provides benefits over a static API key, which is exposed at the app level. OpenAI does provide API keys, but it requires compliance with organisation security standards such as key rotation and API keys allow users to have full access to operations such as model deployment, managing training data, fine tuning, and listing all available models. So controlling API key access can be done, but it still needs to be configured in RBAC to secure the access.

**OpenAI service (7):**
Application use service principal, or managed identity which will have an assigned role to access OpenAI resources. Azure OpenAI offers different levels of permission using [RBAC](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/role-based-access-control) which enables an authorization system for managing individual access to Azure OpenAI resources. Once the application (stage 6) is successfully authenticated with AAD tokens, it can interact with the OpenAI API model either via Python or API calls.


**Summary:**

* Use AAD to authenticate the user request without exposing the OpenAI API Key.
* Enforce authorisation (fine-grained access) at the data source level.
* A combination of use case specific services can also offer RBAC, but it involves complexity.
* Log at application level (API management or bespoke) to track OpenAI interaction.


At present, these approaches are available as this space is evolving rapidly, so I would expect more elegant solutions to be available very soon.
