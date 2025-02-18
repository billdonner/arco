# Adology Architecture v0.8

## Overview

Adology is a intelligent service that analyzes digital advertising by applying knowledge processing to evaluate content(video, image, and text) as it is ingested into a permanent Adology repository.

At its core, Adology is not just a passive ad-tracking tool. Instead, it functions as an AI-powered intelligence engine that captures:

- The raw reality of ad creatives (metadata, images, videos, and text)
- The structured interpretation of these creatives (AI-generated labels, scores, and embeddings)
- The higher-level knowledge extracted from AI insights (brand-wide trends, comparative analysis, and strategic reports)

Adology’s architecture is designed to systematically store, analyze, and generate insights from advertising data. The system must support highly structured AI-driven analysis while also maintaining efficient retrieval of brand, ad, and intelligence reports—all within a cost-effective, scalable database structure.


Adology's data architecture supports multiple users within customer accounts, each managing multiple *brandspaces*. Each brandspace focuses on a primary brand, tracking competitor brands and followed brands to define the scope of intelligence gathering. A user can switch brandspaces for one customer organization at any time but requires a unique email and separate login to support multiple organizations.

Adology largely operates as a conventional, Internet-accessible database and content management service built on top of well-known data stores and access methods, including AWS, EC2, S3, SQS, Lambda, Postgres, Mongo, and Python. Its key value lies in intelligent, contextual analysis and notifications for advertising-based videos and imagery stored and maintained in real time. Newly ingested images and videos are archived in S3 and accessed directly via S3 URLs.

