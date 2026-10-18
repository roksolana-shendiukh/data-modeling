# Conformed and Role-Playing Dimensions

A warehouse is defined by its shared dimensions. Fact tables are specific to a business process, while dimensions are shared across processes, so any facts joined through them give consistent results.

## Conformed dimensions

- A conformed dimension has one definition and one surrogate key, and is used by every fact table that needs it.
- Attributes and the SCD strategy are defined once, so counts and time periods match across teams.
- Only attributes shared by all domains belong in the conformed core. Domain-specific attributes go into extension tables or outriggers.
- Business rules are kept in fact and metrics logic, not in the dimension.
- Source systems often identify the same entity differently. These identifiers are resolved in ETL through a crosswalk, so facts store only the surrogate key.

## Role-playing dimensions

- One physical dimension is referenced several times from the same fact, once per role.
- Each role is a separate foreign key to the same table, with no copies of the dimension.
- Roles are exposed through query aliases or views for readability and BI discoverability.
- The pattern applies to any dimension, not only dates.

## Mini-dimensions and outriggers

- A mini-dimension holds volatile attributes extracted from the main dimension. It has its own surrogate key, and its size is bounded by the number of distinct combinations.
- The fact carries a key to both dimensions, so the attribute values at event time are preserved and the main dimension avoids SCD explosion.
- An outrigger is a dimension attached to another dimension. It reduces denormalization at the cost of an extra join.

## Governance

- **Ownership:** a single team owns each conformed dimension and approves changes.
- **Versioning:** new attributes are added as nullable columns, and renames follow a deprecation plan.
- **Data contracts:** fields, types and SLAs are agreed between source teams and the dimension owner.
- **Metrics layer:** each metric is defined once and reused by all consumers.

