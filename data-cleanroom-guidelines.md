#ECAPI Guidelines for Data Clean Rooms 

## Executive Summary:
The ECAPI standard release in April 2026 focused on server to server transmission of conversion data. A core value proposition of the specification is the standardization of the data attributes and definitions used to measure conversions. 

These guidelines were developed by the ECAPI Working Group to ensure smooth application of the ECAPI standard to data transfer use cases. The guidelines focus on data moving between systems, not how the data should be stored within any specific system. With the specific use case: how data is packaged and sent to a data clean room.
  
## 1. Hashing/Encryption Requirement

If the ECAPI v1.0 standard noted that a value is SHA256 hashed, then that value  MUST be either SHA256 hashed (per ECAPI v1.0 normalization rules) or encrypted (via Parquet Modular Encryption or application-layer encryption) before upload to a DCR. Values that are not encrypted or hashed or otherwise appropriately protected MUST NOT appear in any Parquet file delivered to a clean room.

### 1a. Data Compression

While not strictly required by this standard, compression is a highly recommended best practice for batch data transfers. It significantly reduces payload sizes which directly minimizes: transfer latency, network bandwidth utilization, egress costs, disk I/O, and storage footprints, while optimizing downstream ETL pipeline throughput.

## 2. Parquet Layout

ECAPI events contain nested, variable-length data: an event can have multiple email addresses, multiple items, multiple addresses. Parquet can represent this in multiple ways. The example below represents a common schema that advertisers can use.

### 2.1 Nested Structs (Dremel Encoding)
This is the recommended format. Preserves the ECAPI hierarchy using Parquet's native nested types. One row per event. Children stored as LIST of STRUCT. This event produces 1 row.

| data_set_id | id | event_type | value | user_data.email_address | properties.items |
| --- | --- | --- | --- |--- | --- |
|acme-meta-01 | evt-7890 | purchase | 149.99 | ["a1b2c3...", "d4e5f6..."] | [{id:"SKU-001", name:"Widget", price:99.99, qty:1}, {id:"SKU-002", name:"Gadget", price:50.00, qty:1}] |

### 2.2 Fully Denormalized (Single Wide Table)
Section 2.1 is recommended for complex nested record structures. Every child array is exploded into the parent row. One row per combination of array values. This layout is most practical when the primary source of row multiplication is user identity signals (multiple emails, phones, or UIDs) rather than cart items. For events with item arrays, Section 2.1 (nested structs) avoids combinatorial explosion.

Example: A generate_lead event where the advertiser has 2 email addresses and 2 phone numbers on file for the user. This produces 4 rows (2 emails × 2 phones).

| data_set_id | id | event_type | value | email_address | phone_number | customer_segment |
| --- | --- | --- | --- |--- | --- |
| acme-meta-001 | evt-4521 | generate_lead | 50.00 | a1b2c3... | 9f8e7d... | High Spender |
| acme-meta-001 | evt-4521 | generate_lead | 50.00 | a1b2c3... | 6c5b4a... | High Spender |
| acme-meta-001 | evt-4521 | generate_lead | 50.00 | d4e5f6... | 9f8e7d... | High Spender |
| acme-meta-001 | evt-4521 | generate_lead | 50.00 | d4e5f6... | 6c5b4a... | High Spender |

### 2.3 Partitioning

Parquet files SHOULD use Hive-style directory partitioning. The standard does not mandate a specific partition scheme -- advertisers choose based on their data volume and query patterns. 
Common schemes:

|Scheme | Directory Structure | When to Use |
| --- | --- | --- |
| Day Only | event_date=2026-05-14/ | Single campaign or low volume. Simplest. |
| Campaign + Day | campaign_id=camp-123/event_date=2026-05-14/ | Multi-campaign advertisers. Most common for DCR uploads -- isolates campaigns for independent attribution queries. |
| Data set + Day | data_set_id=acme-meta-001/event_date=2026-05-14/ | Multi-partner uploads where the same events go to multiple DCRs from a single upload pipeline. |

After all files are uploaded for a partition, the uploading party must write an empty marker file named done in the partition directory to signal that the partition is complete and ready for processing. Without this signal, a reader polling the directory cannot distinguish between a partition that is still being uploaded and one that is finished. Processing an incomplete partition produces incorrect results (missing events, wrong aggregates) with no indication of error. The done-marker pattern is an established convention across batch data pipelines: Hadoop MapReduce writes a _SUCCESS file after each job, Spark's FileOutputCommitter does the same, and the WFA cross-media-measurement project uses an identical done blob to gate processing of encrypted impression shards. DCRs MUST NOT process a partition until the done marker is present. If a partition must be re-uploaded (e.g., to correct data), the uploading party should delete the done marker, replace the files, and write a new done marker.

