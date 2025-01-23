# JEV-spillover-risk
GEE code to generate two predictive risk metrics for the transmission of JEV in Australia. Full publication and data sources available at:

### Step 1: Loading data into Google Earth Engine ###
The data used for to calculate JEV spillover risk in Australia included human population density, species distribution models for each potential vector and host, a meta-analysis of bloodmeal data to determine vector-host contact rate and a meta-analysis of vector competence studies to determine vector transmission and infection rates. 

First, all distribution maps must be loaded into GEE.

```javascript
// Load necessary datasets
var human_population = ee.Image('projects/ee-eloiseskinner90/assets/human_pop22'); // Human population density

// Load vector abundance and host distribution layers
var cx_annulirostris = ee.Image('projects/ee-eloiseskinner90/assets/Culex_annulirostris_Resample1');
var cx_quinquefasciatus = ee.Image('projects/ee-eloiseskinner90/assets/Culex_quinquefasciatus_Resample1');
var ardeidae = ee.Image('projects/ee-eloiseskinner90/assets/Ardeidae'); // Bird distribution (Ardeidae family)
var piggeries = ee.Image('projects/ee-eloiseskinner90/assets/Piggery_5km_buffer'); // Domestic pig distribution
var feral_pigs = ee.Image('projects/ee-eloiseskinner90/assets/Sus_scrofa_feral_resampled'); // Feral pig distribution

// --- Visualization for Input Layers ---
Map.addLayer(cx_annulirostris, {min: 0, max: 1, palette: ['black', 'white']}, 'Culex annulirostris');
Map.addLayer(cx_quinquefasciatus, {min: 0, max: 1, palette: ['black', 'white']}, 'Culex quinquefasciatus');
Map.addLayer(ardeidae, {min: 0, max: 1, palette: ['black', 'white']}, 'Ardeidae (Bird Distribution)');
Map.addLayer(piggeries, {min: 0, max: 1, palette: ['black', 'white']}, 'Piggeries (Domestic Pigs)');
Map.addLayer(feral_pigs, {min: 0, max: 1, palette: ['black', 'white']}, 'Feral Pigs');
```
### Step 2: Adding constants into Google Earth Engine ###

```javascript
// Constants for Culex annulirostris (Vector A)
var bloodfeeding_proportion_ardeidae_A = 0.1563;  // Proportion of blood meals from Ardeidae birds
var bloodfeeding_proportion_piggeries_A = 0.0749; // Proportion of blood meals from domestic pigs
var bloodfeeding_proportion_feral_pigs_A = 0.0749; // Proportion of blood meals from feral pigs
var infection_probability_A = 0.91;  // Probability of infection for Vector A
var transmission_potential_A = 0.5133; // Transmission potential for Vector A

// Constants for Culex quinquefasciatus (Vector B)
var bloodfeeding_proportion_ardeidae_B = 0.4311;  // Proportion of blood meals from Ardeidae birds
var bloodfeeding_proportion_piggeries_B = 0.0049; // Proportion of blood meals from domestic pigs
var bloodfeeding_proportion_feral_pigs_B = 0.0049; // Proportion of blood meals from feral pigs
var infection_probability_B = 0.77;  // Probability of infection for Vector B
var transmission_potential_B = 0.25; // Transmission potential for Vector B
```
### Step 3: Calculate JEV vectorial potential for each vector species ###
Transmission risk between each vector and host was calculated in a multiplicative manner at a 1km^2 scale. First the presence of the mosquito vector was multiplied with the occurrence of a host and the contact rate between each vector and host, then multiplied by the infection probability for that given vector and the transmission potential for that given vector. This was repeated for each vector-host combination. For a single vector, this was then combined to give a single vectorial potential per vector species.

