```javascript
/** 
 * P05 PRELIMINARY CLASSIFICATION GAPFILL
 * Update  2018___   Marcos:   
 * Update  20181008  EYTC: Adaptation for col2 
 * Update  20181021  EYTC: Update for years without classification in the whole region
 * Update  20191030  João: optimization and image metadata
 * Update  20201027  EYTC: Update for col3
 * Update  20210123  EYTC: Multifunctions for exclusion and inputasset
 * Update  20250731  Joaquim: Update for col1 MapBiomas Suriname
 *
 * @input
 * param: parameter object in JSON format. Contains:
 *    code_region: classification region ID.
 *    country: country name in uppercase letters.
 *    version_input: input classification version number from step P04.
 *    version_out: number of the version to be exported as asset.
 *    years: year for classification visualization.
 **/

/**
 * User defined parameters
 */
var param = {
  code_region: 80201,  // Classification region
  country: 'SURINAME',
  year: [1985,1986,1987,1990,2000,2021,2024,2025],  // Visualization only
  version_input: 11111,
  version_output: 11112,
  ExportOpcion: {   // Export options
    DriveFolder: 'DRIVE-EXPORT',  // Folder to export drive file
    exportClasifToDrive:  false,  // Export classifications to Drive (true or false)
    exportEstadistica:    false   // Export areas (true or false)
  },
  exclusion:{  // Indicate in the list the classes and years to exclude in the filter
    clases : [],  // list of classes to exclude in all years
    years  : []   // list of years to exclude with all classes
  }
};

var assetCollection      = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/classification/';
var assetOutput          = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/classification/';
var assetOutputMetadata  = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/metadata/';
var assetRegions         = 'projects/mapbiomas-suriname/assets/classification_regions_suriname';
//var assetMosaic          = 'projects/mapbiomas-raisg/MOSAICOS/mosaics-2';
var assetMosaic22        = 'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2';

var years = [
  1985, 1986, 1987, 1988,
  1989, 1990, 1991, 1992,
  1993, 1994, 1995, 1996,
  1997, 1998, 1999, 2000,
  2001, 2002, 2003, 2004,
  2005, 2006, 2007, 2008,
  2009, 2010, 2011, 2012,
  2013, 2014, 2015, 2016,
  2017, 2018, 2019, 2020,
  2021, 2022, 2023, 2024,
  2025
];

var palettes   = require('users/mapbiomas/modules:Palettes.js');
var eePalettes = require('users/gena/packages:palettes');

// Region feature and raster mask (original classification region, without buffer)
var regions = ee.FeatureCollection(assetRegions)
  .filterMetadata('id_regionC', 'equals', param.code_region);

var setVersion = function(item) { return item.set('version', 1); };

var regionRaster = regions
  .map(setVersion)
  .reduceToImage(['version'], ee.Reducer.first());

// Set up mosaics (only for visualization at the end)
var mosaicRegion = param.code_region.toString().slice(0, 3);
if (mosaicRegion === '211' || mosaicRegion === '205'){ mosaicRegion = '210'; }

var mosaic = ee.ImageCollection(assetMosaic22)
  //.merge(ee.ImageCollection(assetMosaic))
  .filter(ee.Filter.eq('region_code', parseInt(mosaicRegion)));

/**
 * Function to apply temporal gap fill
 */
var applyGapFill = function (image) {
  // apply the gap fill from t0 until tn
  var imageFilledt0tn = bandNames.slice(1)
    .iterate(
      function (bandName, previousImage) {
        var currentImage = image.select(ee.String(bandName));
        previousImage = ee.Image(previousImage);
        currentImage = currentImage.unmask(
          previousImage.select([0]));
        return currentImage.addBands(previousImage);
      }, ee.Image(image.select([bandNames.get(0)]))
    );
  imageFilledt0tn = ee.Image(imageFilledt0tn);

  // apply the gap fill from tn until t0
  var bandNamesReversed = bandNames.reverse();

  var imageFilledtnt0 = bandNamesReversed.slice(1)
    .iterate(
      function (bandName, previousImage) {
        var currentImage = imageFilledt0tn.select(ee.String(bandName));
        previousImage = ee.Image(previousImage);
        currentImage = currentImage.unmask(
          previousImage.select(previousImage.bandNames().length().subtract(1)));
        return previousImage.addBands(currentImage);
      }, ee.Image(imageFilledt0tn.select([bandNamesReversed.get(0)]))
    );
  imageFilledtnt0 = ee.Image(imageFilledtnt0).select(bandNames);
  return imageFilledtnt0;
};

// Version control
var version_input  = param.version_input;
var version_output = param.version_output;

// Load classification from P04 (this image already includes the buffer area + mask)
var assetPath = assetCollection + param.country + '-' + param.code_region + '-' + version_input;
var image = ee.Image(assetPath);

print('Loaded image', assetPath);

// IMPORTANT: use the classification image geometry as processing/export region
// This guarantees we keep exactly the same extent (including buffer) as P04.
var processingGeom = image.geometry();

// get band names list 
var bandNames = ee.List(
  years.map(function (year) {
    return 'classification_' + String(year);
  })
);

var bandNamesExclude = ee.List(
  param.exclusion.years.map(function (year) {
    return 'classification_' + String(year);
  })
);

if (param.exclusion.years.length > 0) {
  var yearmaskExcl = ee.Image(0);
  param.exclusion.years.forEach(function(year){
    yearmaskExcl = yearmaskExcl.addBands(
      ee.Image(0).rename('classification_' + String(year))
    );
  });
  yearmaskExcl = yearmaskExcl.slice(1).selfMask();
  print('Year mask exclusion', yearmaskExcl);
}

// Insert pixel mask for class 27
var original = image;
if (param.exclusion.years.length > 0){
  image = image.addBands(yearmaskExcl, null, true);
}

var classif = ee.Image();
var bandnameReg = image.bandNames();

// rebuild image ensuring that masked class 27 stays consistent
bandnameReg.getInfo().forEach(function (bandName) {
  var imagey = image.select(bandName);
  var band0  = imagey.updateMask(imagey.unmask().neq(27));
  classif    = classif.addBands(band0.rename(bandName));
});

image = classif.select(bandnameReg);

// generate a histogram dictionary of [bandNames, image.bandNames()]
var bandsOccurrence = ee.Dictionary(
  bandNames.cat(image.bandNames()).reduce(ee.Reducer.frequencyHistogram())
);

print('Input image (after exclusions and mask handling)', image);

// insert a masked band for missing years
var bandsDictionary = bandsOccurrence.map(
  function (key, value) {
    return ee.Image(
      ee.Algorithms.If(
        ee.Number(value).eq(2),
        image.select([key]).byte(),
        ee.Image().rename([key]).byte().updateMask(image.select(0))
      )
    );
  }
);

// convert dictionary to image (full time series, all years)
var imageAllBands = ee.Image(
  bandNames.iterate(
    function (band, img) {
      return ee.Image(img).addBands(bandsDictionary.get(ee.String(band)));
    },
    ee.Image().select()
  )
);

// generate "pixel year" image (for metadata)
var imagePixelYear = ee.Image.constant(years)
  .updateMask(imageAllBands)
  .rename(bandNames);

// apply the gap fill
var imageFilledtnt0 = applyGapFill(imageAllBands);
var imageFilledYear = applyGapFill(imagePixelYear);

//************************************************
// second dictionary: use original where available, 27 otherwise
var bandsDictionaryTwo = bandsOccurrence.map(
  function (key, value) {
    return ee.Image(
      ee.Algorithms.If(
        ee.Number(value).eq(2),
        original.select([key]).byte(),
        ee.Image(27).rename([key]).byte().updateMask(image.select(0))
      )
    );
  }
);

// convert second dictionary to image
var imageAllBandsTwo = ee.Image(
  bandNames.iterate(
    function (band, img) {
      return ee.Image(img).addBands(bandsDictionaryTwo.get(ee.String(band)));
    },
    ee.Image().select()
  )
);
//************************************************

var Class_Original = imageAllBandsTwo;
var Class_Filtrada = imageFilledtnt0.select(bandNames);

// Exclude classes in gapfill (if any)
if (param.exclusion.clases.length > 0){

  param.exclusion.clases.forEach(function(clase){
    Class_Filtrada = Class_Filtrada.where(
      Class_Filtrada.eq(clase),
      Class_Original
    );
  });
  print('Classes excluded in the temporal filter', param.exclusion.clases);
}

// Exclude specific years from gapfill (keep original values)
if (param.exclusion.years.length > 0){
  var yearExlud = original.select(bandNamesExclude);
  Class_Filtrada = Class_Filtrada.addBands(yearExlud, null, true);
  print('Years excluded in the temporal filter', param.exclusion.years);
}

imageFilledtnt0 = Class_Filtrada.select(bandNames);

/**
 * Export images to asset
 */
var imageName = param.country + '-' + param.code_region + '-' + version_output;

imageFilledtnt0 = imageFilledtnt0.select(bandNames)
  .set('code_region', param.code_region)
  .set('country',     param.country)
  .set('version',     version_output)
  .set('description', 'gapfill')
  .set('step',        'P09');

print('Gapfill Asset (classification)', imageFilledtnt0);

// IMPORTANT: use processingGeom (image geometry) as export region
// This preserves the full classification extent, including buffer.
Export.image.toAsset({
  image: imageFilledtnt0,
  description: imageName,
  assetId: assetOutput + imageName,
  pyramidingPolicy: { '.default': 'mode' },
  region: processingGeom,
  scale: 30,
  maxPixels: 1e13
});

var imageNameGapFill = param.country + '-' + param.code_region + '-' + version_output + '-metadata';

imageFilledYear = imageFilledYear.set('code_region', param.code_region)
  .set('country',     param.country)
  .set('version',     version_output)
  .set('description', 'gapfill metadata')
  .set('step',        'P09');

print('Gapfill metadata (year image)', imageFilledYear);

Export.image.toAsset({
  image: imageFilledYear,  
  description: imageNameGapFill,
  assetId: assetOutputMetadata + imageNameGapFill,
  pyramidingPolicy: { '.default': 'mode' },
  region: processingGeom,
  scale: 30,
  maxPixels: 1e13
});

// Export to Google Drive (optional)
if (param.ExportOpcion.exportClasifToDrive){
  Export.image.toDrive({
    image: imageFilledtnt0.select(bandNames).toInt8(),
    description: imageName + '-DRIVE',
    folder: param.ExportOpcion.DriveFolder,
    scale: 30,
    maxPixels: 1e13,
    region: processingGeom
  });
}

/**
 * Layers (preview)
 */
for (var yearI = 0; yearI < param.year.length; yearI++){
  var vis = {
    bands: ['classification_' + param.year[yearI]],
    min: 0,
    max: 62,
    palette: palettes.get('classification8'),
    format: 'png'
  };

  Map.addLayer(
    mosaic.filterMetadata('year', 'equals', param.year[yearI])
      .mosaic()
      .updateMask(regionRaster),
    {
      bands: ['swir1_median', 'nir_median', 'red_median'],
      gain: [0.08, 0.06, 0.08],
      gamma: 0.65
    },
    'mosaic-' + param.year[yearI],
    false
  );
  
  Map.addLayer(
    original,
    vis,
    'original classification ' + param.year[yearI],
    false
  );

  Map.addLayer(
    imageFilledtnt0.select(bandNames),
    vis,
    'gap fill classification ' + param.year[yearI],
    false
  );
}

Map.addLayer(
  regions.style({
    color: 'ff0000',
    fillColor: 'ff000000'
  }),
  { format: 'png' },
  'Region ' + param.code_region,
  false
);

/**
 * Function to generate coverage statistics by year and class
 */
function getAreas(image, region) {

  var pixelArea = ee.Image.pixelArea();
  
  var reducer = {
    reducer: ee.Reducer.sum(),
    geometry: region.geometry(),
    scale: 30,
    maxPixels: 1e13
  };
  
  var bandNames = image.bandNames();
  var classIds  = ee.List.sequence(0, 34);
  
  bandNames.evaluate(function(bands, error) {
    if (error) print(error.message);
    var yearsAreas = [];
    bands.forEach(function(band) {
      var year = ee.String(band).split('_').get(1),
          yearImage = image.select([band]);
      // Calculate areas for each cover class
      var covers = classIds.map(function(classId) {
        classId = ee.Number(classId).int8();
        var yearCoverImage = yearImage.eq(classId),
            coverArea      = yearCoverImage.multiply(pixelArea).divide(1e6);
        return coverArea.reduceRegion(reducer).get(band);
      }).add(year);
      // Generate the list of keys for the dictionary
      var keys = classIds.map(function(item) {
        item = ee.Number(item).int8();
        var stringItem = ee.String(item);
        stringItem = ee.Algorithms.If(
          item.lt(10),
          ee.String('ID0').cat(stringItem),
          ee.String('ID').cat(stringItem)
        );
        return ee.String(stringItem);
      }).add('year');
      // Create the list of features for each year, without geometries
      var dict = ee.Dictionary.fromLists(keys, covers);
      yearsAreas.push( ee.Feature(null, dict) );
    });
    yearsAreas = ee.FeatureCollection(yearsAreas);
    Export.table.toDrive({
      collection: yearsAreas,
      description: 'COVERAGE-STATISTICS',
      fileFormat: 'CSV',
      folder: 'P09-GapFill-CLASSIFICATION'
    });
  });
}

// Generate coverage statistics (uses official region, not buffer)
if (param.ExportOpcion.exportEstadistica){
  getAreas(imageFilledtnt0.select(bandNames), regions);
}

```
