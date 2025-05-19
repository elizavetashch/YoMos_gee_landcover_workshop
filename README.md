<hr>

# 🌍 YoMos Workshop: Land Cover Metrics in Google Earth Engine and R

## 🔍 Workshop Goals

- Use Google Earth Engine to calculate area and edge metrics of land cover classes within buffers.
- Export the output as CSV.
- Load and process the CSV in R.
- Calculate landscape metrics like Shannon diversity, Simpson evenness, and perimeter-to-area ratios.

---

## 🧩 Step-by-Step

### 🛰️ Google Earth Engine Part 🛰️

### Step 1

1. Upload the 20250520_YoMos_sample.csv into your assets in [Google Earth Engine] https://code.earthengine.google.com/
2. Create a variable called `points` to read in your dataset

<details>
  <summary>Click here to see the solution</summary>

```java
var points = ee.FeatureCollection(".../assets/20250520_YoMos_sample");

```
</details>

3. Visualise the points on the map
<details>
  <summary>Click here to see the solution</summary>

```java
Map.addLayer(points, {color: 'red'}, "points");
```
</details>

### Step 2: Import Data

```java
var five_year = ee.ImageCollection("projects/sat-io/open-datasets/GLC-FCS30D/five-years-map"); // import the landcover map for 1985-1995
var annual = ee.ImageCollection("projects/sat-io/open-datasets/GLC-FCS30D/annual"); // import the annual landcover map for 2000-2022
```

### Step 3: Filter points by year
<details> <summary>Click here to see the solution</summary>

```java

var years = [1985, 1990, 1995];
var filteredPoints = {}; 

years.forEach(function(year) {
  filteredPoints[year] = points.filter(ee.Filter.eq('landcover_map_year', year));
});
```
</details>

### Step 4: Buffer the points 

<details> <summary>Click here to see the solution</summary>
```java
function bufferPoints(radius, bounds) { 
  return function(pt) {
    pt = ee.Feature(pt);
    var bufferedGeom = bounds ? pt.buffer(radius).bounds() : pt.buffer(radius);
    return ee.Feature(bufferedGeom).copyProperties(pt);
  };
}
```
</details>


### Step 5: Edge Length and Area Calculation 

<details> <summary>Click here to see the solution</summary>


```java
// List of all land cover classes present in the map:
var classes = [10, 11, 12, 20, 51, 52, 61, 62, 71, 72, 81, 82, 91, 92, 120, 121, 122, 
               130, 140, 150, 152, 153, 181, 182, 183, 184, 185, 186, 187, 190, 200, 
               201, 202, 210, 220, 0]; 

// (4.1) Select images
var images = {
  1985: five_year.mosaic().select('b1'),
  1990: five_year.mosaic().select('b2'),
  1995: five_year.mosaic().select('b3')
};

// (4.2) Apply Canny edge detection AFTER buffering
function detectEdges(image, bufferedGeometry) {
  return ee.Algorithms.CannyEdgeDetector({
    image: image.clip(bufferedGeometry),  
    threshold: 0.7,
    sigma: 1
  }).selfMask();
}

// (4.3.1) Edge and Area Calculation
function calculateMetrics(image, edges, geometry, classValue) {
  var classMask = image.eq(classValue); 
  var classEdges = edges.updateMask(classMask); 
 
  var edgeLength = classEdges.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: geometry,
    scale: 5,
    maxPixels: 1e29
  }).get(image.bandNames().get(0));
  
  var areaImage = ee.Image.pixelArea().updateMask(classMask);
  var area = areaImage.reduceRegion({
    reducer: ee.Reducer.sum(), 
    geometry: geometry,
    scale: 5,
    maxPixels: 1e29
  }).get('area');
  
  return ee.Dictionary({class: classValue, edgelength_m: edgeLength, area_m2: area});
} 

// (4.4) Process edge length and area for points in a given year
function processMetricsForYear(filteredPoints, image, year) {
  var bufferedPoints = filteredPoints.map(bufferPoints(1000)); 
  var results = bufferedPoints.map(function(point) {
    var bufferGeom = point.geometry();
    var edges = detectEdges(image, bufferGeom);  
    var metrics = ee.List(classes.map(function(classValue) {
      return calculateMetrics(image, edges, bufferGeom, classValue);  
    }));
    return point.set('metrics', metrics);
  });
  return results;
}

// (4.5) Unpack the metrics (area and edge length) list into a feature collection
function unpackMetrics(feature) {
  var metrics = ee.List(feature.get('metrics'));
  var newFeatures = metrics.map(function(metric) {
    metric = ee.Dictionary(metric);
    return feature.copyProperties(feature).set({ 
      'class': metric.get('class'),
      'edgelength_m': metric.get('edgelength_m'),
      'area_m2': metric.get('area_m2')
    });
  });
  return ee.FeatureCollection(newFeatures);
}

```
</details>

