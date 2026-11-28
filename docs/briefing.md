# Briefing: OFF Engine (`off-engine`, OFE)

**Status:** draft for `/init` · **Specification:** [Open Fitness Framework (OFF) 0.3 draft](https://github.com/open-fitness-framework/off-specification/blob/main/spec/0.3/open-fitness-framework-spec.md) · **Builds on:** [`off-core`](https://github.com/open-fitness-framework/off-core)

## 1 Context

The Open Fitness Framework (OFF) separates reusable evidence about data assets from the purposes it is evaluated against. Evidence takes the form of *fitness reports*: VC 2.0 credentials, secured as ES256 JWS, containing test and measurement records about one immutable snapshot of a data asset.

The reference implementation has three components on top of the shared library `off-core`:

| Component | Spec roles | Purpose |
| --- | --- | --- |
| **OFF Engine (OFE)**, this project | Assessor, report issuer; snapshot binding | Runs fitness checks on a snapshot and issues signed reports. |
| OFF Catalogue (OFC) | Repository, discovery, profile and definition publication | Stores and serves reports, profiles, definitions and other pinned artifacts. |
| OFF Evaluator (OVA) | Evaluator, trust policy store, request building | Produces fitness decisions. |

The OFE produces evidence; it never decides fitness. It does not read thresholds, and a measurement carries a value, not a verdict (Section 3).

## 2 Users

- **Data publishers** who want to attach verifiable quality evidence to their releases.
- **Assessment providers** (validation services, QA bots, auditors) who run checks for others and sign the results.
- **Consumers and agents** who order an assessment because the evidence they need is missing.
- **Consumers** who need to check that downloaded data matches an evaluated snapshot binding (`verify-binding`).

## 3 Responsibilities and scope of the first version

The first version implements the OFF core and no extensions. Section numbers refer to the specification.

### 3.1 Snapshot binding (Section 4.4)

- **`bind`** produces an asset binding for a data snapshot:
  - for a single file, a `content` digest;
  - for a directory or collection, a `manifest` digest.
- The OFE defines one **manifest format** and publishes it as a pinned method artifact, as Section 4.4 requires the method to define the manifest. The format is a JSON document that lists every constituent file with its relative path, size and SHA-256 digest. It is sorted and written as deterministic bytes, and the binding digest is the SHA-256 of those bytes.
- **`verify-binding`** checks a local copy of the data against a binding. Consumers use it before relying on an evaluated snapshot ("Matching two metadata strings alone does not establish the identity of subsequently downloaded bytes", 4.4).
- Asset, representation and snapshot ids are supplied by the caller. The OFE never invents mutable labels such as "latest".

### 3.2 Assessment orders

- **An order consists of:**
  - the snapshot (data location plus binding, or the data and ids from which to compute a binding);
  - the statements to assess: each a definition (`ArtifactRef`) plus parameters;
  - the assessment method and, optionally, a configuration.
- **Statements from a profile:** an order may name a profile instead of listing statements. The OFE then copies the profile's statements verbatim and ignores its conditions (5.2, informative note). Records then match the profile exactly.
- **Output:** one signed report per order. All its records share subject, scope, coverage and assessment (6.2).
- **Interfaces:** orders are accepted on the CLI (`assess`) and through the optional HTTP job API ([Section 3.7](#37-interfaces)).

### 3.3 Check plug-ins

- **Two kinds of check implement definitions:**
  - in-process **Java plug-ins**;
  - **external check programs** in any language, called through a small, versioned JSON protocol over stdin and stdout (or files). This lets checks built on Python or native tools such as GDAL, rasterio or the CDISC CORE engine be added without changing the OFE.
- **A check returns only the assessment result:**
  - for a test: the EARL outcome;
  - for a measurement: the execution status, value and details;
  - plus evidence files and limitations.
- **The OFE builds the records**, so every check produces valid output:
  1. it validates each result against the definition with `off-core` (type, unit, range, required details, the 6.3 forms);
  2. a check that crashes or times out becomes `earl:cantTell` with EXECUTION_ERROR, or a measurement with `executionStatus` `error` and no value, never `earl:failed` (6.3);
  3. invalid check output is never signed.
- **A registry** maps pinned definitions to check implementations. A statement whose definition has no check is reported as not executed (`earl:untested` or `notExecuted`), never silently dropped.

### 3.4 First checks

The first version implements the two checks of the specification's illustrative example (Section 15.2, Appendix A):

- **Schema conformity** (test, pattern 7.2): every record of a tabular or feature dataset validates against a pinned JSON Schema. The definition states how warnings, unsupported features and validator failures are handled.
- **Attribute completeness** (measurement, pattern 7.3): the share of eligible records with a present, non-null value of a given attribute. It is a decimal, unitless and truncated downward to 12 decimal places; a zero denominator is inconclusive. The required details are the counts.

Input formats are CSV and GeoPackage feature tables, which are read in Java through SQLite JDBC.

The specification's example definitions do not yet exist as documents. This project writes both as concrete `AssertionDefinition` documents following Section 7.1 and publishes them to the OFC. It also ships one small **reference external check**, a Python implementation of attribute completeness, so that the external protocol is tested end to end.

### 3.5 Evidence

- **Evidence items:** checks may produce supporting files such as validator logs and result listings. The OFE turns them into evidence items with digest, media type and role (6.4).
- **Retrieval locations:** `href` values point to where the OFE deployment publishes the files: the OFC artifact store or the deployment's own HTTPS location.
- **No secrets in reports:** no URL credentials or expiring tokens (6.4).
- **Data protection:** the OFE must not put personal data into evidence or record details on its own initiative. Each check decides what evidence it emits (Section 13).

### 3.6 Signing and issuer identity (Sections 6.1, 8.1)

- **Signing:** the OFE signs reports with `off-core`'s payload builder and `Signer` interface. Deployments configure one issuer identity (an HTTPS URI) and its signing key.
- **Keys:** the first version uses software keys from a protected file or secret store. KMS and HSM signers are later `Signer` implementations.
- **JWKS:** the OFE serves a **JWKS document** at the issuer identity, listing current and previous public keys with their `kid`. It also offers key rotation (new `kid`, old key kept for verification). Policy owners copy keys into their trust policies after checking them out of band. The JWKS is not trust.
- **Report ids:** every report gets a unique HTTPS id. A correction is a new report with a new id and may name the earlier one in `corrects` (10.2).
- **Validity:** `validFrom`, `validUntil` and `issuedAt` follow deployment configuration and the timestamp rules (6.1, 6.2).

### 3.7 Interfaces

- **CLI:** `bind`, `verify-binding`, `assess`, `keys` (generate, rotate, export JWKS), `publish`.
- **Optional HTTP service:**
  - a job API (submit an order, poll its status, fetch the report);
  - the issuer's JWKS endpoint;
  - evidence hosting when no OFC is used.
  - Authenticated clients only; the JWKS and evidence are public.
- **Publishing:** reports and evidence can optionally be pushed to an OFC, as an authenticated publisher, unchanged.
- **Delivery:** as a container image and a runnable JAR.

## 4 Out of scope for the first version

- Extensions E1 to E5. Sampled coverage, selection scopes and credential status (status lists and revocation) follow later; E4 is the first priority.
- Deciding fitness, reading thresholds and comparing results with conditions.
- Domain checks beyond the two above: INSPIRE topology, raster coverage and SDTM conformance follow as plug-ins.
- Hosting or processing the data assets themselves, beyond reading a snapshot for assessment.
- Discovery of assets or profiles.

## 5 Constraints

- **Language:** Java, the same version and build tool as `off-core` (to be confirmed at `/init`). External checks may use any language.
- **Sandboxing:** external checks run as separate processes with time and resource limits and no network access by default. Downloaded definitions never execute code (7.1).
- **Data protection:** data snapshots may contain personal data. The OFE processes them locally, keeps no copies beyond the job's lifetime, and logs no record contents.
- **Factory runtime:** the Software Factory image ships Node and Python only, so a `.factory/Dockerfile` with a JDK is needed. Python is available for external checks.

## 6 Success criteria

- `bind` followed by `assess` with the Appendix A profile on a synthetic 1000-building dataset (987 with height) produces a report whose records match the profile exactly: schema `earl:passed`, completeness 0.987.
- The report verifies with `off-core` and with the `off-conformance` issuer checks.
- `off-core`'s evaluator returns FIT for that report with threshold 0.98 and NOT_FIT with 0.995 (Section 15.2).
- A crashing check produces `earl:cantTell` with EXECUTION_ERROR, never a failure.
- The reference external check gives the same value as the Java plug-in.
- `verify-binding` detects any changed, added or removed file in a manifest snapshot.

## 7 Candidate feature map

| # | Feature | Depends on |
| --- | --- | --- |
| 1 | Snapshot binding: manifest format, `bind`, `verify-binding` | off-core |
| 2 | Issuer configuration, software keys, JWKS export and rotation | off-core |
| 3 | Check SPI, definition registry and in-process plug-ins | off-core |
| 4 | Assessment orders and report building and signing (`assess`) | 1, 2, 3 |
| 5 | Schema conformity and attribute completeness checks with their definitions | 3 |
| 6 | External check protocol and the Python reference check | 3, 4 |
| 7 | Evidence handling and publishing to the OFC | 4 |
| 8 | HTTP service: job API, JWKS endpoint, evidence hosting | 2, 4, 7 |

## 8 Open questions for `/init`

- The exact JSON shape of the external check protocol and of the manifest format. Both are versioned artifacts of this project.
- The job store for the HTTP service: in-memory, or a database.
- License, as for the other OFF repositories.
