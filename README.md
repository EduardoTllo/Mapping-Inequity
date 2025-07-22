# Mapping Inequity: Travel Time Barriers to Health Care in Peru and Implications for Universal Health Coverage
## Contents

The Biostatistics team at the University of Engineering and Technology (UTEC) developed this project to provide policy-makers, researchers and the public with timely, accurate and openly accessible geospatial data on travel times to health care facilities across Peru. Using least-cost path modelling, we produce downloadable datasets of travel times to Level I, II and III facilities, accompanied by R scripts for reproduction and analysis.

## Get the data
https://drive.google.com/drive/folders/1bL4L9Xd-31vuzg48s5kpi0GqiB9GNs0v?usp=sharing

## Project Workflow

This repository is powered by three Quarto scripts (`.qmd`) that execute the full data processing, modelling and visualization pipeline. Run them in numerical order to reproduce every step of the analysis.

1. **`01_DFcleaning.qmd`**  
   _Data cleaning & preparation_  
   - Load raw CSV `TB.csv` (Healthcare facilities) with semicolon delimiter and comma decimals  
   - Filter rows where `categoria` matches “I”, “II” or “III”  
   - Extract category prefix into a new column `cat_pref`  
   - Convert `longitud` and `latitud` to numeric, drop rows with missing coordinates  
   - Split into three data frames (`df_I`, `df_II`, `df_III`) containing only lat/lon for each category  
   - Write out `coords_I.csv`, `coords_II.csv` and `coords_III.csv`

2. **`02_final_cost.qmd`**  
   _Friction surface construction & travel-time modelling_  
   - Rasterize vector inputs to a uniform grid  
   - Assign travel speeds by road class, slope (Tobler’s hiking function) and land-cover coefficient  
   - Build friction surface and apply least-cost path algorithm to compute travel times to Level I, II and III facilities
   - Export GeoTIFFs and data frames for each category 

3. **`03_graphics and data export.qmd`**  
   _Visualization & data export_  
   - Generate maps (departmental boxplots, threshold choropleths, travel-time surfaces)  
   - Compile summary tables of travel-time statistics  
   - Export final CSVs and graphics

## Credits
- UTEC Biostatistics Team
- Data sources: RENIPRESS (MINSA), OpenStreetMap, Copernicus, SRTM, INEI
- Methodology: Least-cost path modelling with the gdistance package in R
