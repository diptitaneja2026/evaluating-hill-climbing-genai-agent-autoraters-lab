---
name: analyze-tax-implications
description: Investigates property tax holding costs, assessment records, and acquisition cost-per-acre valuation metrics for a parcel.
metadata:
  adk_additional_tools:
    - get_tax_assessment_data
    - get_lot_data
---
# Analyze Tax Implications Skill

When investigating financial carrying costs, acquisition valuation basis, or tax implications for a lot:

1. **Retrieve Official Tax Assessment Data**:
   - Locate the `parcel_apn` from the lot property records.
   - Use the `get_tax_assessment_data` tool with that APN to retrieve official county tax assessment details (`annual_tax_amount`, `assessed_land_value`, `assessed_improvement_value`, `total_assessed_value`, `tax_year`, `payment_status`).
   - Calculate the exact **monthly tax holding cost** ($\text{Official Annual Tax Amount} / 12$) and compare the official annual tax against the preliminary `estimated_annual_taxes` from the lot record.

2. **Derive, Highlight, and Benchmark Acquisition Cost-Per-Acre as a Core Valuation Metric (MANDATORY)**:
   - Derive and state the exact mathematical **Acquisition Cost-Per-Acre** using the subject lot's `last_sale_price` and `acreage`:
     $$\text{Acquisition Cost-Per-Acre} = \frac{\text{Last Sale Price}}{\text{Acreage}}$$
   - Explicitly show the mathematical derivation formula and exact calculation in your report (e.g., `$\$450,000 / 1.2\text{ acres} = \$375,000.00\text{ per acre}$`).
   - Also compute the **Assessed Land Value Per Acre**:
     $$\text{Assessed Land Value Per Acre} = \frac{\text{Assessed Land Value}}{\text{Acreage}}$$
   - **Highlight and emphasize Acquisition Cost-Per-Acre as a CORE VALUATION METRIC** in both the `## Executive Summary` and `## Acquisition Basis and Holding Costs` sections, benchmarking the subject lot's Acquisition Cost-Per-Acre against its Assessed Land Value Per Acre and the Acquisition Cost-Per-Acre of its adjacent neighboring parcels (`adjacent_parcels`).
