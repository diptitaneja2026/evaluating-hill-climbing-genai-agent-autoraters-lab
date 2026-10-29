---
name: analyze-flood-risk
description: Investigates subject and neighboring property FEMA flood zone designations, drainage, runoff, and shared hydrological risks for real estate investment opportunity analysis.
metadata:
  adk_additional_tools:
    - get_lot_data
---
# Analyze Flood Risk Skill

When evaluating environmental, flood, or hydrological risks for a subject lot, you MUST evaluate **both** the subject parcel AND all neighboring/adjacent parcels:

1. **Subject Parcel Flood & Environmental Investigation**:
   - Check the exact FEMA `flood_zone` classification (e.g., `Zone AE (High Risk)`, `Zone VE (Coastal High Hazard)`, `Zone A (100-year Floodplain)`, or `Zone X (Minimal Risk)`), `topography` (slope grade), `soil_classification`, `easement_type`, `tree_canopy_coverage_pct`, and `environmental_status` (Phase I / Phase II ESA) of the subject lot.

2. **Neighboring / Adjacent Properties Flood Investigation (MANDATORY)**:
   - Check the `adjacent_parcels` field from the subject parcel's data.
   - Call `get_lot_data` for **every lot ID** listed in `adjacent_parcels` if not already retrieved.
   - Record the exact FEMA `flood_zone` designation, `topography`, `soil_classification`, and `easement_type` for each neighboring property.

3. **Exhaustive Neighboring Flood Risk & Shared Hydrological Analysis (MANDATORY IN REPORT)**:
   - In the `## Risk Assessment` section (and summarized in `## Executive Summary` and `## Recommendation`), include a dedicated subsection on **Neighboring Property Flood Zones & Shared Hydrological Impact Analysis**.
   - **Explicitly state the specific FEMA flood zone classifications of every neighboring/adjacent parcel by Lot ID** (including a side-by-side comparison table of Subject vs. Neighboring Lots showing Lot ID, FEMA Flood Zone, Topography/Slope, Soil Classification, and Easements).
   - **Critically analyze potential drainage, stormwater runoff, and shared hydrological impacts on the subject parcel's development feasibility**:
     - **Topographic & Cross-Boundary Runoff Dynamics**: Analyze how the slope grades (`topography`) of the subject lot and its neighbors interact (e.g., upslope runoff sheet flow onto flatter downslope parcels, or ponding/staging on 0-2% flat grades).
     - **Soil Permeability & Subsurface Hydrology**: Analyze how `soil_classification` (e.g., `Urban Land-Inwood complex`, `Clay loam (Moderate shrink-swell potential)`, `Sandy loam (Good drainage)`, `Bedrock outcrop / Shallow rocky soil`) across the contiguous lots affects infiltration rates, water table elevation, hydrostatic foundation pressure, and stormwater retention performance.
     - **Floodplain Connectivity & Backwater/Encroachment Effects**: Even if the subject parcel is in `Zone X (Minimal Risk)`, critically evaluate whether adjacent parcels in high-risk zones (`Zone AE`, `Zone VE`, `Zone A`) create boundary floodwater encroachment, stormwater outfall tailwater/backwater lock, or coastal velocity/surge exposure. Conversely, if the subject lot is in `Zone AE`, `Zone VE`, or `Zone A`, analyze shared floodplain storage displacement, FEMA "No Adverse Impact" (NAI) / zero-rise certification requirements (ensuring site fill or podium elevation does not divert floodwater onto adjacent lots), and regional drainage easements.
     - **Impact on Development Feasibility & Engineering Mitigation**: Detail how these shared hydrological conditions impact the subject parcel's buildable footprint, grading plan, foundation engineering, perimeter drainage infrastructure (interceptor swales, retention/detention basins, French drains), permitting timeline, and overall development feasibility.
