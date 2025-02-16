# Adology Architecture

## Overview

Adology’s architecture is designed to systematically store, analyze, and generate insights from advertising data. The system must support highly structured AI-driven analysis while also maintaining efficient retrieval of brand, ad, and intelligence reports—all within a cost-effective, scalable database structure.

At its core, Adology is not just a passive ad-tracking tool. Instead, it is an AI-powered intelligence engine that captures:
- The raw reality of ad creatives (metadata, images, videos, and text),
- The structured interpretation of these creatives (AI-generated labels, scores, and embeddings),
- The higher-level knowledge extracted from AI insights (brand-wide trends, comparative analysis, and strategic reports).

Adology's data architecture supports multiple users within customer accounts, each managing multiple _brandspaces_, where each brandspace revolves around a primary brand, tracking competitor brands and followed brands to define the scope of intelligence gathering. A user can switch brandspaces for one customer organization at any time, but will require a unique email and separate login to support multiple organizations.

Adology is, in most respects, a conventional, Internet-accessible database and content management service built on top of conventional data stores and access methods, including AWS, EC2, S3, SQS, Lambda, Postgres, Mongo, and Python. What makes it valuable is the intelligent, contextual analysis and notifications of the advertising-based videos and imagery stored and maintained in realtime. Newly ingested Images and videos are archived in S3 and accessed directly via S3 URLs. 

