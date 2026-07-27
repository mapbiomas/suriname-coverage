```javascript
//Update for MapBiomas Suriname - JR 02/2026
var region = ee.FeatureCollection('projects/mapbiomas-suriname/assets/suriname_class_regions_1kmBuffer');

/*--------------------------------
--- Classification regions
--------------------------------*/

var param = {
  country: 'SURINAME',
  regionId: 80201,
  trees: 120,
  years: [
    1986,1987,1988,1989,1990,1991,1992,1993,1994,1995,1996,1997,1998,1999,2000,
    2001,2002,2003,2004,2005,2006,2007,
    2008,2009,2010,2011,2012,2013,2014,2015,2016,2017,2018,2019,2020,
    2021,2022,2023,2024,2025
  ],
  // Optional: point filtering by spectral indices
  // Each entry: [classId, indexName, lowerLimit, upperLimit]
  FiltroIndices: [],
  variables: [
    // If empty, all available bands from the mosaics will be used.
    // Otherwise, list the band names explicitly, e.g. 'ndvi_median', 'swir1_median', etc.
  ],
  tileScale: 6,
  additionalSamples: {
    polygons: [geometry_3, geometry_11, geometry_33],
    classes: [3, 11, 33],
    points: [1000, 10000, 10000]
  },
  // Years to be visualized on the map (preview only; does not affect export)
  yearsPreview: [1990, 2000, 2020, 2024, 2025],
  driveFolder: 'RF-PRELIMINARY-CLASSIFICATION',
  samplesVersion: 1,
  outputVersion: 1
};

// Filter region of interest by regionId
region = region.filterMetadata("id_regionC", "equals", param.regionId);

// Region raster mask (not strictly required for the buffer logic, but kept for reference)
var regionMask = region
  .map(function(item) {
    return item.set('version', 1);
  })
  .reduceToImage(['version'], ee.Reducer.first());

// Processing area = region + buffer
var bufferKm = 10; 
var processingGeom = region.geometry().buffer(bufferKm * 1000);  // region plus buffer

// Mask of the processing geometry (value 1 inside the buffer)
var processingMask = ee.Image.constant(1).clip(processingGeom);

// Small preview area on the coast:
// - if you draw a geometry in the editor named `geometry_preview`, it is used;
// - else, if a `geometry` exists, it is used;
// - otherwise, a 3 km buffer around the region is used as fallback.
var previewGeom;
if (typeof geometry_preview !== 'undefined') {
  previewGeom = geometry_preview;
} else if (typeof geometry !== 'undefined') {
  previewGeom = geometry;
} else {
  previewGeom = region.geometry().buffer(3000);
}

// Helper geometry image for some spatial filters (outsideVector)
var geom = ee.FeatureCollection(
    region.geometry().bounds()
  )
  .map(function(item) {
    return item.set('version', 1);
  })
  .reduceToImage(['version'], ee.Reducer.first());

// Input parameters
var country = param.country;
var regionId = param.regionId;
var variables = param.variables;
var nBands = variables.length;
var outputVersion = param.outputVersion;
var trees = param.trees;
var additionalSamples = param.additionalSamples;
var allYears = param.years;
var yearsPreview = param.yearsPreview;
var driveFolder = param.driveFolder;
var samplesVersion = param.samplesVersion;

// Paths
var basePath = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/classification/';
var samplesPath = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/SAMPLES/TRAINING/';
var trainingSamples = samplesPath + 'samples-' + country + '-' + regionId;
//var assetMosaic = 'projects/mapbiomas-raisg/MOSAICOS/mosaics-2';
var assetMosaic22 = 'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2';

// Palette
var palette = require('users/mapbiomas/modules:Palettes.js').get('classification8');

// Years list (in case you want to skip some years)
var missingYears = [];
var years = ee.List(allYears).removeAll(missingYears);

// Outputs initialization
var variablesImportance = ee.FeatureCollection([]);
var classified = ee.Image(0);

// Build mosaics collection using the processing geometry (region + buffer)
var joinedMosaics = ee.ImageCollection(assetMosaic22)
  //.merge(ee.ImageCollection(assetMosaic))
  .filterBounds(processingGeom);

// Function to generate additional samples inside polygons
var resampleCover = function(mosaic, additionalSamples) {
  
  var polygons = additionalSamples.polygons,
      classIds = additionalSamples.classes,
      points = additionalSamples.points,
      newSamples = [];
  
  polygons.forEach(function(polygon, i) {
    var newSample = mosaic.sample({
      numPixels: points[i],
      region: polygon,
      scale: 30,
      projection: 'EPSG:4326',
      seed: 1,
      geometries: true,
      tileScale: param.tileScale
    })
    .map(function(item) { return item.set('reference', classIds[i]); });
    newSamples.push(newSample);
  });
  
  return ee.FeatureCollection(newSamples).flatten();
};
 
// Random forest classification over the time series
years.evaluate(function(years, error){
  if (error) print(error.message);
  var variablesImportance = ee.FeatureCollection([]);

  years.forEach(function(year) {

    // Mosaic for the given year, clipped to processing geometry (region + buffer)
    var yearMosaic = joinedMosaics
      .filterMetadata('year', 'equals', year)
      .mosaic()
      .clip(processingGeom);

    // Select specific variables if provided
    if (variables.length > 0) yearMosaic = yearMosaic.select(variables);
    var bands = yearMosaic.bandNames();
    var contained = bands.containsAll(ee.List(variables));
    
    // Training samples for the given year
    var yearTrainingSamples = ee.FeatureCollection(
      ee.Algorithms.If(
        contained,
        ee.FeatureCollection(
          trainingSamples + '-' + year + '-' + samplesVersion),
        null
      )
    );
    
    var nClasSample = ee.List(
      ee.Algorithms.If(
        contained,
        yearTrainingSamples
          .reduceColumns(ee.Reducer.toList(), ['reference'])
          .get('list'),
        null
      )
    );

    // Optional: filter samples by index thresholds (if FiltroIndices is not empty)
    if (param.FiltroIndices.length > 0){
      print('Samples before index filtering (R-' + param.regionId + '-' + year + '): ', yearTrainingSamples.size());
      var tempPtos;
      param.FiltroIndices.forEach(function(Ind){
        tempPtos = yearTrainingSamples
          .filter(ee.Filter.eq('reference', Ind[0]))
          .filter(ee.Filter.gte(Ind[1], Ind[2]))
          .filter(ee.Filter.lte(Ind[1], Ind[3]));
        
        yearTrainingSamples = yearTrainingSamples
          .filter(ee.Filter.neq('reference', Ind[0]));
        
        yearTrainingSamples = yearTrainingSamples.merge(tempPtos);                     
      });
      print('Samples after index filtering (R-' + param.regionId + '-' + year + '): ', yearTrainingSamples.size());
    }

    // Number of distinct classes in samples
    nClasSample = nClasSample.reduce(ee.Reducer.countDistinct());
    
    // Additional samples from polygons
    if (additionalSamples.polygons.length > 0){
      var insidePolygons = ee.FeatureCollection(additionalSamples.polygons)
        .flatten()
        .reduceToImage(['id'], ee.Reducer.first());
      
      var outsidePolygons = insidePolygons.mask().eq(0).selfMask();
      outsidePolygons = geom.updateMask(outsidePolygons);
      
      var outsideVector = outsidePolygons.reduceToVectors({
        reducer: ee.Reducer.countEvery(),
        geometry: region.geometry().bounds(),
        scale: 30,
        maxPixels: 1e13
      });

      var newSamples = resampleCover(yearMosaic, additionalSamples);
      yearTrainingSamples = yearTrainingSamples.filterBounds(outsideVector)
        .merge(newSamples);
    }
    
    // Define classifier and compute importance tables
    var classifier = ee.Classifier.smileRandomForest({
      numberOfTrees: trees, 
      variablesPerSplit: 1
    });

    classifier = ee.Classifier(
      ee.Algorithms.If(
        contained,
        ee.Algorithms.If(
          // handle the "only one class" issue
          ee.Algorithms.IsEqual(nClasSample, 1),
          null,
          classifier.train(yearTrainingSamples, 'reference', bands)
        ),
        null
      )
    );
    
    var explainer = ee.Dictionary(
      ee.Algorithms.If(
        contained,
        ee.Algorithms.If(
          ee.Algorithms.IsEqual(nClasSample, 1),
          null,
          classifier.explain()
        ),
        null
      )
    );
    
    // Variable importance table
    var importances = ee.Feature(
      ee.Algorithms.If(
        contained,
        ee.Algorithms.If(
          ee.Algorithms.IsEqual(nClasSample, 1),
          null,
          ee.Feature(
            null,
            ee.Dictionary(explainer.get('importance'))
          )
          .set('_trees', explainer.get('numberOfTrees'))
          .set('_oobError', explainer.get('outOfBagErrorEstimate'))
          .set('_year', year)
        ),
        null
      )
    );
    
    variablesImportance = variablesImportance
      .merge(ee.FeatureCollection([importances]));
    
    // Classification for this year
    var img = yearMosaic.classify(classifier)
      .select(['classification'], ['classification_' + year]);

    // If classifier is null or only one class, use a constant "27" band as fallback
    var maskBand = ee.Image(27).rename('classification_' + year);

    // Build multi-year classification image (one band per year) over the full buffer
    classified = ee.Image(
      ee.Algorithms.If(
        contained,
        ee.Algorithms.If(
          ee.Algorithms.IsEqual(nClasSample, 1),
          classified.addBands(maskBand),
          classified.addBands(img)
        ),
        classified.addBands(maskBand)
      )
    )
    .unmask(27)
    .updateMask(processingMask)
    .toByte();
    
    // Visualization only for preview years, in a small coastal AOI (previewGeom)
    if (yearsPreview.indexOf(year) > -1) {

      // Mosaic preview for the selected year
      var yearMosaicPreview = yearMosaic.clip(previewGeom);

      Map.addLayer(
        yearMosaicPreview,
        {
          bands: ['swir1_median', 'nir_median', 'red_median'],
          gain: [0.08, 0.06, 0.2]
        },
        'MOSAIC ' + year.toString() + ' (preview)',
        true
      );

      // Classification preview for the selected year, coarser resolution for map rendering
      var imgPreview = img
        .unmask(27)
        .updateMask(processingMask)
        .clip(previewGeom)
        .reproject('EPSG:4326', null, 120);  // coarser scale for visualization only

      Map.addLayer(
        imgPreview,
        {
          min: 0,
          max: 62,
          palette: palette
        },
        'CLASSIFICATION ' + year + ' (preview)',
        true
      );
    }
    
    return classified;
  });
  
  // Final multi-year classification image (all bands: classification_YEAR)
  classified = classified.slice(1).toInt8()
    .set({
      code_region: regionId,
      country: country,
      version: outputVersion,
      RFtrees: trees,
      samples_version: samplesVersion,
      description: 'classification-v1',
      step: 'P04'
    })
    .updateMask(processingMask);  // keep region + buffer as processing domain

  // Export classification to GEE asset
  var filename = country + '-' + regionId + '-' + outputVersion;
  var imageId = basePath + filename;  
  var tableName = 'IMPORTANCE-TABLE-' + country + '-' + regionId + '-' + outputVersion;
  
  Export.image.toAsset({
    image: classified,
    description: filename,
    assetId: imageId,
    scale: 30,
    pyramidingPolicy: {
      '.default': 'mode'
    },
    maxPixels: 1e13,
    region: processingGeom   // region + buffer
  });
  
  // Export classification to Google Drive
  Export.image.toDrive({
    image: classified,
    description: filename + '-DRIVE',
    folder: driveFolder,
    scale: 30,
    maxPixels: 1e13,
    region: processingGeom
  });
  
  // Export variable importance table to Google Drive
  Export.table.toDrive({
    collection: variablesImportance, 
    description: tableName,
    folder: driveFolder,
    fileFormat: 'CSV'
  });
  
});
```
