# Adology Architecture

## Overview

Adology’s architecture is designed to systematically store, analyze, and generate insights from advertising data. The system must support highly structured AI-driven analysis while also maintaining efficient retrieval of brand, ad, and intelligence reports—all within a cost-effective, scalable database structure.

At its core, Adology is not just a passive ad-tracking tool. Instead, it functions as an AI-powered intelligence engine that captures:

- The raw reality of ad creatives (metadata, images, videos, and text)
- The structured interpretation of these creatives (AI-generated labels, scores, and embeddings)
- The higher-level knowledge extracted from AI insights (brand-wide trends, comparative analysis, and strategic reports)

Adology's data architecture supports multiple users within customer accounts, each managing multiple *brandspaces*. Each brandspace focuses on a primary brand, tracking competitor brands and followed brands to define the scope of intelligence gathering. A user can switch brandspaces for one customer organization at any time but requires a unique email and separate login to support multiple organizations.

Adology largely operates as a conventional, Internet-accessible database and content management service built on top of well-known data stores and access methods, including AWS, EC2, S3, SQS, Lambda, Postgres, Mongo, and Python. Its key value lies in intelligent, contextual analysis and notifications for advertising-based videos and imagery stored and maintained in real time. Newly ingested images and videos are archived in S3 and accessed directly via S3 URLs.