## 3. Platform-Specific Fields

ECAPI v1.0 includes an ext (extension) object on every object. These can be used for platform-specific data. In practice, every major ad platform has proprietary fields that advertisers must include for that platform's matching and attribution to work. Talk to your platform partner for any specific guidance on how to send these values.

## 4. Parquet File Metadata (Key-Value Pairs)
Parquet files have a footer that can store arbitrary key-value string pairs. This is the manifest mechanism -- no external manifest file is required. Every ECAPI DCR Parquet file MUST include these keys, prefixed with ecapi. to avoid collisions.

### 4.1 Required File Metadata
|Key | Example | Purpose |
| --- | --- | --- |
| ecapi.version | "1.0" | ECAPI schema version |
| ecapi.layout | "nested" | Which of the 3 layouts this file uses |
| ecapi.advertiser_id | “some-advertiser-id” | The id of advertiser in the DCR |
| ecapi.data_set_id | “some-data-set-id” | The id of the data set |
| ecapi.upload_id | "550e8400-e29b-..." | Unique batch upload ID (UUID v4) |
| ecapi.uploading_party | "acme-corp" | Who created this upload |
| ecapi.hash_encrypted | "sha256" | How the data was processed: "sha256", "encrypted", "both" |

### 4.2 Encryption Metadata (PME)

Section 1 requires that values designated for SHA-256 hashing by the ECAPI v1.0 standard be either hashed or encrypted before upload. When encryption is chosen, it should use Parquet Modular Encryption (PME) per the Apache Parquet format specification. PME encrypts
individual columns using AES and stores encryption metadata in the Parquet binary footer.

Required:
- Columns containing values that are encrypted rather than hashed must use PME.
- The Parquet footer should use plaintext footer mode so ingestion pipelines can read column names and row counts without decryption keys.

We strongly recommend encrypting rather than hashing when the DCR runs validation workflows in a Trusted Execution Environment (TEE) with KMS-based key wrapping (KEK/DEK model), where the TEE obtains decryption access via workload attestation. This ensures raw values are only accessible inside the attested enclave and never exposed to the DCR operator.

If encryption without a TEE is used, or if any non-attested workflow may access the decrypted data, values SHOULD also be SHA-256 hashed (i.e. ecapi.privacy_method = "both") so that raw values are never present even if encryption is bypassed or misconfigured.

Key management is deployment-specific and configured between the advertiser and their DCR operator. DCR operators must document their key management approach and provide advertisers with the PME writer configuration needed to produce compatible files.

The ecapi.privacy_method metadata key (Section 4.1) declares whether values were hashed ("sha256"), encrypted via PME ("encrypted"), or both ("both").

## ECAPI v1.0 DCR Extension: Parquet Schema — Option B (Nested Structs)
  
This document maps every field in the ECAPI v1.0 standard to a Parquet schema using Option B: Nested Structs (Dremel encoding). One Parquet row per event. One Parquet file per partition. Child collections (items, emails, addresses, uids) are represented as Parquet LIST of STRUCT, nested inside the single event row.

### Conventions
- REQUIRED/OPTIONAL follows the ECAPI v1.0 field requirements.
- Fields designated by ECAPI v1.0 as SHA-256 hashed are always STRING regardless of privacy method (hashed, PME-encrypted, or app-layer encrypted).
- All STRING fields use UTF-8 encoding.
- Parquet logical types are noted where they differ from the physical type.
- ext fields at each level are stored as a JSON-encoded STRING column.
- Column names match ECAPI v1.0 field names exactly.

### ECAPI to Parquet Type Mapping
ECAPI transmits over JSON, which represents all numbers as IEEE 754 double-precision. The following table shows where Parquet types differ from the ECAPI JSON types.

