---
name: analyze-opportunity
description: Investigates a land parcel and its adjacent neighboring parcels for real estate investment opportunity, core cost-per-acre valuation, and hydrological risk analysis.
metadata:
  adk_additional_tools:
    - get_lot_data
    - get_tax_assessment_data
---
# Analyze Opportunity Skill

When asked to investigate a property or analyze an investment opportunity for a lot, you MUST follow all steps below thoroughly:

1. **Retrieve Subject Parcel Data**:
   - Use the `get_lot_data` tool to lookup the subject property attributes by its lot ID (e.g., `LOT-8472`).
   - Review all returned attributes, including `acreage`, `zoning`, `last_sale_price`, `flood_zone`, `topography`, `soil_classification`, `easement_type`, `environmental_status`, `parcel_apn`, and `adjacent_parcels`.

2. **Retrieve All Neighboring / Adjacent Parcels Data (MANDATORY)**:
   - Inspect the `adjacent_parcels` array from the subject lot's record.
   - You MUST call `get_lot_data` for **every single lot ID** listed in `adjacent_parcels` so you can investigate their exact FEMA `flood_zone` classifications, `topography`, `soil_classification`, `easement_type`, `acreage`, and `last_sale_price`.

3. **Calculate and Highlight Acquisition Cost-Per-Acre as a Core Valuation Metric (MANDATORY)**:
   - Derive and state the exact mathematical **Acquisition Cost-Per-Acre** based on the subject parcel's `last_sale_price` and `acreage` using the explicit formula:
     $$\text{Acquisition Cost-Per-Acre} = \frac{\text{Last Sale Price}}{\text{Acreage}}$$
   - State the exact formula, the substituted numbers, and the exact resulting dollar figure per acre (e.g., `$\$450,000 / 1.2\text{ acres} = \$375,000.00\text{ per acre}$`). Double-check all arithmetic to ensure 100% mathematical accuracy (round repeating decimals to two decimal places, e.g., `$520,000 / 1.5 acres = $346,666.67/acre`, `$680,000 / 4.4 acres = $154,545.45/acre`, `$380,000 / 2.6 acres = $146,153.85/acre`, `$450,000 / 1.7 acres = $264,705.88/acre`).
   - **Highlight and emphasize Acquisition Cost-Per-Acre as a CORE VALUATION METRIC** in **both** the `## Executive Summary` and a dedicated subsection under `## Acquisition Basis and Holding Costs`.
   - **Benchmark** the subject parcel's Acquisition Cost-Per-Acre against:
     1. The county **Assessed Land Value Per Acre** ($\text{Assessed Land Value} / \text{Acreage}$) from the tax assessment records, and
     2. The **Acquisition Cost-Per-Acre of the Neighboring/Adjacent Parcels** ($\text{Neighbor Last Sale Price} / \text{Neighbor Acreage}$) to evaluate relative land valuation and acquisition basis competitiveness.

4. **Consult Specialized Flood Risk and Tax Skills**:
   - Consult the `analyze-flood-risk` skill to perform an exhaustive analysis of both the subject parcel's flood zone and all neighboring/adjacent properties' FEMA flood zone classifications, drainage, runoff, and shared hydrological impacts on the subject parcel's development feasibility.
   - Consult the `analyze-tax-implications` skill and call `get_tax_assessment_data` using the subject lot's `parcel_apn` to investigate official property tax holding costs (annual and monthly) and assessed valuations.
