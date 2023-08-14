---
title: 'Azure OpenAI on Enterprise Data with Security'
date: 11-08-2023
permalink: _posts/2023/08/Azure_OpenAI_on_Enterprise_Data_with_Security/
tags:
  - OpenAI Security
  - Azure
  - Authenitcation & Authorisation
---

Azure OpenAI on Enterprise Data with Security
======

Azure OpenAI Service provides powerful large language models (LLM) which can be used in many use cases (searching, knowledge mining, summarise, classification and sentiment analysis) and bring the massive value to business by looking at both structured and unstructured data. 
Organisation beginning to explore and in fact opt in Azure OpenAI to uncover largely untapped insights effectively on their unstructured data. Unstructured data including Team Chat, Slack conversion, Teams Video / audio calls, Internal documents, messages, emails and images that are stored across different eco systems SharePoint, collaboration tools (JIRA, Confluence), internal sites and stores. 

As organisation wants monetise the information but facing security challenges on their sensitive data. They want this capability within organizations but personas who access the info should be restricted based on their user profile. Ex HR functions want employees to search and chat with data about HR related queries and policies however their access should restricted and should not have to access other’s confidential info such as other employee’s personal info, pay and other confidential information. . Similarly organisation has multiple business unit which requires separation of access with lot of restriction to documents or files based on their business unit. This requires security control (fine-grained access control) in order to protect the confidentiality of the data based on the persona’s profile or level within the organisations.
Authentication & Authorisation (AA) comes in to play in order to provide appropriate level of security controls.
Here we will focus on possible approaches in Azure cloud platform to implement Authentication & Authorisation (AA) to control the access. 
Below high level reference design provide an overview for an application

![alt text](/images/Azure OpenAI Reference Security Architecture.jpg "Azure OpenAI Security")



Flow
======

**App login (1)**: User login with credential to access to OpenAI enable application that are integrated with AAD (Azure Active Directory).

**AAD (2):**  User credentials are validated and authenticated and AAD issue time bounded authentication tokens. Reference _______

**Application (3):** On user authentication is successful, use case specific services that are implemented would be used. Ex: Cognitive Search or similar search tool with vector store will be used if use case demands to have embeddings search. AI Document Intelligence or any open source tool to crack to extract entities from documents for summarisation use cases. Similarly Azure Video indexer or Translator or Whisper API used to extract transcription of the audio / video files.
Authorisation - RBAC (Role Based Access Control):  Authorisation at application level enable to control access irrespective of the service specific tools that we use. But it requires build bespoke component to manage access based on the user permission or profile. Though it is not an ideal place but it provide some level of controls. 

**Stage (4):**
Here use case specific services comes to play and enforce fine-grained access based on the user profile. Azure provided RBAC (Role Based Access Control) across some of the these services. In this stage (a-e) any one of the services or combination of services can be used considering the use cases and its complexity.

**Cog Search (4-a):**
Cognitive search index the files, documents and data in order to provide search services. Cognitive search comes with filters to deny access  to documents by trim result based on user profiles. This can be done as part of the indexing with associated groups (user should be part of group). [Approach details] (https://learn.microsoft.com/en-us/azure/search/search-security-trimming-for-azure-search-with-aad)
Unfortunately above approach at present support only documents and doesn’t support row or column level records.
To overcome this gap, following alternative approach would be possible:
a)	Build separate index for each of the use group based on the profile / group but it is not scalable if you have too many user groups.
b)	Build custom RBAC mechanisms in Application code as explained in stage (3).

**Vector Store (4-b):**
Embeddings (converted binary value of the textual info with semantic context) that are stored in Vector store. Search tool can be used on top of the vector store to perform semantic search to find similar meaning of the search query. Cognitive Search provide Vector Store (currently in private preview) or other store such as Redis offers similar capability. [Redis store provide ACL](https://redis.com/blog/rediscover-redis-security-with-redis-enterprise-6/) 
<https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-configure-role-based-access-control>

**AI Document Intelligence (4-c):** Azure AI document intelligence (Azure Form Recognizer)  handles the document cracking by extract entities / content from the documents. Unfortunately at present it doesn’t support RBAC hence custom build at Application level RBAC is only option. However RBAC approach can be implemented if any combination of services that support the capability (ex AI Document Intelligence + Redis Store + Cognitive Search)

**Azure Functions (4-d):**
If use case demands to implement Azure functions calling bespoke code in serverless world such as orchestrate series of event,  RBAC approach at Azure functions level to control the access to data https://learn.microsoft.com/en-us/azure/architecture/serverless-quest/functions-app-security#set-up-azure-role-based-access-control-azure-rbac

**Plugins / Connectors (4-e):** Plugins are tools designed specifically for OpenAI LL Models with safety as a core principle, and help access up-to-date information, run computations, or use third-party services. Plugins can be custom code to integrate with data sources and OpenAI to provide seamless interaction of the data with LL Models. Plugins can be embed RBAC at the code level to control the access of the data sources.
Similarly connectors to connect data sources to retrieve data for OpenAI API calls. So access controls should be considered at connectors level.

In the above flow, services that are specified can be integrated together to achieve use case needs. Services that are direct connect to data sources should responsible for access control or it can be control at data source level ie DB, files or events based on the users privileges. But downside of this approach would be each individual users / group should be created rather than using service account which might be management overhead if organisation is very large.

**Orchestration (5):** Application use the backend as orchestration services to manage the OpenAI interaction such as maintain “in-context” background with grounding data as well as perform identity authentication before calling OpenAI APIs. Here wrapper tools like LangChain or Semantic Kernel used to manage interaction with OpenAI. Also it enable application to log / audit the prompting and associated events to track the interaction or OpenAI model completion.

**OpenAI - Authentication (6):** To secure the OpenAI API interaction, there are 2 approach a) API Keys b) AAD. It is recommend to use AAD authentication using either managed identity or service principal and grant Azure services user RBAC role as specified here https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal

AAD provide benefits over static API key which is exposed at app level.  OpenAI does provide API Key but it requires compliance to organisation security standards such as key rotation plus API key allow users to have full access operations such as model deployment, managing training data, fine tune and listing of all available models. So controlling API key access can be done however still need to be configured RBAC to secure the access

**OpenAI service (7):**
Once application authenticated successfully with AAD which issue access tokens with the access scope of services (ex: Cognitive Service) that can be used to pass across OpenAI API call either via Python SDK.

