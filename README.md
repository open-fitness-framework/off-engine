# off-engine

The OFF Engine (OFE) is the evidence producer of the [Open Fitness Framework (OFF)](https://github.com/open-fitness-framework/off-specification) reference implementation. It takes the assessor and report issuer roles of the specification. It is built on the shared library [off-core](https://github.com/open-fitness-framework/off-core).

The OFE:

- **binds data asset snapshots:** it computes content or manifest digests (`bind`), and lets consumers check downloaded data against an evaluated binding (`verify-binding`);
- **runs fitness checks** for the statements in an assessment order, usually copied verbatim from a fitness profile. Checks are in-process Java plug-ins or external programs in any language, called through a JSON protocol;
- **issues signed fitness reports:** it builds and validates the test and measurement records and signs them as VC 2.0 credentials (ES256 JWS). A crashing check is reported as inconclusive, never as a failure;
- **publishes its issuer keys** as a JWKS document, and optionally publishes reports and evidence to the [off-catalogue](https://github.com/open-fitness-framework/off-catalogue).

The OFE produces evidence only. It never reads thresholds or decides fitness; that is the job of the [off-evaluator](https://github.com/open-fitness-framework/off-evaluator).

The first version covers the OFF core. It ships schema-conformity and attribute-completeness checks for CSV and GeoPackage data, a CLI and an optional HTTP job API.

**Status:** not yet implemented. See the [briefing](docs/briefing.md) for the planned scope.
