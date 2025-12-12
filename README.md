// 1. LOAD POINTS + AOI
var BIRD_POINTS_ASSET =
  'projects/sincere-essence-470122-t2/assets/S123_7b3ca6342bf541ab8e07a00f4f75f277_SHP';

var birdPoints = ee.FeatureCollection(BIRD_POINTS_ASSET);

// aoi geometry
var aoi = birdPoints.geometry();

// Show the points
print('Bird points:', birdPoints);
Map.addLayer(birdPoints, {color: 'red'}, 'Bird points');
Map.centerObject(birdPoints, 13);

// ----- AOI from points -----
var aoi = birdPoints.geometry();  // your points

// Convert FeatureCollection of points to geometry to convex hull polygon
var aoiGeom = ee.FeatureCollection(aoi).geometry();
var aoiPolygon = aoiGeom.convexHull(50);  // 50m buffer

// Check AOI is valid 
print('AOI area:', aoiPolygon.area());
Map.addLayer(aoiPolygon, {color: 'red'}, 'AOI Polygon');



// ================== 2. LEAFY-SEASON IMAGERY (REDUCED) ==================
var leafyStart = '2023-05-01';
var leafyEnd   = '2023-09-30';

var s2_leafy = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(aoi)                     // <-- AOI USED HERE, must be defined ABOVE
  .filterDate(leafyStart, leafyEnd)
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 40));

print('Reduced leafy-season image count:', s2_leafy.size());

// Make a composite
var leafyComposite = s2_leafy.median();

// Display it
Map.addLayer(
  leafyComposite,
  {bands: ['B4','B3','B2'], min:0, max:3000},
  'Leafy Season (2023)'
);

Map.addLayer(birdPoints, {color: 'red', pointRadius: 6}, 'Bird Points (on top)');

Map.centerObject(aoi, 13);



// ===================================
// CLOUD MASK FUNCTION for Sentinel-2
// ===================================
function maskS2clouds(img) {
  var qa = img.select('QA60');
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
               .and(qa.bitwiseAnd(cirrusBitMask).eq(0));
  return img.updateMask(mask).copyProperties(img, ['system:time_start']);
}

// ===========================
// NDVI FOR 2023 leafy season
// ===========================
var s2_2023 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(aoi)
  .filterDate('2023-05-01', '2023-09-30')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 40))
  .map(maskS2clouds);

var img2023 = s2_2023.median();          // do NOT clip yet
var ndvi2023 = img2023.normalizedDifference(['B8','B4'])
  .rename('NDVI_2023');

Map.addLayer(
  ndvi2023,
  {min: 0, max: 1, palette: ['white','lightgreen','green']},
  'NDVI 2023 (full)'
);

var ndvi2023_clip = ndvi2023.clip(aoiPolygon);
Map.addLayer(
  ndvi2023_clip,
  {min: 0, max: 1, palette: ['white','lightgreen','green']},
  'NDVI 2023 (clipped)'
);

// ===========================
// NDVI FOR 2018 leafy season
// ===========================
var s2_2018 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(aoi)
  .filterDate('2018-05-01', '2018-09-30')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 40))
  .map(maskS2clouds);

print('2018 image count:', s2_2018.size());   // should be 90+

var img2018 = s2_2018.median();   // do NOT clip yet
var ndvi2018 = img2018.normalizedDifference(['B8','B4'])
  .rename('NDVI_2018');

Map.addLayer(
  ndvi2018,
  {min: 0, max: 1, palette: ['white','lightgreen','green']},
  'NDVI 2018 (full)'
);

var ndvi2018_clip = ndvi2018.clip(aoiPolygon);
Map.addLayer(
  ndvi2018_clip,
  {min: 0, max: 1, palette: ['white','lightgreen','green']},
  'NDVI 2018 (clipped)'
);

// =================================
// NDVI CHANGE ANALYSIS 2023 - 2018
// =================================
var ndviChange = ndvi2023.subtract(ndvi2018).rename('NDVI_Change');

Map.addLayer(
  ndviChange,
  {min: -0.3, max: 0.3, palette: ['red', 'white', 'blue']},
  'NDVI Change (2023 - 2018)'
);

// =======================
// ADD BIRD POINTS ON TOP
// =======================
Map.addLayer(
  birdPoints.style({color: 'yellow', pointSize: 6}),
  {},
  'Bird Points (on top)'
);


// histogram
var hist = ui.Chart.image.histogram(
  ndvi2018_clip.addBands(ndvi2023_clip),
  aoiPolygon,
  10
)
.setSeriesNames(['NDVI 2018', 'NDVI 2023'])
.setOptions({
  title: 'NDVI Histogram Comparison (2018 vs 2023)',
  hAxis: {title: 'NDVI Value'},
  vAxis: {title: 'Frequency'},
  colors: ['#1f77b4', '#d62728']
});


print(hist);


// NDVI diagnostics
var stats2018 = ndvi2018_clip.reduceRegion({
  reducer: ee.Reducer.minMax().combine('mean', null, true),
  geometry: aoi,
  scale: 10,
  maxPixels: 1e13
});

var stats2023 = ndvi2023_clip.reduceRegion({
  reducer: ee.Reducer.minMax().combine('mean', null, true),
  geometry: aoi,
  scale: 10,
  maxPixels: 1e13
});

print('NDVI 2018 stats:', stats2018);
print('NDVI 2023 stats:', stats2023);



// ==================================
// K-MEANS LAND COVER CLASSIFICATION
// ==================================