### Step 6: Processing and Export 

<details> <summary>Click here to see the solution</summary>

```java
// Apply function to process metrics (edge length and area) for each year
/*
years.forEach(function(year) {
  var metricsResults = processMetricsForYear(filteredPoints[year], images[year], year);
  var unpackedFeatureCollection = ee.FeatureCollection(metricsResults).map(unpackMetrics).flatten();

  Export.table.toDrive({
    collection: unpackedFeatureCollection, 
    description: 'metrics_1000m_' + year, 
    folder: "YoMos_Landcover_Workshop",
    fileFormat: 'CSV' 
  });
});
*/

// For annual mosaic (2000 onwards)
/*
for (var i = 1; i <= 23; i++) {
  var year = 1999 + i;  // starts at year 2000
  var image = annual.mosaic().select("b" + i);
  var filteredPoints = points.filter(ee.Filter.eq('landcover_map_year', year));

  var metricsResults = processMetricsForYear(filteredPoints, image, year);
  var unpackedFeatureCollection = ee.FeatureCollection(metricsResults).map(unpackMetrics).flatten();

  Export.table.toDrive({
    collection: unpackedFeatureCollection,
    description: 'metrics_1000m_' + year, 
    folder: "YoMos_Landcover_Workshop", 
    fileFormat: 'CSV' 
  });
}
*/

```
</details>

### 🌟 Bonus Step: Visualisation  

<details> <summary>Click here to see the solution</summary>
  