The APIs used by client programs, including both UX and background programs, support a Contextual Data Model that offers a variety of convenient features and returns filtered ,  transformed and AI-enhanced slices of the database. Clients see all these REST APIs at [https://adology.ai/live/…](https://adology.ai/live/…)

The data stores (S3, Postgres, Mongo) that support the Adology Contextual Data Model are filled by Acquisition Engines, which are EC2 or dynamically launched Lambda functions that pull data from remote sources, including specifically: Facebook, SerpApi, and  customer-specific sources and databases.

Most of the data is shared by all customers such as brands and advertising data including images and video, but some of it is private to each customer including _brandspaces_

Different non-UI components are runnable as either EC2 or Lambda functions and are connected through a series of SQS queues. The SQS queues and Lambdas support parallelization of workflows for maximum concurrency.




## Central Data Spine



The primary table in the central spine is the Brand Descriptions Table. There are two types of entries:

- Brands We Are Analyzing: These brands are requested by our customers. The are fully analyzed by Adology's AI Engines.

- Brands We Are Tracking: These brands are generally specified by Adology and will be downloaded and stored in S3. They are not analyzed until  request by a customer.

When a customer requests information about a brand, all videos and images previously captured by Adology are made available, along with any newer assets, and then the collection is made available  to the Analysis AI Engines.

#### Ad Descriptions/Attributes

The natural language, open-ended text attributes that are mapped to an ad by AI processing. There are 50+ attributes for which we generate descriptions. They are all open ended.
- Used to power text summaries across the app.
- Currently used for INSIGHT text summaries and INQUIRE/chatbot.
- They are also used as inputs for trend detection.

#### Detailed/Long Descriptions

A single field we gather: a 500-word description of an ad.

**Labels:** The close-ended, finite, hardcoded labels that we apply to each ad, based on predefined taxonomies. We have two label sets: UNIVERSAL and TREND labels.
- Used to power graphs and charts across the app.
- Currently used in INSIGHT, but will also be key in modules like TRACK.
- They are also used as inputs for trend detection.

#### Embeddings

Embeddeings are numerical representations of the creative or of text.

We have text embeddings of detailed descriptions, which we use to map images to insights in INSIGHT. 
 
We have visual embeddings of the ads themselves, which we use for trend detection.

- We use text embeddings of detailed descriptions to map images to insights in INSIGHT. We do this by mapping the embedding of the INSIGHT text to the detailed description embeddings to find the ad that most closely matches the insight.
- We don’t actually analyze ads or visual embeddings here.
- We use text embeddings of the detailed description to enable the chatbot to retrieve the right information.

## Adology Intelligent Data Storage

Adology is not just a database but an intelligence engine. Storage decisions must be made with retrieval efficiency & AI processing costs in mind.

### Data Hierarchy

The Adology Data Store is organized in a hierarchy of logical data stores that are continuously updated by the Adology Service Infrastructure

Organization Account > 
Brandspace > 
Tracked Brand Intelligence (1 primary, multiple competitor and followers) > 
Channels > 
Brands > 
Ads > 
AI Results > 
Reports & Displays
 
![spine]("‎https://billdonner.com/0311/ark1.png")

#### Organization Account
Represents a customer organization or business entity using Adology.
Manages multiple brand-focussed workspaces (Brandspaces) where ad intelligence analysis is conducted.
#### Brandspaces
A user is connected to at least one Brandspace, and can switch brandspace.
Each brandspace is centered around one primary brand, containting lists of competitors & followed brands, which dictate the scope of analysis.
This level ensures customized tracking and insights for each brand strategy.
#### Brand-Level Intelligence
- Captures brand metadata, such as logos, category, website, and positioning summaries.
- Brand descriptions are dynamically updated based on AI-generated insights.
- Brands connect to multiple channels (e.g., Meta, SERP, YouTube) that serve as data sources.
#### Channel-Level Tracking
- Each brand runs ads across multiple channels, and each channel is treated as a separate data source.
- Tracks where each ad originates, ensuring proper categorization and segmentation.
#### Shared Brand Data Store
- All Brand Data is from all sources is shared amongst all Adology users in S3.
#### Shared Ads Data Store
- Ads from all sources are accumulated permenently in S3, All Ads Data is shared amongst all Adology users

#### Ad-Level Intelligence (The Building Blocks of AI-Driven Insights)

1. Ad metadata is captured first (e.g., raw text, CTA, platform, status, Image/Video URL).
1. AI extracts structured attributes, generating:
- Raw Ad Descriptions → Text-based analysis of ad messaging.
- AI Labels & Scores → Categorical attributes applied by AI.
1. Embeddings → AI-generated vector representations of ad visuals & text to power semantic search & similarity analysis.
1. Ad-Level AI Data (per individual ad):
 - Raw Descriptions → AI-generated explanations of what the ad conveys.
- Labels & Scores → AI-assigned attributes (e.g., "Product Demo," "Emotional Appeal: Inspiration").
- Embeddings → AI-generated vectorized representations of text & images (used for similarity search).


#### AI Summarization & Brand-Level Insights
Once enough ad descriptions exist, they are aggregated into brand-wide insights.AI creates high-level descriptions summarizing key brand themes, messaging patterns, and performance insights.Insights remain up-to-date as new ads are processed.

Brand-Level AI Data (aggregated from ad-level AI data):
- Theme Clusters → Groups of ads that share common storytelling techniques.
- Attribute Summaries (Features, Claims, Offers) → AI-generated analysis of how 
- Messaging Trends → AI-detected patterns in claims, benefits, and CTA effectiveness.

#### Reports & Competitive Analysis
Reports are generated by & displayed in modules like Inspire Recs & Trends, Market Intelligence/Brand Details.
AI generated Reports are pre-generated at the end of the acquire process, based on Competitor & Follower Lists, for customer-facing insights (to reduce API costs).
These reports synthesize AI-generated insights from both ad-level and brand-level data. (#’s 5 & 6 in this list)
Includes competitive benchmarking, creative trends, and strategic recommendations.
EXAMPLE: Generating Ad Recommendations in INSPIRE
Software pulls the Ad & Brand Level AI data, for the Competitor & Follower listed Brands.
Runs Ad Recommendation Prompt
Saves Ad Recommendations, displays in INSPIRE REC module
We update these when 2 of the following happen:
User comes in and adds/removes a new follower/competitor brand
We see a new ad come in from a follower/competitor brand
Logic may be slightly more advanced: When 7 days have elapsed AND new ads have come in.


Reports serve as snapshots of key insights that don’t need to be recalculated in real-time, ensuring cost efficiency.
Reports are generated based on the competitor & followed brand lists. INSPIRE RECS/TRENDS, BRAND DETAILS/MARKET INTELLIGENCE
Stored insights include:
Ad Theme Comparisons (e.g., “Nike focuses on speed, while Adidas emphasizes lifestyle.”)
Messaging Effectiveness Reports (e.g., “Top 3 CTAs in the running shoe market.”)
Trend Tracking (e.g., “Limited-time offers are increasing in Meta ads.”)
Since reports are prebuilt, they are only updated intermittently—not every time a user loads the app.



### Optimizations

#### Ad AI Data is Stored in Per-Ad Documents
- Ad descriptions, labels, and embeddings are saved per ad.
- Allows fast retrieval of ad-level insights.

#### Brand AI Data is Stored Separately & Updated Incrementally
- Brand-wide insights must be refreshed as new ads arrive—but only when necessary.
- Prevents unnecessary recomputation.

#### Reports & Dashboards are Cached
- Pre-built reports reduce OpenAI API calls.
- Dashboard analytics are stored separately for fast loading times.
- Reports driven by which Brands are on Competitor & Follower List

#### Brandspaces Dictate Intelligence Scope
- Brandspaces are stored at an organizational level
- An organizations totality of Brandspaces determine which brands & competitors get tracked.
- Ensures the app remains focused on the user’s needs.



## Operating Environment

Above All, Adology strives for  nearly continuous availability of its computing infrastructure. Every component can fail independently or together with restoral within minutes.

Adology is deeply dependent upon AWS managed services for its operation, particularly Mongo DB Atlas, Postgres RDS,SQS, AWS Load Balancer, and AWS CloudFront.



![HW Pic]("https://billdonner.com/0311/ark3.png")

### Backend 

- Amazon RDS for PostgreSQL is a managed service, so AWS handles many administrative tasks like backups, software patching, automatic scaling, and failover

- MongoDB Atlas is a fully managed cloud service. It automates backups, scaling, replication, and maintenance tasks

- S3: Data stored in S3 is automatically replicated and highly durable.

- SQS: Amazon SQS (Simple Queue Service) is a fully managed message queuing service. It is designed to decouple and scale distributed systems, microservices, and serverless applications. Adology makes heavy use of SQS queues to drive its multiple application pipelines

- EC2 Servers: Initially we are deploying two EC2 hardware instances behind the load balancer, which will automatically distribute traffic and handle failover. When both machines are available only one will function as the Adology API Server; the other is running as a warm standby and can also potentially function as a webserver for static Adology content.

- Amazon API Gateway: serves as an interface between clients  and backend services (such as AWS SQS). A client will establish a WebSocket connection with the Adology WebService and pass its connection ID thru calls to the API gateway to enqueue a message to Amazon SQS. When the Lambda function associated with the SQS completes it notifies the Adology WebService to pass the result back to the frontend client via a WebSocket

- AWS Elastic Load Balancer (ELB): sits in front of an array of EC2 instances (at this point two), the application load balancer distributes both web page traffic (static content) and API requests (handled by a Python server) efficiently across multiple instances. It automatically detects unhealthy EC2 instances and stops sending traffic to them. **As long as at least one EC2 instance is running, the system remains operational.**



### FrontEnd

The main user interface is the Adology -  a javascript application that makes API calls to the Adology API server infrastructure. Since many API calls instantiate long-running background lambda functions, WebSockets are utilized to inform the client of operations completion. 

Frontend api requests run thru the load balancer to the active API Server. A Websocket connection is established in advance of long running API calls to receive notifications of request completions. Upon such notification the frontend will Typically make another API call to gather a detailed response from the Adology data stores.

 



## The Main Acquisition Flow

The main flow for acquiring brand information is the most time-consuming and time-critical part of the system, and a key performance metric is to process 10,000 ads in 10 minutes.

![mainflow]("‎https://billdonner.com/0311/ark1.png")

To kick off processing for a brand:

#### 1 - A frontend dashboard (JavaScript/TypeScript) program makes an API call that puts a message on the Apify SQS queue  
Multiple Apify processes serve this queue to handle multiple brands in parallel. Each process uses the Apify API to transform the message containing a BrandName into a stream of messages with URLs for specific images and videos.

#### 2 - The stream of URL-based messages is placed on the Acquire SQS queue  
The Acquire queue triggers the download of one URL's content and streams it to S3 concurrently. After it's in S3, it puts another message—with the S3 URL and other data/meta-data—on the analysis queue. Many Acquire processes/Lambda functions are set to service the downloads and uploads in parallel.

#### 3 - The stream of S3 URL-based messages is placed on the Analyze SQS queue  
The Analyze SQS queue feeds a collection of analysis processes that perform all of the open AI work at whatever pace they can. They are all running potentially in parallel up to the limits of the AI itself.

#### 4 - After analysis, everything rests in the database  
The frontend dashboard will periodically check for the completion of brand analysis before displaying results to customers. A future release will use WebSockets to provide a dynamic display.

#### 5 - Front-end programs access S3 directly, and Postgres + Mongo through the API  
The dashboard, which in our scaling model might be running in hundreds of browsers simultaneously, never touches the database directly—only through the API. Programs can read the data on S3, but not change it.

---

## Other Flows

Most UX activity involves responding to a user's actions by invoking an Adology API call and awaiting results. These will be discussed in more detail in the Dashboard section.

Each of the brands in the database is periodically updated by a periodic CRON job that places a message on the Apify or SerpApi SQS queues.

---


## Appendices

### Common Themes & General Recommendations Across All Modules

- **Coding Standards & Documentation**
  - Enforce a consistent code style (PEP8) using linters (e.g., flake8, Black).
  - Add comprehensive docstrings and inline comments for clarity and maintainability.

- **Centralized Configuration & Secrets Management**
  - Externalize all hardcoded values (API keys, model names, S3 bucket names, etc.) into configuration files or environment variables.
  - Use a secure secrets management solution (e.g., AWS Secrets Manager).

- **Structured Logging & Monitoring**
  - Replace all print statements with a structured logging framework that includes contextual data (user IDs, request IDs, timestamps).
  - Integrate centralized logging and monitoring systems (e.g., ELK, CloudWatch) for better production observability.

- **Error Handling & Retry Logic**
  - Use specific exception handling (e.g., catching JSONDecodeError, KeyError) rather than broad except blocks.
  - Implement standardized retry mechanisms (e.g., using the tenacity library) for transient errors in external API and S3 interactions.

- **Separation of Concerns & Modularity**
  - Refactor code to separate business logic from I/O operations (e.g., decouple database, S3, and API integrations into dedicated service layers).
  - Consolidate duplicated logic into shared modules and utilities.

- **Concurrency & Asynchronous I/O**
  - Reevaluate thread pool sizes and consider adopting asynchronous I/O (using libraries like asyncio or aiohttp) for network-bound tasks.
  - Ensure proper thread safety when sharing resources (e.g., cycling API clients).

- **Testing & CI/CD**
  - Increase test coverage with unit tests, integration tests, and end-to-end tests.
  - Integrate static analysis, security scans, and dependency checks into the CI/CD pipeline.

- **External API Integration**
  - Add input validation and robust error handling around external API calls.
  - Use retries and fallback strategies to handle transient API failures.
### Understanding Key AWS Components

For readers who are less familiar with AWS, here's a brief overview of the key services mentioned:

- **Amazon EC2 (Elastic Compute Cloud):**  
  EC2 provides scalable computing capacity in the cloud. It allows you to run virtual servers (instances) on-demand without needing to invest in hardware. This service is ideal when you need full control over the server environment or when running long-running, stateful applications.  
  [Learn more about EC2](https://docs.aws.amazon.com/ec2/).

- **Amazon S3 (Simple Storage Service):**  
  S3 is a highly scalable object storage service designed to store and retrieve any amount of data at any time. It is commonly used for storing files, backups, and media assets like images and videos.  
  [Learn more about S3](https://docs.aws.amazon.com/s3/).

- **Amazon SQS (Simple Queue Service):**  
  SQS is a managed message queuing service that enables the decoupling of components in a distributed system. It reliably transmits messages between different parts of your application, making it especially useful for scaling and parallel processing.  
  [Learn more about SQS](https://docs.aws.amazon.com/sqs/).

---

### Best Practices for Writing Functions for Parallel Execution

- **Statelessness:**  
  Python functions do not depend on or modify shared state. If state is required, store it externally (e.g., in a database, S3) so that each function invocation is independent.

- **Idempotency:**  
  Multiple identical invocations produce the same result without causing unintended side effects. This is especially important for distributed systems and when implementing retry mechanisms.

- **Robust Error Handling:**  
  Implement comprehensive error handling and logging. This is critical in parallel execution environments, as it helps with diagnosing issues and ensuring system reliability.

- **Granularity:**  
  Aim for small, well-defined functions that perform a single unit of work. Functions that are too large or complex can become harder to scale and manage in a parallel environment.

- **Testing:**  
  Test your functions both in isolation and within the parallel execution framework. This ensures that they behave as expected when running concurrently on EC2 or as AWS Lambda functions.
---

### Amazon SQS
 Amazon SQS (Simple Queue Service) is a fully managed message queuing service. It is designed to decouple and scale distributed systems, microservices, and serverless applications.
 
 - Fully managed:  Amazon SQS handles provisioning, maintenance, and scaling of the messaging infrastructure. Adology doesn’t need to manage servers, apply patches, or configure failover.

- Scalability & Performance: SQS automatically scales to handle millions of messages per second.  No need to provision capacity in advance; it scales dynamically with demand.

- Reliability & Availability: No need to set up your own replication or clustering.


# More

# Brand Descriptions Table Specification

This section describes the design and processing steps for building a **Brand Descriptions Table**. The table summarizes each brand’s creative attributes (such as features, claims, benefits, offers, key messages, etc.) and supports downstream modules like Brand Details and Inspire. It is the key component of the Central Data Spine

##   Overview

###  Primary User Personas
- **Backend Functions:** Data processing, analytics, and integration with downstream modules.

###   Jobs to Be Done
- **Summarize Brand Attributes:** Provide a cohesive view of a brand’s features, claims, benefits, etc.
- **Support Downstream Modules:** Supply standardized data for modules such as Brand Details, Inspire, and Ad Spend Reporting.
- **Ensure Consistent Data Representation:** Harmonize attribute values across both primary and competitor brands.

##   Context and Functional Requirements
  Purpose & Approach
- **Functionality:** Similar to ad descriptions:
  - One row per unique brand item.
  - Attributes calculated to summarize themes, features, and benefits.
- **Mapping:** Each attribute is derived from source fields in the ad description:
  - **Features:** from `features`
  - **Benefits:** from `benefits`
  - **Claims:** from `claims`
  - **Offers:** from `offers`
  - **Themes:** from `Long Description`
  - **Key Messages:** from `key message`

###   Key Attributes

#### MVP Attributes
- **Features**
- **Benefits**
- **Claims**
- **Offers**
- **Key Messages**

#### V2 Attributes
- **Visual Styles**
- **Emotions**
- **Category Entry Points**

#### Information Types (Example: Claims)
- **All Unique Claims**
- **Categories of Claims**
- **Summary of Claims:** Detailed description explaining the claims and their usage.

## Data Processing Pipeline

The processing pipeline extracts raw data, harmonizes entries using GPT, categorizes them, counts occurrences, and generates summaries. It supports both initial processing and incremental updates.

###   Raw Extraction
- **Source Data:** Extract raw creative attributes from ad descriptions.
- **Staging:** Store ad-level data with timestamps and source identifiers.
- **Field Mapping:**  
  - Features → `features`
  - Benefits → `benefits`
  - Claims → `claims`
  - Offers → `offers`
  - Themes → `Long Description`
  - Key Messages → `key message`

###   Unique Attribute Extraction & Harmonization

####   Initial Run (First Time Processing)
For each attribute (using **Features** as an example):

1. **Create Unique Features List**
   - **Code:** Extract unique entries from the raw features column.
   - **GPT:** Analyze and harmonize syntax to create an exhaustive list of unique features.
   - **Storage:** Save the list of unique features in the Brand Descriptions Table.
   - **Mapping:** Create a dictionary mapping raw entries to the unified unique features.
   - **Labeling:** Apply these labels back to the original ad descriptions (new column) for counting.

2. **Create Feature Categories List**
   - **GPT:** Analyze the unique features and generate corresponding feature categories.
   - **Mapping:** Build a dictionary mapping each unique feature to a feature category.
   - **Storage:** Save category mappings in both the Brand Descriptions Table and the Ad Descriptions Table (to allow counting by feature and category).

3. **Count Ads for Each Feature and Category**
   - **Code:** Count the number of ads matching each unique feature and category.
   - **Storage:** Save counts and percentages in the Brand Descriptions Table.

4. **Generate Feature Summary Information**
   - **Selection:** For each unique feature, select a sample (e.g., 5 ads).
   - **GPT:** Generate a summary explaining how the brand leverages that feature, including ad counts and percentages.
   - **Storage:** Save the generated summary in the Brand Descriptions Table.

#### B. Incremental Updates (Subsequent Runs)
When new ads are detected:

1. **Identify New Ads & Unique Features**
   - **Code:** Extract unique feature entries from new ads.
   - **Comparison:** Compare new entries with the existing unique features list.
   - **Labeling:** Mark new entries as “New Features” and update older ones (older than 4 weeks) to “Not New.”

2. **Update Unique Features and Categories**
   - **GPT:** Re-analyze the updated list to harmonize new entries.
   - **Mapping:** Update the Brand Descriptions Table and apply labels to new ads.
   - **Category Check:** Compare the updated unique features against existing categories and create new categories if necessary.
   - **Storage:** Save updated mappings in both the Brand Descriptions Table and the Ad Descriptions Table.

3. **Re-count Ads and Update Summaries**
   - **Code:** Recalculate counts and percentages.
   - **GPT:** Generate updated summaries for any new unique features.
   - **Storage:** Update summary information in the Brand Descriptions Table.

###  Competitor Brands Processing
- **Initial Run:** Use the primary brand’s unique features and category list as a baseline.
- **Subsequent Runs:**  
  - Follow the same incremental update process using the competitor’s dedicated unique features and categories list.

###   Data Hierarchy Summary
- **Raw Extraction:** Stored at the ad level.
- **Unified Uniques:** Stored at the brand level and applied as labels to ad-level data.
- **Categories:** Stored at the brand level and used to label ad data for counting.
- **Feature Summary:** Stored only at the brand level.

> **Note:** This pipeline is repeated for all creative attributes (e.g., benefits, claims, offers) except for themes, which follow a different process.

##  GPT Feedback and Enhanced Pipeline Architecture

###   Strengths of the Current Spec
- **Comprehensive Coverage:** Clearly identifies key creative attributes and the necessary information types.
- **Hierarchical Data Flow:** Logical progression from raw extraction to unified values, categorization, and summary generation.
- **Effective GPT Integration:** Uses GPT for language harmonization, deduplication, and generating human-readable summaries.
- **Incremental Update Strategy:** Processes new ads without reprocessing the entire dataset.



#### Optimizing GPT Usage
- **Batching and Caching:**  
  - Process data in batches.
  - Cache GPT outputs to reduce repeated calls.
- **Hybrid Methods:**  
  - Use traditional NLP techniques (e.g., clustering, similarity scoring) to pre-group entries before GPT processing.

#### Handling Ambiguity
- **Confidence & Ambiguity Handling:**  
  - Define fallback processes or human review triggers for ambiguous or low-confidence mappings.

#### Incremental Updates & Version Control
- **Change Detection:**  
  - Maintain timestamps and version tags for entries.
- **Versioning:**  
  - Implement version control to track changes over time in the Brand Descriptions Table.

#### Error Handling and Quality Assurance
- **Structured Logging:**  
  - Implement robust error logging for each step (e.g., GPT call errors, JSON parsing issues).
- **Validation Metrics:**  
  - Establish metrics to assess the quality and consistency of outputs (e.g., category mappings, summary accuracy).

#### Documentation & Visualization
- **Diagrams & Schemas:**  
  - Include data flow diagrams to illustrate the journey from raw extraction to final summaries.
- **Field Mapping Documentation:**  
  - Clearly document the mapping between source fields and target attributes.

#### Scalability & Performance
- **Parallel Processing:**  
  - Optimize extraction and counting steps using parallel or distributed processing.
- **Resource Management:**  
  - Schedule resource-intensive tasks during off-peak hours to manage costs and performance.

##  Additional Considerations

###   Inspire Module & Unified Ad Tags
- **Unified Ad Tags:**  
  - Ensure ad tags are consistent across brands.
  - Consider universal tags for standardized labels across all brands, in addition to brand-specific tags.
  
- **Competitor Brand Mapping:**  
  - Define competitor brands in the context of a user’s primary brand.
  - Map competitor features/claims to the customer’s taxonomy first, then create new taxonomies for unmatched entries.

###   User-Defined Taxonomy and Data Management
- **Central vs. Account-Specific Data:**
  - Define which data is stored centrally (shared across accounts) and which is calculated per account.
  
- **Taxonomy Manager:**
  - Calculated Uniques & Categories are written into the Taxonomy Manager.
  - Users can override and lock in their taxonomy, ensuring user-defined labels are prioritized.
  - This approach supports:
    - Surfacing new tags for taxonomy management.
    - Influencing pivot options in ad spend reporting.
    - Catering to both agency pitches (using identified labels) and customer account reporting (using a mix of generated and user-defined taxonomy).

##  Summary

This specification defines a comprehensive, modular, and scalable approach for creating and maintaining a **Brand Descriptions Table**. The proposed pipeline:

- **Integrates GPT** for harmonizing language and generating narrative summaries.
- **Supports Incremental Updates** to efficiently process new ad data.
- **Provides Robust Error Handling** and quality assurance.
- **Facilitates Downstream Modules** such as Brand Details, Inspire, and Ad Spend Reporting.
- **Accommodates User-Defined Inputs** via a Taxonomy Manager for added flexibility.

By following this architecture and the outlined improvements, the system will deliver consistent, high-quality brand summaries and scale effectively as new data and attributes are processed over time.

## App Routes

These are specific routes scattered throughout the code:

```python
@app.route('/api/dashboard')
@app.route('/api/index')
@app.route('/api/brand_suggestions', methods=['GET'])
@app.route('/api/lists/create', methods=['POST'])
# @app.route('/test', methods=['POST'])
@app.route('/api/query', methods=['POST'])
@app.route("/api/tasks/<task_id>", methods=["GET"])
@app.route('/api/lists', methods=['GET'])
@app.route('/api/lists/<list_item_id>', methods=['GET'])
@app.route('/api/lists/update/<list_item_id>', methods=['PUT'])
@app.route('/api/lists/remove/<list_item_id>', methods=['DELETE'])
@app.route('/api/lists/add_brands/<list_item_id>', methods=['PATCH'])
@app.route('/api/lists/remove_brand/<list_item_id>', methods=['DELETE'])
@app.route('/api/process_brand_list', methods=['POST'])
@app.route('/api/acquire/test', methods=['POST'])
@app.route('/api/process_custom_prompt', methods=['POST'])
@app.route('/api/list_status', methods=['GET'])
@app.route('/api/get_acquisition_date', methods=['GET'])
@app.route('/api/user/brands/export_all', methods=['GET'])
@app.route('/api/user/brands/export_single', methods=['POST'])
@app.route('/api/user/brands/view_all', methods=['GET'])
@app.route('/api/user/brands/view_single', methods=['POST'])
@app.route('/api/user/brands', methods=['GET'])
@app.route('/api/user/list_or_acquire_brands', methods=['GET'])
@app.route('/api/visualize_primary_brand', methods=['GET'])
@app.route('/api/user/select_comparison_brands', methods=['POST'])
@app.route('/api/user/visualize_comparison_brands', methods=['POST'])
@app.route('/api/brand_report', methods=['GET'])
@app.route('/api/brand_report_comparison', methods=['POST'])
@app.route('/api/high_rated_images')
@app.route('/send', methods=['POST'])
@app.route('/', methods=['GET'])
```
These routes are generic - they represent a collection of endpoints and require parsing of detailed arguments by the application:
```python
app.register_blueprint(auth_bp, url_prefix='/')
app.register_blueprint(chatbot_bp, url_prefix='/')
app.register_blueprint(inspire_bp, url_prefix='/')
app.register_blueprint(raw_footage_bp, url_prefix='/')
app.register_blueprint(categories_bp, url_prefix='/')
app.register_blueprint(acquire_bp, url_prefix='/')
app.register_blueprint(enrich_bp, url_prefix='/')
app.register_blueprint(brands_bp, url_prefix='/')
```
---




## Database

The plan is to gradually migrate from Mongo to Postgres for performance reasons. Any data best served by Mongo (e.g., vector embeddings) will remain there.

To facilitate this, a set of "shell functions" that hide the Postgres and Mongo specifics will be introduced into the codebase.


#### Collections and Supported Operations

- **brand_categories**: find_one, find
- **brand_states**: find_one, update_one, delete_one
- **chatbots**: find, find_one, insert_one
- **conversation_chatbot_history**: find, insert_one, delete_one, delete_many
- **conversation_history**: find, insert_one
- **custom_reports**: count_documents, find, find_one, update_one, delete_one
- **inspire_folders**: find, find_one, insert_one, update_one, delete_one
- **inspire_folder_items**: count_documents, find, find_one, insert_one, delete_one
- **inspire_subfolders**: find, find_one, insert_one, update_one, delete_one
- **inspire_subfolder_items**: count_documents, find, find_one, insert_one, delete_one
- **inspire_trends**: find_one, update_one
- **list_auto_track**: find_one, update_one
- **master_ads_meta**: count_documents, find, find_one, insert_one
- **master_ads_metadata**: find
- **master_ads_serp**: find_one, insert_one
- **master_analysis_meta**: find_one, insert_one
- **master_analysis_serp**: find_one, insert_one
- **meta_ads_data**: find_one, insert_one
- **meta_pages_info**: find_one, insert_one
- **tasks**: find_one, insert_one, update_one, delete_many
- **users**: find_one, insert_one, update_one
- **vectors_embeddings**: find_one, insert_one, update_one

 

---
---