// Bands used for clustering
var trainingBands = ['B2', 'B3', 'B4', 'B8'];  // blue, green, red, NIR

// Train classifier using your corrected polygon AOI
var classifier = ee.Clusterer.wekaKMeans(6).train({
  features: leafyComposite.sample({
    region: aoiPolygon,   
    scale: 10,
    numPixels: 5000
  }),
  inputProperties: trainingBands
});

// Apply classifier to the image
var classified = leafyComposite.select(trainingBands).cluster(classifier);

// Clip to AOI polygon
var classified_clip = classified.clip(aoiPolygon);

// Add to map for visualization
Map.addLayer(classified_clip.randomVisualizer(), {}, 'K-Means Classification');


var clusterStats = classified_clip.reduceRegion({
  reducer: ee.Reducer.frequencyHistogram(),
  geometry: aoiPolygon,
  scale: 10,
  maxPixels: 1e13
});

print('Cluster frequencies:', clusterStats);

// Convert dictionary for charting
var dict = ee.Dictionary(clusterStats.get('cluster'));
var chart = ui.Chart.array.values(dict.values(), 0, dict.keys())
  .setChartType('ColumnChart')
  .setOptions({
    title: 'Pixel Count by K-Means Cluster',
    hAxis: {title: 'Cluster'},
    vAxis: {title: 'Pixel Count'},
    legend: 'none'
  });

print(chart);

// Full classification map
Map.addLayer(classified.randomVisualizer(), {}, 'K-Means Classification (Full)');


Map.addLayer(birdPoints, {color: 'blue', pointRadius: 6}, 'Bird Points (on top)');



// ============= CANOPY MASKS =============
var canopyThreshold = 0.5;  // you can adjust to 0.55 or 0.6

// --- FULL CANOPY MASK (entire NDVI image, NOT clipped) ---
var canopyMask_full = ndvi2023.gt(canopyThreshold).rename('canopy_full');

Map.addLayer(
  canopyMask_full.updateMask(canopyMask_full),
  {palette: ['darkgreen']},
  'Full Canopy Mask (NDVI > 0.5)'
);


// --- AOI-CLIPPED CANOPY MASK (only inside polygon) ---
var canopyMask_clipped = ndvi2023_clip.gt(canopyThreshold).rename('canopy_clipped');

Map.addLayer(
  canopyMask_clipped.updateMask(canopyMask_clipped),
  {palette: ['#1B5E20']},
  'Clipped Canopy Mask (NDVI > 0.5)'
);



// ================== BUFFER AROUND POINTS ==================
var bufferRadius = 50;  // meters (you can change to 75 or 100)

var bufferedPoints = birdPoints.map(function(pt) {
  return pt.buffer(bufferRadius).set({'buffer_m': bufferRadius});
});

Map.addLayer(bufferedPoints, {color: 'yellow'}, 'Buffered Points');



// =========== FIXED CANOPY AREA & PERCENT =============

var pixelArea = ee.Image.pixelArea();

// Canopy area in m²
var canopyAreaImage = canopyMask_clipped
    .updateMask(canopyMask_clipped)
    .multiply(pixelArea)
    .rename('can_area');

// TOTAL area = buffer area, not NDVI mask area
var totalAreaImage = ee.Image.constant(1)
    .multiply(pixelArea)
    .rename('tot_area');

var pointsWithCanopy = bufferedPoints.map(function(feat) {
  var geom = feat.geometry();

  // Sum canopy m²
  var canopyStats = canopyAreaImage.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: geom,
    scale: 10,
    maxPixels: 1e9
  });

  // Sum total m²
  var totalStats = totalAreaImage.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: geom,
    scale: 10,
    maxPixels: 1e9
  });

  var canopyArea = ee.Number(canopyStats.get('can_area'));
  var totalArea  = ee.Number(totalStats.get('tot_area'));

  var canopyPct = canopyArea.divide(totalArea).multiply(100);

  return feat.set({
    leafy_canopy_pct: canopyPct
  });
});

print('Corrected canopy %:', pointsWithCanopy);


// ============= CHECK CANOPY % VALUES =============
pointsWithCanopy.evaluate(function(fc) {
  print('------- CANOPY % CHECK -------');
  fc.features.forEach(function(f, i) {
    print(
      'Point ' + i + 
      '  |  leafy_canopy_pct = ', 
      f.properties.leafy_canopy_pct
    );
  });
});


// ================== GFCC TREE CANOPY COVER (PRODUCT-BASED) ==================
// Global Forest Canopy Cover (~2010), 30m resolution
var gfcc = ee.ImageCollection('NASA/MEASURES/GFCC/TC/v3')
              .first()
              .select('tree_canopy_cover');

Map.addLayer(
  gfcc,
  {min: 0, max: 100, palette: ['white','green']},
  'GFCC Tree Canopy %'
);

// Add GFCC canopy % to each point buffer
var pointsWithProduct = pointsWithCanopy.map(function(feat) {
  var geom = feat.geometry();

  var prodMean = gfcc.reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: geom,
    scale: 30,
    maxPixels: 1e9
  }).get('tree_canopy_cover');

  return feat.set({
    gfcc_canopy_pct: prodMean
  });
});

print('Points with NDVI canopy % + GFCC canopy %:', pointsWithProduct);


Export.table.toDrive({
  collection: pointsWithProduct,
  description: 'bird_canopy_metrics',
  fileFormat: 'CSV'
});