The APIs used by client programs (UX or background) support a Contextual Data Model that offers various convenient features and returns filtered, transformed, and AI-enhanced slices of the database. These REST APIs are documented at [https://adology.ai/live/…](https://adology.ai/live/…).

The data stores (S3, Postgres, Mongo) that power the Adology Contextual Data Model are filled by Acquisition Engines, which are EC2 or dynamically launched Lambda functions that pull data from remote sources, including Facebook, SerpApi, and customer-specific databases.

Most of the data, such as brands and advertising assets, is shared by all customers, whereas brandspaces are private to each customer.

Different non-UI components run on either EC2 or Lambda functions and are connected through a series of SQS queues. The SQS queues and Lambdas support parallelization of workflows for maximum concurrency.


## Operating Environment

Adology strives for near-continuous availability of its computing infrastructure, with each component able to fail independently and be restored within minutes. It is heavily dependent on AWS-managed services (MongoDB Atlas, Postgres RDS, SQS, AWS Load Balancer, AWS CloudFront).

<img src="https://billdonner.com/adology/two.png" width=700>

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


# **Central Data Spine**

The **Brand Descriptions Table** serves as the primary table in the central data spine. It contains two categories of brand entries:

- **Analyzed Brands**: These brands are explicitly requested by customers and undergo full AI-driven analysis within Adology's processing engine.
- **Tracked Brands**: These brands are selected by Adology for tracking purposes. They are stored in S3 but remain unanalyzed until explicitly requested by a customer. The AI processing, which is expensive, is explicitly deferred until a customer changes its status to "analyzed"

The analysis of ads is a complex and time-consuming process and requires careful coordination to ensure a pleasant user experience.

These are the fundamental data components that are tracked  by the Spine:

## **Ad Descriptions and Attributes**

Natural language attributes are mapped to each ad through AI processing. Over 50 attributes are generated, all of which serve key functions, including:

- Providing text summaries across the application.
- Powering **INSIGHT** text summaries and chatbot functionality (**INQUIRE**).
- Acting as inputs for **trend detection**.

### **Detailed Ad Descriptions**

A single **500-word (max) description** is stored for each ad, providing a comprehensive textual summary. This is the primary field that is analyzed by many AI prompts.

### **Labels**

Close-ended, predefined labels are applied to ads using **UNIVERSAL** and **TREND** label taxonomies. Labels are used for:

- Enabling structured insights in graphs and charts.
- Powering features within modules such as **INSIGHT** and **TRACK**.
- Supporting **trend detection** models.

### **Embeddings**

Embeddings provide numerical representations of ad creatives or text content. Multiple types of embeddings are stored:

- **Text embeddings** of detailed descriptions, used for mapping images to insights within **INSIGHT**.
- **Visual embeddings** of ads, leveraged for **trend detection**.
- **Chatbot retrieval embeddings**, ensuring accurate information retrieval based on user queries.

   

# **Spine Operations**


The **Brand Descriptions Table** drives all operations within the central spine.

<img src="https://billdonner.com/adology/three.png" width=700>

Adology continuously updates the spine through a **background process (SpineScheduler)**, which sequentially scans the entire table of brands and places messages onto a **low-priority Acquire SQS Queue**. This triggers **Lambda functions** that update brands flagged for updates. In cases where brands source data from multiple locations, multiple Acquire-like SQS queues distribute processing across different Lambda functions.

To optimize response times, **Adology administrators pre-load popular and commonly accessed brands** into the system, ensuring their data is readily available before a request occurs. However, if a user requests a brand that has never been analyzed, the system follows the same general process but prioritizes the request using **high-priority SQS queues**.

### **Acquisition and Processing Flow**

1. At a high level, adology triggers **one of a set of SQS queues** to **download** content from brand-specific external sources into S3.
2. **Lambda functions** determine the acquisition strategy for each brand.
3. **Streaming-based ingestion** maximizes concurrency by eliminating a discrete upload step.
4. **AI-based content analysis** is performed using OpenAI API calls, which must be executed sequentially due to content dependencies.
 

# **Adology Intelligent Data Storage**

Adology is **more than a database**—it is an **intelligent processing engine**. Data storage is optimized for **fast retrieval** while balancing **AI processing costs**.

## **Data Hierarchy**

Adology maintains a structured hierarchy of logical data stores, continuously updated by the **Adology Service Infrastructure**.

<img src="https://billdonner.com/adology/four.png" width=700>

1. **Organization Account**
   Represents a customer entity that manages multiple brandspaces.

2. **Brandspaces**
   Each user is connected to at least one **brandspace**, which is centered on a **primary brand** and includes competitors and followed brands. Brandspaces define the **scope of analysis**.

3. **Brand-Level Intelligence**
   Includes:
   - Brand metadata (logos, categories, websites).
   - AI-generated **brand descriptions and trends**.
   - Connections to multiple data sources (**Meta, SERP, YouTube**).

4. **Channel-Level Tracking**
   Each brand runs ads across multiple channels. Ads are categorized based on **originating platform**.

5. **Shared Brand Data Store**
   - Stores **brand metadata** from all sources in **S3**.
   - Data is shared across **all Adology users**.

6. **Shared Ads Data Store**
   - Ads are permanently stored in **S3**, categorized by brand.
   - Data is available to **all Adology users**.

   

## **AI Summarization & Brand-Level Insights**

Once sufficient ad descriptions have been processed, Adology aggregates them into **brand-wide insights**. The AI continuously updates **high-level brand themes, messaging patterns, and performance insights**.

### **Brand-Level AI Data (Aggregated from Ad-Level Data)**
- **Theme Clusters** → Identifies groups of ads sharing **common storytelling techniques**.
- **Attribute Summaries** → AI-generated analysis of **features, claims, and offers**.
- **Messaging Trends** → Detects evolving patterns in **claims, benefits, and call-to-action (CTA) effectiveness**.
 

### **Reports & Competitive Analysis**

Adology generates **competitive reports** that synthesize **ad-level and brand-level AI insights**. These reports are integrated into **Inspire Recs & Trends** and **Market Intelligence/Brand Details** modules.

#### **Report Generation Process**
1. **AI pre-generates reports** at the conclusion of the data acquisition process.
2. Reports provide:
   - **Competitive benchmarking**.
   - **Creative trends analysis**.
   - **Strategic recommendations**.

3. **Example: Generating Ad Recommendations in INSPIRE**
   - **Data Acquisition** → AI pulls brand and competitor data.
   - **Ad Recommendation Prompt** → AI evaluates trends and ad effectiveness.
   - **Final Output** → Recommendations are displayed in **INSPIRE REC**.

#### **Report Updates**
Reports refresh **when two of the following conditions are met**:

- A **follower or competitor brand** is **added or removed**.
- A **new ad** arrives from a **followed** or **competitor brand**.
- Additional logic: **If seven days have elapsed AND new ads have been received**.

### **Stored Insights**
Reports act as **snapshots** of AI-generated insights, **reducing API processing costs** while preserving **data accuracy**.

**Examples of insights stored in reports:**
- **Ad Theme Comparisons** → *Nike emphasizes speed; Adidas focuses on lifestyle*.
- **Messaging Effectiveness Reports** → *Top 3 most successful CTAs in the running shoe industry*.
- **Trend Tracking** → *Increase in limited-time offers in Meta ads*.
 
### **Enhancements & Improvements**
To ensure maximum system performance, Adology employs:

- **Pre-built AI dashboards** → Minimize OpenAI API calls for efficiency.
- **Cached Reports & Insights** → Ensures fast retrieval for users.
- **Parallelized AI Processing Pipelines** → Optimizes query execution across distributed systems.

 
The **central data spine, intelligent data storage, and AI-powered insights** collectively power Adology’s **brand intelligence engine**. The system continuously refines and updates its models to **deliver actionable, high-quality marketing analytics** while **minimizing processing overhead**.


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


The same pipeline is repeated for competitor or follower brands. Each competitor may have a separate Brand Descriptions Table entry. Comparisons against the primary brand are handled in downstream modules.


###  Brand Pipeline Optimizations

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



   

## The Main Acquisition Flow Example

A key performance metric is to process 10,000 ads in under 10 minutes. The overall process is:

<img src="https://billdonner.com/adology/one.png" width=400>

1. A frontend dashboard program places a message on the **Apify** SQS queue with a **BrandName**.
2. Multiple Apify processes handle this queue, using the Apify API to split the **BrandName** into a stream of URLs for specific images and videos.
3. The stream of URL-based messages is placed on the **Acquire** SQS queue, triggering downloads to S3 in parallel.
4. When each item is in S3, a new message (with S3 URL and metadata) is placed on the **Analyze** SQS queue.
5. Analysis processes consume these messages and perform AI tasks in parallel, limited by the AI service’s capacity.
6. All analyzed data is then stored in the database, and the frontend checks for completion or is notified via WebSockets.

Adology’s dashboard accesses S3 directly and only writes to the database through the Adology API.

   

## Other Flows

Most UX functionality involves responding to user actions by calling an Adology API endpoint. Additional recurring processes (e.g., periodic CRON jobs) place messages on the Apify or SerpApi SQS queues to keep brand data updated.

   

## Appendices

### Project Requirements

- **JSON, not CSV**
  Data should preferrably stored in database fields or as JSON blobs, and never as CSV. JSON is more expressive. CSV can be generated at the last moment when an API call must deliver an explicit CSV object.
  
  - **SQS First**
  Lambda expressions should not be invoked directly from application software. Instead, a message with a custom payload should be placed on a FIFO SQS queue which will in turn trigger one of a group of lambas to handle the message,.
   

- **Coding Standards & Documentation**
   A consistent code style (PEP8) is recommended, enforced by linters (e.g., flake8, Black).
   Comprehensive docstrings and inline comments should be included for clarity and maintainability.

- **Centralized Configuration & Secrets Management**
  All hardcoded values (API keys, model names, S3 bucket names) should be externalized in configuration files or environment variables.
 

- **Structured Logging & Monitoring**
 Print statements should be replaced with a structured logging framework that includes contextual data (user IDs, request IDs, timestamps).  CloudWatch logging and monitoring systems will report exceptional events to Adology operions

- **Error Handling & Retry Logic**
  Specific exception handling (e.g., JSONDecodeError, KeyError) should be used instead of broad `except` blocks.
  Standardized retry mechanisms (e.g., the tenacity library) are recommended for transient errors in external API/S3 interactions.

- **Separation of Concerns & Modularity**
 Code should be refactored to separate business logic from I/O operations (database, S3, external APIs).
 Duplicated logic can be consolidated into shared modules and utilities.

- **Concurrency & Asynchronous I/O**
 Thread pool sizes should be evaluated. There is a two level concurrency scheme - first the lambdas provide fanout and secondly the internal python concurrent worker threads provide the opportunity to tune concurrency.  Proper thread safety is necessary when sharing resources.


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

# Database
 

This is a stripped down representation of the logical tables needed to run the Adology Service. Consider this an appendix to the Architecture document.


## Background Flow

There is a background process, either a Kron job, or Celery job, or perhaps a dormant EC2 process that periodically checks for new content on all of its sources for each Brand in the Brands Table.

As new content / ads flow into the acquisition engines, the Ads Table is is updated. The new content is uploaded to S3 as it is being acquired.

Depending on the brand and whether it is being followed or analyzed by any customer, a message specifying additional AI work is enqued to a SQS queue for Lambda processing.

## User Interactive Flow

Apart from the background, a Dashboard user can enter an arbitrary Brand name , requesting complete analysis of the competion according to a preset profile in the active brandspace/workspace. In this case a message containing the brandname and a websocket ID is enqueued to the appropriate SQS queue for analysis and processing.

Customer Administrators set up users and separately workspaces for their users. These workspaces/brandspaces contain specific lists of brands to track and brands to analyze. As the user interface adjusts these lists in response to administrative requests, it must also adjust the appropriate row in the Brands Table, to inser or remove onself from these lists.

Most users will work for one customer. A User working for multiple customers will need to use distinct emails.



<img src="https://billdonner.com/adology/dbo-erd1.png" width=500>


### Brands Table

| Field                   | Data Type | Description |
|-------------------------|----------|-------------|
| `name`                 | TEXT      | Name of the brand (Primary Key) |
| `metadata`             | JSONB     | Additional structured metadata about the brand |
| `description`          | TEXT      | Description of the brand |
| `demographics`         | JSONB     | Demographic information about the brand's audience |
| `last_ingestion`       | TIMESTAMP | Timestamp of the last data ingestion |
| `image_count`          | INTEGER   | Number of images associated with the brand |
| `video_count`          | INTEGER   | Number of videos associated with the brand |
| `text_count`           | INTEGER   | Number of text-based ads associated with the brand |
| `followed_by_customer` | BYTEA     | Blob representing all of the customers following this brand |
| `analyzed_by_customer` | BYTEA     | Blob representing all of the customers analyzing this brand |

The **Brands Table** serves as the authoritative record for all brands in the system, tracking not only metadata and descriptions but also real-time ingestion statistics and competitive relationships. The use of **JSONB fields** for metadata and demographics ensures flexibility in storing structured brand data, while **BYTEA blobs** for `followed_by_customer` and `analyzed_by_customer` enable efficient retrieval of relationships without complex joins. The `name` field, acting as the primary key, simplifies external integrations but requires careful normalization and indexing to maintain consistency. Proper management of this table allows for dynamic insights into brand visibility, audience engagement, and market competition.

---

### Ads Table

| Field             | Data Type  | Description |
|------------------|-----------|-------------|
| `ad_id`         | UUID       | Unique identifier for the ad |
| `brand_name`    | TEXT       | Foreign key linking to the Brands table (`name`) |
| `content_url`   | TEXT       | URL pointing to the ad content |
| `source`        | TEXT       | Platform where the ad was acquired (e.g., TikTok, Meta, Facebook) |
| `acquisition_time` | TIMESTAMP | Timestamp of when the ad was acquired |

The **Ads Table** stores all acquired advertisements, linking them to a brand. It maintains metadata about the source of the ad and when it was acquired, ensuring a structured way to track advertising content. The `brand_name` field acts as a foreign key, linking each ad to a specific brand, providing an easy lookup for all advertisements related to a given brand.

---

### Customers Table

| Field           | Data Type | Description |
|---------------|----------|-------------|
| `name`        | TEXT     | Name of the customer (Primary Key) |
| `demographics` | JSONB    | Structured demographic data for the customer |

The **Customers Table** is designed to store information about individual customers or customer segments interacting with brands in the system. The `name` field serves as the primary identifier, while the `demographics` field, stored as a **JSONB object**, allows for flexible storage of structured data such as age group, geographic location, purchasing behavior, or other attributes. This structure ensures that customer insights can be dynamically expanded without modifying the schema, making it adaptable to evolving data needs.

---

### Users Table

| Field            | Data Type | Description |
|-----------------|----------|-------------|
| `name`          | TEXT     | Name of the user (Primary Key) |
| `customer_name` | TEXT     | Name of the customer this user is associated with (Foreign Key to Customers table) |
| `demographics`  | JSONB    | Structured demographic data for the user |
| `current_workspace` | TEXT  | Name of the currently active workspace for the user |
| `workspaces`    | JSONB    | List of all workspaces associated with the user |

The **Users Table** stores user-specific details, including their name, demographics, and workspace associations. The `customer_name` field links each user to a specific customer, enabling multi-user relationships within a customer entity. The `current_workspace` field tracks the actively used workspace, while the `workspaces` field, stored as a **JSONB array**, allows for dynamic workspace management. This structure ensures users can belong to multiple workspaces under a single customer entity.

---

### Workspaces Table

| Field                | Data Type | Description |
|----------------------|----------|-------------|
| `name`              | TEXT      | Name of the workspace (Primary Key) |
| `customer_name`     | TEXT      | Name of the customer associated with this workspace (Foreign Key to Customers table) |
| `demographics`      | JSONB     | Structured demographic data about the workspace |
| `metadata`          | JSONB     | Additional structured metadata related to the workspace |
| `tracked_brands`    | JSONB     | List of brands actively tracked by this workspace |
| `analyzed_brands`   | JSONB     | List of brands being analyzed by this workspace |

The **Workspaces Table** defines logical groupings within a customer's ecosystem, allowing users to operate in distinct environments. The `customer_name` field links each workspace to a customer, ensuring proper segmentation. The **JSONB fields** provide flexible storage for demographic and metadata details, supporting a variety of business use cases, including user permissions, settings, and analytics. The `tracked_brands` field contains a list of brands that this workspace is actively monitoring, while the `analyzed_brands` field stores a list of brands undergoing deeper analytical processes.


## MongoDB Schema
```json
{
  "brands": {
    "name": { "type": "String", "unique": true, "required": true },
    "metadata": { "type": "Object" },
    "description": { "type": "String" },
    "demographics": { "type": "Object" },
    "last_ingestion": { "type": "Date" },
    "image_count": { "type": "Number", "default": 0 },
    "video_count": { "type": "Number", "default": 0 },
    "text_count": { "type": "Number", "default": 0 },
    "followed_by_customer": { "type": "Binary" },
    "analyzed_by_customer": { "type": "Binary" }
  },
  
  "ads": {
    "ad_id": { "type": "UUID", "required": true },
    "brand_name": { "type": "String", "ref": "brands", "required": true },
    "content_url": { "type": "String", "required": true },
    "source": { "type": "String", "enum": ["TikTok", "Meta", "Facebook", "Other"] },
    "acquisition_time": { "type": "Date", "required": true }
  },
  
  "customers": {
    "name": { "type": "String", "unique": true, "required": true },
    "demographics": { "type": "Object" }
  },
  
  "users": {
    "name": { "type": "String", "unique": true, "required": true },
    "customer_name": { "type": "String", "ref": "customers", "required": true },
    "demographics": { "type": "Object" },
    "current_workspace": { "type": "String", "ref": "workspaces" },
    "workspaces": { "type": "Array", "items": { "type": "String", "ref": "workspaces" } }
  },
  
  "workspaces": {
    "name": { "type": "String", "unique": true, "required": true },
    "customer_name": { "type": "String", "ref": "customers", "required": true },
    "demographics": { "type": "Object" },
    "metadata": { "type": "Object" },
    "tracked_brands": { "type": "Array", "items": { "type": "String", "ref": "brands" } },
    "analyzed_brands": { "type": "Array", "items": { "type": "String", "ref": "brands" } }
  }
}
```

## Postgres Schema
```json
-- Brands Table
CREATE TABLE brands (
    name TEXT PRIMARY KEY,
    metadata JSONB,
    description TEXT,
    demographics JSONB,
    last_ingestion TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    image_count INTEGER DEFAULT 0,
    video_count INTEGER DEFAULT 0,
    text_count INTEGER DEFAULT 0,
    followed_by_customer BYTEA,
    analyzed_by_customer BYTEA
);

-- Ads Table
CREATE TABLE ads (
    ad_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_name TEXT REFERENCES brands(name) ON DELETE CASCADE,
    content_url TEXT NOT NULL,
    source TEXT CHECK (source IN ('TikTok', 'Meta', 'Facebook', 'Other')),
    acquisition_time TIMESTAMP NOT NULL
);

-- Customers Table
CREATE TABLE customers (
    name TEXT PRIMARY KEY,
    demographics JSONB
);

-- Users Table
CREATE TABLE users (
    name TEXT PRIMARY KEY,
    customer_name TEXT REFERENCES customers(name) ON DELETE CASCADE,
    demographics JSONB,
    current_workspace TEXT REFERENCES workspaces(name) ON DELETE SET NULL,
    workspaces JSONB -- Array of workspace names
);

-- Workspaces Table
CREATE TABLE workspaces (
    name TEXT PRIMARY KEY,
    customer_name TEXT REFERENCES customers(name) ON DELETE CASCADE,
    demographics JSONB,
    metadata JSONB,
    tracked_brands JSONB, -- Array of brand names
    analyzed_brands JSONB -- Array of brand names
);
```