| ECAPI Type | Parquet Type | Fields | Notes |
| --- | --- | --- | --- |
| float | DOUBLE | value, price, discount, quantity, shipping, tax | JSON numbers are IEEE 754 double-precision; DOUBLE preserves the original precision | 
| enum (string values) | STRING | event_type, source, cattax | Parquet has no string-enum type; dictionary encoding compresses repeated values internally | 
| enum (integer values) | INT32 | age_range, address_type, atype, availability, body_style, condition_of_vehicle, drivetrain, fuel_type, listing_type, transmission | ECAPI defines these as integer-valued enumerations | 
| array | LIST | email_address, phone_numbers, customer_segments, gpp_sid, uids, address, coupon, payment_type, items | Parquet standard representation of arrays | 
| object | group (STRUCT) | user_data, properties | Parquet standard representation of nested objects | 

## Parquet Schema Definition (Dremel Notation)
The complete Parquet schema in Dremel notation. Everything is nested inside a single message — one row per event, one or more files per partition. Inline comments are drawn from the ECAPI v1.0 field descriptions.

```
message ecapi_event {

  -- core Event Object
  required binary data_set_id (STRING);           -- identifies data destination on receiving system
  optional binary id (STRING);                     -- unique event identifier chosen by advertiser
  required int64 timestamp;                        -- unix epoch seconds
  required binary event_type (STRING);             -- see event_type enumeration
  optional binary custom_event (STRING);           -- required when event_type="custom"
  optional double value;                           -- total value of event to advertiser
  optional binary currency_code (STRING);          -- ISO 4217; required when value is set
  optional binary source (STRING);                 -- where event took place; see source enumeration
  optional binary event_ext_json (STRING);         -- event-level exchange-specific extensions as JSON

  -- user_data Object
  optional group user_data {
    optional binary customer_identifier (STRING);  -- SHA-256 hashed customer identifier
    optional group customer_segments (LIST) {      -- general category, e.g. "Gold Member"
      repeated group list { required binary element (STRING); }
    }
    optional group email_address (LIST) {          -- SHA-256 hashed; lowercase, trim before hashing
      repeated group list { required binary element (STRING); }
    }
    optional group phone_numbers (LIST) {          -- SHA-256 hashed; remove symbols/letters/leading zeros, add +country code
      repeated group list { required binary element (STRING); }
    }
    optional int32 utcoffset;                      -- local time as +/- minutes from UTC
    optional binary gpp_string (STRING);           -- Global Privacy Platform consent string
    optional group gpp_sid (LIST) {                -- applicable GPP section IDs
      repeated group list { required int32 element; }
    }
    optional boolean mmt_only;                     -- if true, use for attribution only, not optimization
    optional binary click_id (STRING);             -- click ID for receiving partner/platform
    optional binary impression_id (STRING);        -- impression ID for receiving partner/platform
    optional binary event_ip_address (STRING);     -- IP address of the event; valid IPv4 or IPv6
    optional binary event_user_agent (STRING);     -- browser user agent for the event
    optional binary ifa (STRING);                  -- device identifier for advertising
    optional binary landing_ip_address (STRING);   -- IP address from campaign landing page click-through
    optional binary landing_user_agent (STRING);   -- user agent captured with landing_ip_address
    optional int32 age_range;                      -- age range enumeration (1-13); see ECAPI spec
    optional binary gender (STRING);               -- SHA-256 hashed; lowercase before hashing
    optional binary user_data_ext_json (STRING);   -- user-level exchange-specific extensions as JSON

    -- uids Object (array)
    optional group uids (LIST) {
      repeated group list {
        required group element {
          optional binary id (STRING);             -- SHA-256 hashed user identifier
          optional binary source (STRING);         -- canonical domain of the ID
          optional int32 atype;                    -- agent type per AdCOM 1.0
          optional binary uids_ext_json (STRING);  -- uid-level exchange-specific extensions as JSON
        }
      }
    }

    -- address Object (array)
    optional group address (LIST) {
      repeated group list {
        required group element {
          optional binary first_name (STRING);     -- SHA-256 hashed; lowercase, trim before hashing
          optional binary last_name (STRING);      -- SHA-256 hashed; lowercase, trim before hashing
          optional binary street (STRING);         -- SHA-256 hashed; lowercase, trim before hashing
          optional binary city (STRING);           -- lowercase, Roman alphabet, no punctuation
          optional binary state (STRING);          -- lowercase, Roman alphabet, no punctuation
          optional binary country_code (STRING);   -- 2-letter ISO 3166-1, lowercase
          optional binary postal_code (STRING);    -- SHA-256 hashed; 5-digit US zip or UK area/district/sector
          optional int32 address_type;             -- 1=billing, 2=shipping, 3=unknown
          optional binary address_ext_json (STRING); -- address-level exchange-specific extensions as JSON
        }
      }
    }
  }

  -- properties Object
  optional group properties {
    optional binary transaction_id (STRING);       -- identifier for a transaction
    optional binary page_url (STRING);             -- URL of the page
    optional binary ad_source (STRING);            -- utm_source parameter value
    optional binary referrer (STRING);             -- traffic referrer for the event
    optional group coupon (LIST) {                 -- coupon name/code
      repeated group list { required binary element (STRING); }
    }
    optional double shipping;                      -- shipping cost; currency from core event
    optional double tax;                           -- tax cost; currency from core event
    optional group payment_type (LIST) {           -- chosen payment method
      repeated group list { required binary element (STRING); }
    }
    optional binary shipping_tier (STRING);        -- shipping tier
    optional binary virtual_currency_name (STRING); -- virtual currency used for purchase
    optional binary virtual_item_name (STRING);    -- name of virtual item purchased
    optional binary lead_source (STRING);          -- source of the lead
    optional binary lead_status (STRING);          -- status of the lead
    optional binary lead_reason (STRING);          -- reason for the lead
    optional binary ad_platform (STRING);          -- ad platform
    optional binary ad_format (STRING);            -- ad format used
    optional binary ad_unit_name (STRING);         -- ad unit name
    optional binary login_method (STRING);         -- method used to login
    optional binary group_id (STRING);             -- ID of the group
    optional int32 character_level;                -- level of the character
    optional binary character (STRING);            -- character that leveled up
    optional int32 post_score;                     -- score to be posted
    optional binary achievement_id (STRING);       -- ID of an achievement
    optional binary search_term (STRING);          -- term used for the search
    optional binary creative_name (STRING);        -- name of promotional creative
    optional binary creative_slot (STRING);        -- promotional creative slot
    optional binary promotion_id (STRING);         -- promotion ID
    optional binary promotion_name (STRING);       -- name identifying a promotion or campaign
    optional int32 availability;                   -- enumeration (1-6); see ECAPI spec
    optional int32 body_style;                     -- vehicle body style enumeration (1-11)
    optional int32 condition_of_vehicle;           -- 1=new, 2=used
    optional binary arrival_date (STRING);         -- YYYYMMDD or YYYY-MM-DD
    optional binary departure_date (STRING);       -- YYYYMMDD or YYYY-MM-DD
    optional binary destination_airport (STRING);  -- IATA airport code
    optional binary destination_ids (STRING);      -- destination catalog IDs
    optional int32 drivetrain;                     -- vehicle drivetrain enumeration (1-7)
    optional binary exterior_color (STRING);       -- exterior color
    optional int32 fuel_type;                      -- vehicle fuel type enumeration (1-9)
    optional binary lease_end_date (STRING);       -- YYYYMMDD or YYYY-MM-DD
    optional binary lease_start_date (STRING);     -- YYYYMMDD or YYYY-MM-DD
    optional int32 listing_type;                   -- listing type enumeration (1-7)
    optional binary make (STRING);                 -- vehicle make or brand
    optional binary model (STRING);                -- vehicle model
    optional int32 transmission;                   -- vehicle transmission enumeration (1-4)
    optional binary vin (STRING);                  -- Vehicle Identification Number
    optional binary properties_ext_json (STRING);  -- properties-level exchange-specific extensions as JSON

    -- item Object (array)
    optional group items (LIST) {
      repeated group list {
        required group element {
          optional binary id (STRING);             -- UPC or SKU
          optional binary name (STRING);           -- product name
          optional double price;                   -- product price; currency from core event
          optional double discount;                -- unit monetary discount; currency from core event
          optional double quantity;                -- quantity of the item
          optional binary brand (STRING);          -- brand of the item
          optional binary affiliation (STRING);    -- supplying company or store location
          optional binary category (STRING);       -- product category; taxonomy declared in cattax
          optional binary cattax (STRING);         -- category taxonomy per AdCOM 1.0
          optional binary item_coupon (STRING);    -- coupon name/code for the item
          optional binary item_list_id (STRING);   -- ID of the list the item was presented in
          optional binary item_list_name (STRING); -- name of the list the item was presented in
          optional binary item_variant (STRING); -- item variant or additional details/options
          optional binary item_location_id (STRING); -- physical location associated with the item
          optional binary item_ext_json (STRING);  -- item-level exchange-specific extensions as JSON
        }
      }
    }
  }
}
```
