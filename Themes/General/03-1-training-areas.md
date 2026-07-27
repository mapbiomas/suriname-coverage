```javascript
/** 
 * STEP 3: CALCULATION OF AREAS TO SELECT TRAINING SAMPLES
 * SEPTEMBER 2020
 * ----------------------------------------------------------------------------------------------
 * UPDATE FOR SURINAME AND ENGLISH VERSION 07/2025
 * joaquim.pereira@ipam.org.br
 */
  
/** 
 * USER PARAMETERS:
 * Adjust the parameters below to generate the corresponding stable pixels image
 * ----------------------------------------------------------------------------------------------
 */

var param = {
  regionId: 80201,// 80201,80202
  referenceYear: '',   // reference year or '' for stable map
  years:[2022],
  remap: {
    from: [3, 4, 5, 6, 9, 11, 12, 13, 14, 15, 18, 19, 20, 21, 22, 23, 24, 25, 26, 29, 30, 31, 32, 33, 34, 61],
    to:   [3, 3, 3, 3, 3, 11, 12, 12, 21, 21, 21, 21, 21, 21, 25, 25, 25, 25, 33, 25, 25, 33, 25, 33, 33, 61]
  },
  
  driveFolder: 'DRIVE-EXPORT',
  ciclo: 'ciclo-1'
};



/**
 * ----------------------------------------------------------------------------------------------
 * APPLICATION INITIALIZATION
 * Self-invoked expression that executes step 3 of the methodology
 * ----------------------------------------------------------------------------------------------
 */
(function init(param) {
  

  var assets = {
    trainingAreas: 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/SAMPLES/AREA/',
    regions: ee.FeatureCollection('projects/mapbiomas-suriname/assets/classification_regions_suriname'),
    referenceImage:  ee.Image('projects/mapbiomas-raisg/COLECCION6/INTEGRACION/mapbiomas_raisg_panamazonia_collection6_integration-0-2'),
    stablePixels: 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/MASKS/'
  };

  var palette = require('users/mapbiomas/modules:Palettes.js')
    .get('classification8');
  var version = getVersion(param.ciclo);
  
  var region = getRegion(assets.regions, assets.regionsRaster, param.regionId);
  var rasterMask = region.rasterMask;

  var country = region.vector.first().get('pais').getInfo().toUpperCase()
      .replace('Ç', 'C')
      .replace(' ', '_');
      
  var baseName = country + '-' + param.regionId + '-' + 
        version.inputVPixelesEstables.toString();
    
  //print('basename', baseName)      
  
  var classes = ee.List.sequence(1, 34).getInfo();
  
  var reference, updtReference;
  
  if(param.referenceYear){
    // Year selection for balancing 
    reference = ee.Image(assets.referenceImage)
      .select('classification_' + param.referenceYear.toString())
      .updateMask(rasterMask);

    // Class remapping
    var originalClasses = param.remap.from;
    var newClasses = param.remap.to;
    updtReference = remapBands(reference, originalClasses, newClasses);
  }
  else{
    updtReference = ee.Image(assets.stablePixels + 'stablePixels-' + baseName);
  }
  
  var areas = getAreas(updtReference, classes, region.vector);
  
  print('Class area layer', areas);


  // Show reference layer on the map
  Map.addLayer(updtReference, {
    min: 0,
    max: 62,
    palette: palette
  },
  'Stable Areas');
  
  

  // Export statistics to Google Drive
  var tableName = 'classArea-'+ country + '-' + param.regionId + '-' + 
    version.outputVCalcArea.toString();

  print('Direction out', assets.trainingAreas);
  
  print('Table Name', tableName);
  
  exportFeatures(
    areas, 
    tableName, 
    assets.trainingAreas + tableName,
    param.driveFolder
  );

})(param);





/**
 * FUNCTIONALITIES
 * Below are defined the functionalities that are used in the application.
 * These features are injected into the init() function which executes them and generates the
 * results.
 * ----------------------------------------------------------------------------------------------
 */
 
/**
 * Function to assign a version by cycle
 * 
 */
function getVersion(cicle) { 
  var version = {
    'ciclo-1': {
      // Cycle I
      inputVPixelesEstables: 1,
      outputVCalcArea: 1,
    },
    'ciclo-2': {
      // Cycle II
      inputVPixelesEstables: 2,
      outputVCalcArea: 2
    }
  };
  
  return version[cicle];
}


/**
 * Function to remap (reclassify) classified bands
 * In the order of execution, this function runs before remapping with polygons
 */
function remapBands(image, originalClasses, newClasses) {
  var bandNames = image.bandNames().getInfo();
  var collectionList = ee.List([]);
  
  bandNames.forEach(
    function( bandName ) {
      var remapped = image.select(bandName)
        .remap(originalClasses, newClasses);
    
      collectionList = collectionList.add(remapped.int8().rename(bandName));
    }
  );
  var collectionRemap = ee.ImageCollection(collectionList);
  image = collectionRemap.toBands();
  

  
  var actualBandNames = image.bandNames();
  var singleClass = actualBandNames.slice(1)
    .iterate(
      function( bandName, previousBand ) {
        bandName = ee.String(bandName);
                
        previousBand = ee.Image(previousBand);

        return previousBand.addBands(image
          .select(bandName)
          .rename(ee.String('classification_')
          .cat(bandName.split('_').get(2))));
      },
      ee.Image(image.select([actualBandNames.get(0)])
          .rename(ee.String('classification_')
          .cat(ee.String(actualBandNames.get(0)).split('_').get(2))))
    );
  return ee.Image(singleClass);
}



/**
 * Function to calculate areas (in Km2) per class, based on the stable
 * pixels image.
 */
function getAreas(image, classes, region){
  
  var reducer = {
      reducer: ee.Reducer.sum(),
      geometry: region.geometry(), 
      scale: 30,
      maxPixels: 1e13
  };
  
  var propFilter = ee.Filter.neq('item', 'OBJECTID');
  var propFilter2 = ee.Filter.neq('item', 'OBJECTID_1');
  
  classes.forEach( function( classId, i ) {
      var imageArea = ee.Image.pixelArea()
        .divide(1e6)
        .mask(image.eq(classId))
        .reduceRegion(reducer);
      
      var area = ee.Number(imageArea.get('area')).round();
          
      region = region.map(function(item){
        var props = item.propertyNames();
        var selectProperties = props.filter(propFilter)
                               .filter(propFilter2);
        
        return item
          .select(selectProperties)
          .set('ID' + classId.toString(), area);
      });
      
      return region;
  });
  
  return region;
  
}




/**
 * Function to generate region of interest (ROI) based on
 * the classification region or a millionth grid contained in it
 */
function getRegion(regionPath, regionImagePath, regionId){
  
  var region = ee.FeatureCollection(regionPath)
    .filterMetadata("id_regionC", "equals", regionId);
  
  // var regionMask = ee.Image(regionImagePath).eq(regionId).selfMask();
  var setVersion = function(item) { return item.set('version', 1) };
  var regionMask = region
    .map(setVersion)
    .reduceToImage(['version'], ee.Reducer.first());
    
  return {
    vector: region,
    rasterMask: regionMask
  };

}



/**
 * Function to export the calculated areas as GEE assets
 */
function exportFeatures(features, tableName, tableId, driveFolder) {
  print('Class Area export',features)
  Export.table.toAsset({
    collection: features, 
    description: tableName,
    assetId: tableId,
  });
  
  var featuresTable = ee.FeatureCollection([
    ee.Feature(null, features.first().toDictionary())
  ]);
  
  if(driveFolder !== '' && driveFolder) {
    Export.table.toDrive({
      collection: featuresTable, 
      description: tableName + '-DRIVE',
      folder: driveFolder,
      fileFormat: 'CSV',
    });
  }
}
```