```java
// Setup basemaps
var snazzy = require("users/aazuspan/snazzy:styles");
snazzy.addStyle("https://snazzymaps.com/style/38/shades-of-grey", "Shades of Grey");


var palette = [
  "#ffff64", "#ffff64", "#ffff00", "#aaf0f0", "#4c7300", "#006400", "#a8c800", "#00a000", 
  "#005000", "#003c00", "#286400", "#285000", "#a0b432", "#788200", "#966400", "#964b00", 
  "#966400", "#ffb432", "#ffdcd2", "#ffebaf", "#ffd278", "#ffebaf", "#00a884", "#73ffdf", 
  "#9ebb3b", "#828282", "#f57ab6", "#66cdab", "#444f89", "#c31400", "#fff5d7", "#dcdcdc", 
  "#fff5d7", "#0046c8", "#ffffff", "#ffffff"
];

var recodeClasses = function(image) {
  // Define the class values
  var classes = [10, 11, 12, 20, 51, 52, 61, 62, 71, 72, 81, 82, 91, 92, 120, 121, 122, 
                 130, 140, 150, 152, 153, 181, 182, 183, 184, 185, 186, 187, 190, 200, 
                 201, 202, 210, 220, 0];
  var reclassed = image.remap(classes, ee.List.sequence(1, classes.length));
  return reclassed;
};

  // Function to add a layer with given settings
var addLayer = function(image, name) {
  Map.addLayer(image, {palette: palette}, name,false);
};

  // Apply the function to your images and add layers
addLayer(recodeClasses(five_year.mosaic().select('b1')), 'Land Cover 1985');
addLayer(recodeClasses(five_year.mosaic().select('b2')), 'Land Cover 1990',false);
addLayer(recodeClasses(five_year.mosaic().select('b3')), 'Land Cover 1995',false);


  // Load the GLC-FCS30D collection
var image = annual.mosaic();

  // Iterate over each band (year) in the image
for (var i = 1; i <= 23; i++) {
  var year = 1999 + i; // starts at year 2000 for annual maps
  var layerName = "Land Cover " + year.toString();
  var band = image.select("b" + i);
  
  // Apply the function to the band and add layer
  addLayer(recodeClasses(band), layerName);
}

  // Define a dictionary for legend and visualization
var dict = {
  "names": [
    "Rainfed cropland",
    "Herbaceous cover cropland",
    "Tree or shrub cover (Orchard) cropland",
    "Irrigated cropland",
    "Open evergreen broadleaved forest",
    "Closed evergreen broadleaved forest",
    "Open deciduous broadleaved forest (0.15<fc<0.4)",
    "Closed deciduous broadleaved forest (fc>0.4)",
    "Open evergreen needle-leaved forest (0.15< fc <0.4)",
    "Closed evergreen needle-leaved forest (fc >0.4)",
    "Open deciduous needle-leaved forest (0.15< fc <0.4)",
    "Closed deciduous needle-leaved forest (fc >0.4)",
    "Open mixed leaf forest (broadleaved and needle-leaved)",
    "Closed mixed leaf forest (broadleaved and needle-leaved)",
    "Shrubland",
    "Evergreen shrubland",
    "Deciduous shrubland",
    "Grassland",
    "Lichens and mosses",
    "Sparse vegetation (fc<0.15)",
    "Sparse shrubland (fc<0.15)",
    "Sparse herbaceous (fc<0.15)",
    "Swamp",
    "Marsh",
    "Flooded flat",
    "Saline",
    "Mangrove",
    "Salt marsh",
    "Tidal flat",
    "Impervious surfaces",
    "Bare areas",
    "Consolidated bare areas",
    "Unconsolidated bare areas",
    "Water body",
    "Permanent ice and snow",
    "Filled value"
  ],
  "colors": [
    "#ffff64",
    "#ffff64",
    "#ffff00",
    "#aaf0f0",
    "#4c7300",
    "#006400",
    "#a8c800",
    "#00a000",
    "#005000",
    "#003c00",
    "#286400",
    "#285000",
    "#a0b432",
    "#788200",
    "#966400",
    "#964b00",
    "#966400",
    "#ffb432",
    "#ffdcd2",
    "#ffebaf",
    "#ffd278",
    "#ffebaf",
    "#00a884",
    "#73ffdf",
    "#9ebb3b",
    "#828282",
    "#f57ab6",
    "#66cdab",
    "#444f89",
    "#c31400",
    "#fff5d7",
    "#dcdcdc",
    "#fff5d7",
    "#0046c8",
    "#ffffff",
    "#ffffff",
    "#ffffff"
  ]
};

var legend = ui.Panel({
  style: {
    position: 'middle-right',
    padding: '8px 15px'
  }
});

  // Create and add the legend title.
var legendTitle = ui.Label({
  value: 'GLC FCS Classes',
  style: {
    fontWeight: 'bold',
    fontSize: '18px',
    margin: '0 0 4px 0',
    padding: '0'
  }
});
legend.add(legendTitle);

  // Creates and styles 1 row of the legend.
  var makeRow = function(color, name) {
    // Create the label that is actually the colored box.
    var colorBox = ui.Label({
      style: {
        backgroundColor: color,
        // Use padding to give the box height and width.
        padding: '8px',
        margin: '0 0 4px 0'
      }
    });

  // Create the label filled with the description text.
  var description = ui.Label({
    value: name,
    style: {margin: '0 0 4px 6px'}
  });

  return ui.Panel({
    widgets: [colorBox, description],
    layout: ui.Panel.Layout.Flow('horizontal')
  });
};
  var palette = dict['colors'];
  var names = dict['names'];

  for (var i = 0; i < names.length; i++) {
    legend.add(makeRow(palette[i], names[i]));
  }

  // Print the panel containing the legend
print(legend);


// Edge Visualisation of Land Cover Classes

// five_year mosaic 
var edges = {};
var years = [1985, 1990, 1995];
var images = {
  1985: five_year.mosaic().select('b1'),
  1990: five_year.mosaic().select('b2'),
  1995: five_year.mosaic().select('b3')
};

years.forEach(function(year) {
  
  edges[year] = ee.Algorithms.CannyEdgeDetector({
    image: images[year],
    threshold: 0.7,
    sigma: 1
  }).selfMask();

  Map.addLayer(edges[year], {palette: ['white'], min: 0, max: 1}, 'Edges for ' + year, false);

});

// annual mosaic
for (var i = 1; i <= 23; i++) {
  var year = 1999 + i;  
  var image = annual.mosaic().select("b" + i);
  
    var  edge = ee.Algorithms.CannyEdgeDetector({
    image: image,
    threshold: 0.7,
    sigma: 1
    }).selfMask();
  
    Map.addLayer(edge, {palette: ['white'], min: 0, max: 1}, 'Edges for ' + year, false);
  
}

```
</details>


<hr>

By the end of the GEE step, you should have csv outputs for landcover metrics such as area and edgelength for every point  between the years 1985-2015.
If you were unable to produce the csv files, please download the Workshop folder from the [drive](https://drive.google.com/drive/folders/1r6OcZywKoa0x-rZ0m0M665cp94B02NRq?usp=sharing).

<hr>

### 🧮 R Part 🧮

To proceed to the R part, simply download the R markdown 20250520_YoMos_R_Landcover.Rmd  and legend_classcode_landcovertypes.csv.
The R markdown is well structured and should be easy to follow.

The shannon and simpson indices will be calculated using the `vegan`package.

<hr>
To check your code, or if you lost track of steps, please check the provided solution: 
[Land cover metrics calculation GEE](https://code.earthengine.google.com/4731a23a969102a10127c3523887c235?noload=true)