```javascript

// --- Risk Calculations for Culex annulirostris (Vector A) ---
var ardeidae_risk_A = ardeidae.multiply(bloodfeeding_proportion_ardeidae_A)
    .multiply(infection_probability_A)
    .multiply(transmission_potential_A);

var piggeries_risk_A = piggeries.multiply(bloodfeeding_proportion_piggeries_A)
    .multiply(infection_probability_A)
    .multiply(transmission_potential_A);

var feral_pigs_risk_A = feral_pigs.multiply(bloodfeeding_proportion_feral_pigs_A)
    .multiply(infection_probability_A)
    .multiply(transmission_potential_A);

// Combined risk for Vector A
var combined_risk_A = cx_annulirostris.multiply(
    ardeidae_risk_A.add(piggeries_risk_A).add(feral_pigs_risk_A)
);
Map.addLayer(combined_risk_A, {min: 0, max: 100, palette: ['black', 'red']}, 'Combined Risk (Vector A)');

// --- Risk Calculations for Culex quinquefasciatus (Vector B) ---
var ardeidae_risk_B = ardeidae.multiply(bloodfeeding_proportion_ardeidae_B)
    .multiply(infection_probability_B)
    .multiply(transmission_potential_B);

var piggeries_risk_B = piggeries.multiply(bloodfeeding_proportion_piggeries_B)
    .multiply(infection_probability_B)
    .multiply(transmission_potential_B);

var feral_pigs_risk_B = feral_pigs.multiply(bloodfeeding_proportion_feral_pigs_B)
    .multiply(infection_probability_B)
    .multiply(transmission_potential_B);

// Combined risk for Vector B
var combined_risk_B = cx_quinquefasciatus.multiply(
    ardeidae_risk_B.add(piggeries_risk_B).add(feral_pigs_risk_B)
);
Map.addLayer(combined_risk_B, {min: 0, max: 100, palette: ['black', 'red']}, 'Combined Risk (Vector B)');
```

### Step 4: Calculate JEV vector-host transmission potential risk ###
By combining the vectorial potential for each mosquito species we estimate the vector-host transmission risk for JEV in Australia per 1km^2. The overall probability of having at least one infection was calculated using the simple probability law for the union of two elements 

```javascript
// --- Total Combined Risk ---
var total_risk = combined_risk_A.add(combined_risk_B)
    .subtract(combined_risk_A.multiply(combined_risk_B)); // Union Law for combined risks
Map.addLayer(total_risk, {min: 0, max: 100, palette: ['black', 'red']}, 'Total Combined Risk');
```

### Step 5: Calculate JEV spillover risk to people ###
Finally we estimate the spillover risk to people by multiplying the vector-host transmission risk by human population density. This was then aggregated by Local Government Area (in ArcGIS) to give total number and proportion of people at risk in each locality. 

```javascript
// --- Final Risk Map (Adjusted for Human Population) ---
var risk_map = total_risk.multiply(human_population);
Map.addLayer(risk_map, {min: 0, max: 100, palette: ['yellow', 'red']}, 'Final Risk Map');
```

### Step 6: Extracting maps ###
All maps were extracted from GEE and aligned in ArcGIS. The below code was used to extract the maps. 

```javascript
// --- Define Region and Export Outputs ---
var australia = ee.Geometry.Rectangle([113.0, -43.0, 156.0, -10.0]); // Define Australia's bounding box

// Export risk layers
Export.image.toDrive({
  image: combined_risk_A,
  description: 'Combined_Risk_Vector_A',
  scale: 1000,
  region: australia,
  maxPixels: 1e13,
  folder: 'Risk_Maps'
});
Export.image.toDrive({
  image: combined_risk_B,
  description: 'Combined_Risk_Vector_B',
  scale: 1000,
  region: australia,
  maxPixels: 1e13,
  folder: 'Risk_Maps'
});
Export.image.toDrive({
  image: total_risk,
  description: 'Total_Combined_Risk',
  scale: 1000,
  region: australia,
  maxPixels: 1e13,
  folder: 'Risk_Maps'
});
Export.image.toDrive({
  image: risk_map,
  description: 'Final_Risk_Map',
  scale: 1000,
  region: australia,
  maxPixels: 1e13,
  folder: 'Risk_Maps'
});
```
