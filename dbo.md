## DB

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
