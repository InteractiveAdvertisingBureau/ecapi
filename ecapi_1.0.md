# About the IAB Tech Lab <a name="techlab"></a>

The IAB Technology Laboratory is a nonprofit research and development consortium charged with producing and helping companies implement global industry technical standards and solutions. The goal of the Tech Lab is to reduce friction associated with the digital advertising and marketing supply chain while contributing to the safe growth of an industry. The IAB Tech Lab spearheads the development of technical standards, creates and maintains a code library to assist in rapid, cost-effective implementation of IAB standards, and establishes a test platform for companies to evaluate the compatibility of their technology solutions with IAB standards, which for 18 years have been the foundation for interoperability and profitable growth in the digital advertising supply chain. Further details about the IAB Technology Lab can be found at www.iabtechlab.com

### Significant Contributors to the ECAPI version specification

Steven Ware Jones, Meta, Sean Bedford, Meta, Randy Weinstein, Basis Technology, Alan Merzon, Google, Chandan Giri, Google, Akchat Jha, Walmart, Barbara Kalicki, Publicis Sapient, Melissa DeLuca, NBC Universal, Celina Mbale, Paramount, Steven Huang, Bytedance, Matt Reid, Roku, Brian May

### IAB Tech Lab Lead:

Jill Wittkopp, VP Product, IAB Tech Lab

### License <a name="license"></a>

This specification is licensed under a Creative Commons Attribution 3.0 License. To view a copy of this license, visit creativecommons.org/licenses/by/3.0/ http://creativecommons.org/licenses/by/3.0/ or write to Creative Commons, 171 Second Street, Suite 300, San Francisco, CA 94105, USA.

### Disclaimer <a name="disclaimer"></a>

THE STANDARDS, THE SPECIFICATIONS, THE MEASUREMENT GUIDELINES, AND ANY OTHER MATERIALS OR SERVICES PROVIDED TO OR USED BY YOU HEREUNDER (THE “PRODUCTS AND SERVICES”) ARE PROVIDED “AS IS” AND “AS AVAILABLE,” AND IAB TECHNOLOGY LABORATORY, INC. (“TECH LAB”) MAKES NO WARRANTY WITH RESPECT TO THE SAME AND HEREBY DISCLAIMS ANY AND ALL EXPRESS, IMPLIED, OR STATUTORY WARRANTIES, INCLUDING, WITHOUT LIMITATION, ANY WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, AVAILABILITY, ERROR-FREE OR UNINTERRUPTED OPERATION, AND ANY WARRANTIES ARISING FROM A COURSE OF DEALING, COURSE OF PERFORMANCE, OR USAGE OF TRADE. TO THE EXTENT THAT TECH LAB MAY NOT AS A MATTER OF APPLICABLE LAW DISCLAIM ANY IMPLIED WARRANTY, THE SCOPE AND DURATION OF SUCH WARRANTY WILL BE THE MINIMUM PERMITTED UNDER SUCH LAW. THE PRODUCTS AND SERVICES DO NOT CONSTITUTE BUSINESS OR LEGAL ADVICE. TECH LAB DOES NOT WARRANT THAT THE PRODUCTS AND SERVICES PROVIDED TO OR USED BY YOU HEREUNDER SHALL CAUSE YOU AND/OR YOUR PRODUCTS OR SERVICES TO BE IN COMPLIANCE WITH ANY APPLICABLE LAWS, REGULATIONS, OR SELF-REGULATORY FRAMEWORKS, AND YOU ARE SOLELY RESPONSIBLE FOR COMPLIANCE WITH THE SAME, INCLUDING, BUT NOT LIMITED TO, DATA PROTECTION LAWS, SUCH AS THE PERSONAL INFORMATION PROTECTION AND ELECTRONIC DOCUMENTS ACT (CANADA), THE DATA PROTECTION DIRECTIVE (EU), THE E-PRIVACY DIRECTIVE (EU), THE GENERAL DATA PROTECTION REGULATION (EU), AND THE E-PRIVACY REGULATION (EU) AS AND WHEN THEY BECOME EFFECTIVE.