The APIs used by client programs (UX or background) support a Contextual Data Model that offers various convenient features and returns filtered, transformed, and AI-enhanced slices of the database. These REST APIs are documented at [https://adology.ai/live/…](https://adology.ai/live/…).

The data stores (S3, Postgres, Mongo) that power the Adology Contextual Data Model are filled by Acquisition Engines, which are EC2 or dynamically launched Lambda functions that pull data from remote sources, including Facebook, SerpApi, and customer-specific databases.

Most of the data, such as brands and advertising assets, is shared by all customers, whereas brandspaces are private to each customer.

Different non-UI components run on either EC2 or Lambda functions and are connected through a series of SQS queues. The SQS queues and Lambdas support parallelization of workflows for maximum concurrency.


## Operating Environment

Adology strives for near-continuous availability of its computing infrastructure, with each component able to fail independently and be restored within minutes. It is heavily dependent on AWS-managed services (MongoDB Atlas, Postgres RDS, SQS, AWS Load Balancer, AWS CloudFront).

<img src="https://billdonner.com/adology/two.png" width=400>

### Backend

- **Amazon RDS for PostgreSQL**: Managed service handling administrative tasks (backups, patching, scaling, failover).
- **MongoDB Atlas**: Fully managed cloud service for backups, scaling, replication, and maintenance.
- **Amazon S3**: Data is automatically replicated for high durability.
- **Amazon SQS**: Fully managed message queuing, used to decouple and scale pipeline components.
- **EC2 Servers**: Initially, two EC2 instances are deployed behind a load balancer. One functions as the Adology API server, and the other remains a warm standby (or handles static content).
- **Amazon API Gateway**: Serves as an interface between clients and backend services, enabling communication via WebSocket connections and message queues.
- **AWS Elastic Load Balancer (ELB)**: Distributes traffic across multiple EC2 instances, automatically detecting and avoiding
  unhealthy instances.
- **Amazon Cloudwatch**: Monitors queue depths of SQS queues, S3, lambda performance, triggers.   PostgreSQL RDS has built-in CloudWatch metrics and enhanced monitoring.   MongoDB Atlas requires manual integration with CloudWatch via EventBridge. Both databases are monitored for query latency, CPU usage, memory, and disk I/O.

### Frontend

The main interface is a JavaScript application (the Adology UI) that serves as a Dashboard and communicates with the Adology API servers. Many API calls trigger long-running background Lambda functions, so WebSockets are used to notify the client when processing is complete. The UI then makes an additional API call to retrieve and display processed data.

The Dashboard programs use the Amazon API Gateway to communicate with SQS and sends all HTTPS: requests to the Load Balancer for delivery to the Adology Web Server and Adology Application Server.

Several Front End modules (Inquire, Enrich, AdSpend Reporting)follow this same pattern:

- optionally setup a WebSocket for notifications
- accept User Input from the Dashboard and post a Message on an SQS queue which triggers a long running operation on the backend.
- poll for completion by checking the database or receive a notificaiton of completion by a WebSocket call
- make an API call to receive the results that are now in the database


## Central Data Spine

The primary table in the central spine is the **Brand Descriptions Table**. There are two types of entries:

- **Analyzed Brands**: These brands are requested by customers and are fully analyzed by Adology’s AI Engines.
- **Tracked Brands**: These brands are generally specified by Adology and are downloaded and stored in S3, but are not analyzed until a customer requests it.

When a customer requests information about a brand, all videos and images previously captured by Adology become available, along with any new assets. The collection is then made accessible to the Analysis AI Engines.

#### Ad Descriptions/Attributes

These are natural language, open-ended text attributes mapped to an ad through AI processing. There are over 50 attributes for which the system generates descriptions. They are all open-ended and used for:

- Text summaries throughout the application
- INSIGHT text summaries and chatbot (INQUIRE)
- Inputs for trend detection

#### Detailed/Long Descriptions

A single field is stored for each ad: a 500-word (max) description that captures comprehensive details.

**Labels:** Close-ended, finite, hardcoded labels applied to each ad based on predefined taxonomies. Two label sets are supported: UNIVERSAL and TREND labels. Labels are used to:

- Power graphs and charts
- Drive functionality in modules such as INSIGHT and TRACK
- Serve as inputs for trend detection

#### Embeddings

Embeddings are numerical representations of creatives or text. Multiple types of embeddings are maintained:

- **Text embeddings** of detailed descriptions, used to map images to insights in INSIGHT. This is done by aligning the embedding of INSIGHT text with detailed description embeddings to find the most relevant ad.
- **Visual embeddings** of ads, used for trend detection.
- Text embeddings of detailed descriptions are also used by the chatbot to retrieve relevant information.

## Spine Operations

Keeping the spine up to date is primarily driven by the Brand Record in the Brand Data Table.

 
<img src=https://billdonner.com/adology/three.png width=800>

Adology attempts to keep the spine as up to date as possible. As such there is a background process (SpineScheduler) which  periodically runs thru the entire table of brands sequentially and places messages on the low-priority Aquire SQS Q to trigger Lambda functions to update those brands needing an update. Some brands will be multi-sourced. In this case messages will be placed on multiple Acquire-like SQS QUeues to trigger different Lambdas for different processing.

Adology administrators will pre-load popular and common brands so that they are already cached before anyone asks for them, but nevertheless there will be situations where a user at a dashboard requests a brand that has never been seen. In this case the general flow is the same but it is performed on a different set of higher-priority SQS Queues.

In either case the Lambdas end up determining how to Acquire the content and then pass the flow to another set of SQS Queues to activate different Lambdas to  Download content from the remote sources and store in S3. The step can be done thru streaming so that there is no discrete upload step thus providing maximum concurrency with.

Finally, in all cases there is considerable AI Intelligence applied to the newly downloaded content. This involves potentially many separate calls from Lambdas into the OpenAI API, which must be sequenced due to content dependecies.



---

## Adology Intelligent Data Storage

Adology is designed not just as a database but as an intelligence engine. Storage decisions must balance retrieval efficiency with AI processing costs.

### Data Hierarchy

The Adology Data Store is organized in a hierarchy of logical data stores that are continuously updated by the Adology Service Infrastructure:

 
<img src=https://billdonner.com/adology/four.png width=800>
 

1. **Organization Account**
   Represents a customer organization or business entity using Adology. Manages multiple brand-focused workspaces (brandspaces).

2. **Brandspaces**
   Each user is connected to at least one brandspace and can switch brandspaces as needed. A brandspace is centered around a primary brand and includes competitor & followed brands, defining the scope of analysis.

3. **Brand-Level Intelligence**
   Includes brand metadata (logos, category, website) and AI-generated brand descriptions/trends. Each brand can connect to multiple channels (e.g., Meta, SERP, YouTube).

4. **Channel-Level Tracking**
   Each brand runs ads across multiple channels. Ads are categorized by their originating channel.

5. **Shared Brand Data Store**
   Brand data from all sources is stored in S3 and shared across all Adology users.

6. **Shared Ads Data Store**
   Ads from all sources are associated with a brand and accumulated permanently in S3, with data shared across all Adology users.

---

## AI Summarization & Brand-Level Insights

When sufficient ad descriptions are available, they are aggregated into brand-wide insights. AI generates high-level summaries of key brand themes, messaging patterns, and performance insights, which remain up-to-date as new ads are processed.

### Brand-Level AI Data (Aggregated from Ad-Level AI Data)

- **Theme Clusters**: Groups of ads sharing common storytelling techniques
- **Attribute Summaries**: AI-generated analysis of features, claims, and offers
- **Messaging Trends**: AI-detected patterns in claims, benefits, and CTA effectiveness

---

## Reports & Competitive Analysis

Reports are generated and displayed in modules such as **Inspire Recs & Trends** and **Market Intelligence/Brand Details**. These reports synthesize AI-generated insights from ad-level and brand-level data to provide client-facing information while reducing API costs.

### Report Generation Process

1. AI pre-generates reports at the end of the data acquisition process, based on competitor & follower lists.
2. Reports include competitive benchmarking, creative trends, and strategic recommendations.
3. **Example (Generating Ad Recommendations in INSPIRE):**
   - The system pulls AI-generated ad & brand-level data for competitor & follower brands.
   - An Ad Recommendation prompt is executed.
   - Results are saved and displayed in the **INSPIRE REC** module.

### Report Updates

Reports are refreshed when two of the following occur:

- A follower or competitor brand is added/removed.
- A new ad arrives from a followed or competitor brand.
- Additional logic: If seven days have elapsed **and** new ads have been received.

### Stored Insights

Reports act as snapshots of key insights, optimizing cost efficiency by avoiding real-time recalculations. Examples include:

- **Ad Theme Comparisons** (e.g., *"Nike focuses on speed, while Adidas emphasizes lifestyle."*)
- **Messaging Effectiveness Reports** (e.g., *"Top 3 CTAs in the running shoe market."*)
- **Trend Tracking** (e.g., *"Limited-time offers are increasing in Meta ads."*)

---

## Brand Pipeline Optimizations

1. **Efficient Data Storage**
   - **Per-Ad Storage**: Ad descriptions, labels, and embeddings are stored per ad.
   - **Incremental Brand-Level Updates**: Brand-wide insights are updated only when necessary.

2. **Caching & Performance Enhancements**
   - **Pre-built Reports & Dashboards**: Reduces OpenAI API calls and improves performance.
   - **Dashboard Analytics**: Stored separately for fast loading.
   - **Competitor & Follower List**: Drives which reports need updates.

3. **Brandspaces Define Intelligence Scope**
   - **Managed at an Organizational Level**: Determines brands & competitors to track.
   - **Organizational Brandspaces**: Ensures intelligence remains focused on the requirements of each organization.



---

## The Main Acquisition Flow

A key performance metric is to process 10,000 ads in under 10 minutes. The overall process is:

<img src="https://billdonner.com/adology/one.png" width=400>

1. A frontend dashboard program places a message on the **Apify** SQS queue with a **BrandName**.
2. Multiple Apify processes handle this queue, using the Apify API to split the **BrandName** into a stream of URLs for specific images and videos.
3. The stream of URL-based messages is placed on the **Acquire** SQS queue, triggering downloads to S3 in parallel.
4. When each item is in S3, a new message (with S3 URL and metadata) is placed on the **Analyze** SQS queue.
5. Analysis processes consume these messages and perform AI tasks in parallel, limited by the AI service’s capacity.
6. All analyzed data is then stored in the database, and the frontend checks for completion or is notified via WebSockets.

Adology’s dashboard accesses S3 directly and only writes to the database through the Adology API.

---

## Other Flows

Most UX functionality involves responding to user actions by calling an Adology API endpoint. Additional recurring processes (e.g., periodic CRON jobs) place messages on the Apify or SerpApi SQS queues to keep brand data updated.

---

## Appendices

### Common Themes & General Recommendations Across All Modules

- **Coding Standards & Documentation**
  - A consistent code style (PEP8) is recommended, enforced by linters (e.g., flake8, Black).
  - Comprehensive docstrings and inline comments should be included for clarity and maintainability.

- **Centralized Configuration & Secrets Management**
  - All hardcoded values (API keys, model names, S3 bucket names) should be externalized in configuration files or environment variables.
  - A secure secrets management solution (e.g., AWS Secrets Manager) is advised.

- **Structured Logging & Monitoring**
  - Print statements should be replaced with a structured logging framework that includes contextual data (user IDs, request IDs, timestamps).
  - Centralized logging and monitoring systems (e.g., ELK, CloudWatch) improve production observability.

- **Error Handling & Retry Logic**
  - Specific exception handling (e.g., JSONDecodeError, KeyError) should be used instead of broad `except` blocks.
  - Standardized retry mechanisms (e.g., the tenacity library) are recommended for transient errors in external API/S3 interactions.

- **Separation of Concerns & Modularity**
  - Code should be refactored to separate business logic from I/O operations (database, S3, external APIs).
  - Duplicated logic can be consolidated into shared modules and utilities.

- **Concurrency & Asynchronous I/O**
  - Thread pool sizes should be reevaluated, and asynchronous I/O (e.g., asyncio, aiohttp) considered for network-bound tasks.
  - Proper thread safety is necessary when sharing resources.

- **Testing & CI/CD**
  - Test coverage should be expanded with unit, integration, and end-to-end tests.
  - Static analysis, security scans, and dependency checks should be integrated into the CI/CD pipeline.

- **External API Integration**
  - Input validation and robust error handling around external API calls are crucial.
  - Retries and fallback strategies handle transient API failures effectively.

---

### Best Practices for Writing Functions for Parallel Execution

- **Statelessness**
  Functions should avoid depending on or modifying shared state. Any required state is preferably stored in a database or S3.

- **Idempotency**
  Multiple identical invocations should produce the same result without unintended side effects. This approach supports safe retries and distributed workflows.

- **Robust Error Handling**
  Comprehensive error handling and logging greatly assist in diagnosing issues in parallel environments.

- **Granularity**
  Smaller, well-defined functions that perform a single unit of work scale and maintain more easily.

- **Testing**
  Functions should be tested in isolation and within the parallel execution framework to ensure correct behavior in concurrent scenarios.

---

### Amazon SQS

Amazon SQS (Simple Queue Service) is a fully managed message queuing service designed to decouple and scale distributed systems, microservices, and serverless applications:

- **Fully Managed**: No server provisioning or failover configuration is required.
- **Scalability & Performance**: SQS automatically scales to handle extremely high message throughput.
- **Reliability & Availability**: Internal replication ensures message durability.

---

## Brand Descriptions Table Specification

### Overview

This section describes the design and processing steps for building a **Brand Descriptions Table**. The table summarizes each brand’s creative attributes (features, claims, benefits, offers, key messages, etc.) and supports downstream modules like Brand Details and Inspire. It is a key component of the Central Data Spine.

#### Primary User Personas

- **Backend Functions**: Data processing, analytics, and integration with downstream modules.

#### Jobs to Be Done

- **Summarize Brand Attributes**: Provide a cohesive view of features, claims, benefits, etc.
- **Support Downstream Modules**: Supply standardized data for Brand Details, Inspire, and Ad Spend Reporting.
- **Ensure Consistent Data Representation**: Harmonize attribute values across both primary and competitor brands.

---

### Context and Functional Requirements

**Purpose & Approach**
- One row per unique brand item.
- Each attribute is derived from source fields in the ad description:
  - `features` → **Features**
  - `benefits` → **Benefits**
  - `claims` → **Claims**
  - `offers` → **Offers**
  - `Long Description` → **Themes**
  - `key message` → **Key Messages**

#### Key Attributes

**MVP**
- Features, Benefits, Claims, Offers, Key Messages

**V2**
- Visual Styles, Emotions, Category Entry Points

Each attribute may include a list of unique items, corresponding categories, and aggregated counts or summaries.

---

### Data Processing Pipeline

The pipeline extracts raw data, harmonizes entries (with GPT), categorizes them, counts occurrences, and generates summaries. Both initial processing and incremental updates are supported.

1. **Raw Extraction**
   - Pull raw creative attributes from ad descriptions.
   - Store ad-level data with timestamps.

2. **Unique Attribute Extraction & Harmonization**
   - **Initial Run**: Gather all raw entries for an attribute (e.g., Features). Use GPT or NLP to harmonize synonyms and create a unified list of unique features plus categories. Store them in the Brand Descriptions Table.
   - **Incremental Updates**: Process only new ads. Check for any new features (or other attributes), expand lists, and regenerate categories and summaries as needed.

3. **Counting & Summaries**
   - Calculate how many ads match each unique feature or category.
   - Generate GPT-based summaries explaining how a brand leverages its features, claims, offers, etc.

---

### Competitor Brands Processing

The same pipeline is repeated for competitor or follower brands. Each competitor may have a separate Brand Descriptions Table entry. Comparisons against the primary brand are handled in downstream modules.

---

### GPT Feedback & Enhanced Pipeline Architecture

**Strengths**
- Comprehensive coverage of brand attributes
- Hierarchical flow from raw extraction to summarized outputs
- Incremental updates reduce redundant processing

**Improvements**
- **Batching & Caching**: Group entries before GPT calls and cache results
- **Ambiguity Handling**: Allow fallback logic or human review for low-confidence mappings
- **Version Control**: Track changes to the Brand Descriptions Table over time

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

## Brand Descriptions Pipeline

- **Integrates GPT** for harmonizing language and generating narrative summaries.
- **Supports Incremental Updates** to efficiently process new ad data.
- **Provides Robust Error Handling** and quality assurance.
- **Facilitates Downstream Modules** such as Brand Details, Inspire, and Ad Spend Reporting.
- **Accommodates User-Defined Inputs** via a Taxonomy Manager for added flexibility.
        
