# Amazon SimpleDB (amazon-simpledb)

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Amazon SimpleDB is a highly available NoSQL data store that offloads the work of database administration for web applications.

**URL:** [https://aws.amazon.com/simpledb/](https://aws.amazon.com/simpledb/)

**Run:** [Capabilities Using Naftiko](https://github.com/naftiko/fleet?utm_source=api-evangelist&utm_medium=readme&utm_campaign=company-api-evangelist&utm_content=repo)

## Tags:

 - AWS, Cloud Storage, Data Storage, Database, NoSQL

## Timestamps

- **Created:** 2026-03-16
- **Modified:** 2026-04-19

## APIs

### Amazon SimpleDB API

Amazon SimpleDB is a highly available NoSQL data store that offloads the work of database administration for web applications.

**Human URL:** [https://aws.amazon.com/simpledb/](https://aws.amazon.com/simpledb/)

#### Tags:

 - Data Storage, Database, NoSQL

#### Properties

- [Documentation](https://docs.aws.amazon.com/AmazonSimpleDB/latest/DeveloperGuide/Welcome.html)
- [OpenAPI](openapi/amazon-simpledb.yaml)
- [GettingStarted](https://aws.amazon.com/simpledb/)
- [Pricing](https://aws.amazon.com/simpledb/pricing/)
- [FAQ](https://aws.amazon.com/simpledb/faqs/)

## Common Properties

- [Portal](https://aws.amazon.com/simpledb/)
- [TermsOfService](https://aws.amazon.com/service-terms/)
- [PrivacyPolicy](https://aws.amazon.com/privacy/)
- [GitHubOrganization](https://github.com/aws)
- [StatusPage](https://health.aws.amazon.com/health/status)

## Features

| Name | Description |
|------|-------------|
| Simple Data Storage | NoSQL data store with no database administration overhead. |
| Select Query Language | Query data using a simple SELECT expression language. |

## Use Cases

| Name | Description |
|------|-------------|
| Web Application Data Storage | Store and query structured data for web applications. |

## Integrations

| Name | Description |
|------|-------------|
| Amazon S3 | Store metadata for S3 objects in SimpleDB. |

## Artifacts

Machine-readable API specifications organized by format.

### OpenAPI

- [amazon-simpledb.yaml](openapi/amazon-simpledb.yaml)

### JSON Schema

- [amazon-simpledb-attribute-does-not-exist-schema.json](json-schema/amazon-simpledb-attribute-does-not-exist-schema.json)
- [amazon-simpledb-attribute-schema.json](json-schema/amazon-simpledb-attribute-schema.json)
- [amazon-simpledb-batch-delete-attributes-request-schema.json](json-schema/amazon-simpledb-batch-delete-attributes-request-schema.json)
- [amazon-simpledb-batch-put-attributes-request-schema.json](json-schema/amazon-simpledb-batch-put-attributes-request-schema.json)
- [amazon-simpledb-create-domain-request-schema.json](json-schema/amazon-simpledb-create-domain-request-schema.json)
- ... and 33 more

### JSON Structure

- [amazon-simpledb-attribute-does-not-exist-structure.json](json-structure/amazon-simpledb-attribute-does-not-exist-structure.json)
- [amazon-simpledb-attribute-structure.json](json-structure/amazon-simpledb-attribute-structure.json)
- [amazon-simpledb-batch-delete-attributes-request-structure.json](json-structure/amazon-simpledb-batch-delete-attributes-request-structure.json)
- [amazon-simpledb-batch-put-attributes-request-structure.json](json-structure/amazon-simpledb-batch-put-attributes-request-structure.json)
- [amazon-simpledb-create-domain-request-structure.json](json-structure/amazon-simpledb-create-domain-request-structure.json)
- ... and 33 more

### JSON-LD

- [amazon-simpledb-context.jsonld](json-ld/amazon-simpledb-context.jsonld)

### Examples

- [amazon-simpledb-attribute-does-not-exist-example.json](examples/amazon-simpledb-attribute-does-not-exist-example.json)
- [amazon-simpledb-attribute-example.json](examples/amazon-simpledb-attribute-example.json)
- [amazon-simpledb-batch-delete-attributes-request-example.json](examples/amazon-simpledb-batch-delete-attributes-request-example.json)
- [amazon-simpledb-batch-put-attributes-request-example.json](examples/amazon-simpledb-batch-put-attributes-request-example.json)
- [amazon-simpledb-create-domain-request-example.json](examples/amazon-simpledb-create-domain-request-example.json)
- ... and 33 more

## Capabilities

Naftiko capabilities organized as shared per-API definitions.

### Shared Per-API Definitions

- [amazon-simpledb.yaml](capabilities/shared/amazon-simpledb.yaml) — Amazon SimpleDB operations for resource management

## Vocabulary

- [Amazon SimpleDB Vocabulary](vocabulary/amazon-simpledb-vocabulary.yaml) — Unified taxonomy mapping resources, actions, workflows, and personas

## Rules

- [Amazon SimpleDB Spectral Rules](rules/amazon-simpledb-spectral-rules.yml) — Rules enforcing Amazon SimpleDB API conventions

## Maintainers

**FN:** Kin Lane

**Email:** kin@apievangelist.com