# TABLE OF CONTENTS

- [About the IAB Tech Lab](#techlab)

- [License](#license)

- [Disclaimer](#disclaimer)

## [Overview](#overview)

## [How It Works](#howitworks)

## [API Conventions](#apiconventions)

  - [Normalization](#normalization)
  
  - [Versioning](#versioning)

  - [Customized Extensions](#customizedextensions)

## [Using Events API](#usingeventsapi)

  - [Sending Events](#sendingevents)
  
  - [ID Based Records Processing](#idbasedrecords)

  - [Individual vs Batch Events](#individualvsbatch)

## [Event Payload](#eventpayload)

  - [core Event Object](#coreeventobject)
  
  - [event_type(s) Enumeration](#eventtypes)
      
      - [Standard Events](#standardevents)
    
      - [Additional Events](#additionalevents)
  
  - [user_data Object](#user_data)

  - [uids Object](#uids)
    
  - [address Object](#address)
  
  - [item Object](#item)

  - [age_range Enumeration](#age_range)

  - [source Enumeration](#source)

  - [properties Object](#properties)

## [Verifying the Integration](#verifying)

  - [Response Codes](#responsecodes)


## Overview <a name="overview"></a>

The Event & Conversion API (commonly known as CAPI) is designed to standardize communication of marketing-related events from advertisers’ systems to advertising platforms and partners. The recipients are able to leverage this information to optimize marketing campaigns toward desired outcomes and to better measure and report on marketing campaign performance across complete marketing funnels.

## How It Works <a name="howitworks"></a>

The Event & Conversion API defines a set of full-funnel events that are of interest to advertisers. By providing these events to partners, advertisers enable them to:
- Optimize ad campaigns toward desired outcomes
- Measure campaign performance throughout the marketing funnel

These capabilities enable advertising platforms and partners to improve campaign ROI through more effective targeting and better insight into what strategies lead to high value events.

## API Conventions <a name="apiconventions"></a>

This API adheres to many of the conventions of RESTful APIs. The base protocol assumes communication is HTTPS, and JSON is used to represent the body of requests and responses. Requests should be made with HTTP headers of "Accept: application/json" and "Content-Type: application/json" to indicate that the body of the request will be JSON and that JSON is expected in return. Individual advertising platforms and partners will have specific requirements for advertising partner integrations and implementations should start with a review of their guidance. 

While most of the fields are designated by the standard as optional, there are fields which are defined as required and conditionally required. Pay close attention to these fields during integration. Additionally, partners and platforms may require that certain fields be present for various business use cases. When this is the case, it is expected that they will communicate to partners these minimum requirements. Receivers should be aware that they may encounter unexpected values for fields that aren’t strictly defined, that optional fields may be populated inconsistently, and that extension (ext) fields may be added, removed or otherwise altered.

### Normalization <a name="normalization"></a>

This standard includes a number of fields which are SHA256 hashed. In order to maintain consistency across implementations and ensure equivalent inputs produce the same hash value, inputs must be normalized in the same way by all participants prior to hashing. The standard recommends that values consist of lowercase Roman alphabet characters (a-z). Whitespace should be trimmed. There should be no irrelevant formatting and no punctuation. In cases where special characters are required, the text must be encoded in UTF-8 format.

### Versioning <a name="versioning"></a>

When breaking changes are made to this specification, they will be indicated by incrementing the major version number (1.x, 2.x, etc.).

### Customized Extensions <a name="customizedextensions"></a>

This API specification allows for implementation-specific customization through extension (ext) objects which are included as the last field in every object definition in the standard. It is assumed that advertisers will work directly with partners and platforms to define and implement support for extensions as needed.

## Using Events API <a name="usingeventsapi"></a>

This API is intended for server to server transmission from advertisers to their advertising partners/platforms to support measurement and optimization use cases.

### Sending Events <a name="sendingevents"></a>

Advertisers will communicate events to partners through server-to-server calls to partner API endpoints. To send new events, the advertiser should make a POST request to the chosen partner’s API endpoint. Requirements for accessing APIs will be partner specific and advertisers should work with each partner when implementing support for the relationship.

Advertising partners/platforms may support incoming webhooks or other modes of communication through their API endpoints for events and advertisers are urged to discuss available options with their integration partners.

### ID Based Records Processing <a name="idbasedrecords"></a>

Receiving systems will assume that every unique event has a unique combination of data_set_id and event ID (see the Core Event Object, below for definitions). Event ID is expected to be the same for all records related to a specific event and if no ID is present, records will be assumed to be for unique events unless the recipient supports other ways of matching records (see below). If an advertiser is generating records for the same event via multiple channels (e.g., both a server-side event is generated and a pixel fires when an item is carted) and they want the records to be merged, both the server-side event and pixel fire  should include the same event ID. If it is not possible to provide the same ID for the event across multiple channels, advertisers should only submit a given event from a single channel to avoid creating multiple records for the same event. Some advertising platforms and partners may have the capability to match records based on other fields if no ID is provided; when working with them, IDs should not be included for channels producing records that should be merged. 

When platforms receive an event record with an ID, they are expected to look up the associated data_set_id + ID to determine if the record is for a new or existing event. If the data_set_id + ID is not found, a new record should be added to the data store. If the data_set_id + ID is found, the records should be merged. Merge rules are implementation specific. Please see partner-specific documentation for further details.

When platforms receive an event record with no ID, they may or may not attempt to determine if a record is for a new or existing event by using other fields. If they don’t use alternative matching methods, multiple records for the same event will be created if records are received from multiple channels. If they do use alternate methods, records will be processed as if they had an ID: if no match is found, a new record should be added and if a match is found, the records should be merged.

Recipients may impose record update time limits and discard any new records with an ID that matches that of an existing record after the time limit has been exceeded.

### Individual vs Batch Events <a name="individualvsbatch"></a>

Advertisers may choose to send events individually as they occur or in batches, assuming the partners/platforms they are working with support batch processing. While capturing events as close to the time of occurrence as possible generally provides the most value, there are use cases for which batching events makes more sense; examples include offline events from a retail store and lead qualification events from a CRM. 

Advertising partners/platforms that support batch processing will have requirements for how data is packaged and delivered and advertisers are expected to consult with them to determine how to submit batch data. 

## Event Payload <a name="eventpayload"></a>

### core Event Object <a name="coreeventobject"></a>

These are the core attributes necessary to operationalize data received through the Event & Conversion API.

| Attribute | Type | Description |
| --- | --- | --- |
| `data_set_id` | string, required | An identifier, coordinated by both parties, that indicates the destination for the data on the receiving system, this may be an advertiser or account related id. Note that this field is used to separate ID spaces: records with the same ID and data_set_id are assumed to refer to a single event, while records with records with the same ID and different data_set_ids are assumed to refer to unique, unrelated events. If this field is missing or invalid, the record may be discarded. |
| `id` | string, strongly recommended | This ID can be any unique string chosen by the advertiser. If records for the same event are sent from multiple sources (e.g. a pixel and CAPI), they should include the same event_id; see ID-Based Record Process, above for why. For events without an intrinsic ID number, a random number (so long as the same random number is sent between browser, server and other event sources) can be used. If the same ID is not guaranteed to be provided for the same event by different sources, it is preferred that this field be omitted. |
| `timestamp` | integer, required | The unix epoch timestamp of the event. For example,1746558464 (May 06 2025 19:07:44 GMT+0000). If this field is missing or invalid, the record may be discarded. |
| `event_type` | enum, required | The type of the  event; see [event_type list](#eventtypes). If event_type is “custom”, the custom_event field (below) must be populated. If this field is missing or invalid, the record may be discarded. |
| `custom_event` | string, optional | Required if the enum of event_type is custom. If this field is populated and event_type is not “custom”, it will be ignored. If the event_type field (above) is “custom” and this field is missing or invalid, the record may be discarded. |
| `user_data` | object | The object specifies information about the user associated with the event. |
| `value` | float, recommended | The total value of the event to the advertiser. This could be the total price of the order or the advertiser's determined value of a specific event to their business. If this field value is set, the currency_code (below) is required. |
| `currency_code` | string, recommended | The ISO 4217 currency code. This field is required when “value” (above) is set. The currency reported here applies to all monetary fields. If value is set and this field is missing or invalid, the record may be discarded. |
| `source` | enum, recommended | This field specifies where the event took place. See [source](#source) list. |
| `properties` | object | This object specifies additional properties associated with the event. |
| `ext` | object | Placeholder for exchange-specific extensions to OpenRTB. |

### event_type(s) Enumeration <a name="eventtypes"></a>

There are many ways in which businesses describe the events of importance to them. For this standard, we sought to support the most common use cases in the ‘standard events’ table. We also included an Additional Events table, which identifies less common types that were still  deemed worthy of standardization, but lower priority for overall support.
  
#### Standard Events <a name="standardevents"></a>

These are the events advertisers most commonly want to share with a partner or platform.

| Event Name | Event Description |
| --- | --- |
| `purchase` | A purchase. |
| `page_view` | A page on the advertiser website was viewed. |
| `ad_impression` | A user was exposed to an ad impression. |
| `add_to_wishlist` | An item was added to a wishlist. |
| `add_to_cart` | An item was added to a cart for purchase. |
| `viewed_cart` | A purchase cart was viewed. |
| `viewed_item` | An item or product page was viewed. |
| `begin_checkout` | A user has begun a checkout flow. |
| `add_payment_info` | A user submitted their payment information in a checkout flow. |
| `remove_from_cart` | An item was removed from a cart. |
| `refund` | One or more items was refunded to a user. |
| `generate_lead` | A lead was generated. |
| `qualify_lead` | A user is identified as a qualified lead. |
| `close_convert_lead` | A lead has been converted and closed. |
| `disqualify_lead` | A lead has become disqualified. |
| `close_unconvert_lead` | A user is identified as failing to become a converted lead. |
| `sign_up` | A user signed up for an account or offer. |
| `search` | A user has performed a search. |
| `unlock_achievement` | A user unlocked an achievement. |
| `install` | An app install. |
| `customize_product` | A user customized a product. |
| `contact` | A user contact. |
| `donate` | A donation was made. |
| `find_location` | A user searches for a location. |
| `schedule` | A user schedules an appointment. |
| `start_trial` | A user begins a trial. |
| `subscribe` | A user subscribes to a product or service. |
| `custom` | Custom event identified in the `custom_event` field in the [core Event Object](#coreeventobject). When this event type is indicated, the `custom_event` field must contain a value or the record is invalid. |

#### Additional Events <a name="additionalevents"></a>

While still common, these are generally deemed a lower priority of events that advertisers share with a partner/platform.

| Event Name | Event Description |
| --- | --- |
| `add_shipping_info` | A user submitted shipping information. |
| `share` | A user shared content. |
| `select_content` | A user selected content. |
| `select_item` | A user selected an item from a list. |
| `select_promotion` | A user selected a promotion from a list. |
| `view_item_list` | A user viewed a list of items. |
| `view_promotion` | A user viewed a promotion. |
| `view_search_results` | A user was presented with the results of a search. |
| `spend_virtual_currency` | A user spent virtual currency. |
| `earn_virtual_currency` | A user earned virtual currency. |
| `working_lead` | A user contacted, or was contacted by, a representative. |
| `login` | A user logged in to a website or app. |
| `join_group` | A user joined a group. |
| `level_up` | A user leveled up in a game. |
| `post_score` | A user posted a score. |
| `tutorial_begin` | A user started an on-boarding process. |
| `tutorial_complete` | A user completed an on-boarding process. |
      
### user_data Object <a name="user_data"></a>

### uids Object <a name="uids"></a>

### address Object <a name="address"></a>

### item Object <a name="item"></a>

### age_range Enumeration <a name="age_range"></a>

### source Enumeration <a name="source"></a>

### properties Object <a name="properties"></a>

## Verifying the Integration <a name="verifying"></a>

### Response Codes <a name="responsecodes"></a>
