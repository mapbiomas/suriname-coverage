```javascript
/** 
 * STEP 03-2: GENERATION OF TRAINING SAMPLES AND SELECTION OF VARIABLES
 * January 2022
 * UPDATE FOR SURINAME AND ENGLISH VERSION 07/2025
 * joaquim.pereira@ipam.org.br
 * ----------------------------------------------------------------------------------------------
 */ 

   
 
/** 
 * ----------------------------------------------------------------------------------------------
 * USER PARAMETERS:
 * Adjust the parameters below to generate your training samples 
 * It is not recommended to modify the script in any other section.
 * ----------------------------------------------------------------------------------------------
 */
 /*--------------------------------
--- classification regions
GUYANA COAST: 50201
GUYANA COAST: 50202
GUYANA INTERIOR: 50203
GUYANA INTERIOR: 50204
GUYANA INTERIOR: 50205
GUYANA CERRADO SAVANA: 50903
GUYANA CERRADO SAVANA: 50904
SURINAME COAST: 80201
SURINAME INTERIOR: 80202
FRENCH GUIANA COAST: 60201
FRENCH GUIANA INTERIOR: 60202
*/
var param = {
  regionId: 80201,
  gridName: '',
  sampleSize: 2000, //5000
  minSamples: 1000, //1000
  yearsPreview: [2000, 2023, 2025],
  variables: [
    // spectral bands:
    
    // SMA derived bands:
    
    // indices:
  ],
  ciclo: 'ciclo-1',
  tileScale:4,// increase this value to 2,4,8,12 up to 16 if export fails
};





/**
 * ----------------------------------------------------------------------------------------------
 * APPLICATION INITIALIZATION
 * Self-invoked expression that executes step 3 of the methodology
 * ----------------------------------------------------------------------------------------------
 */
(function init(param) {


  var assets = {
    grids: 'projects/mapbiomas-raisg/DATOS_AUXILIARES/VECTORES/grid-world',
    regions:ee.FeatureCollection('projects/mapbiomas-suriname/assets/classification_regions_suriname'),
    mosaics: 'projects/mapbiomas-raisg/MOSAICOS/mosaics-2',
    mosaics22: 'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2',
    stablePixels:'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/MASKS/',
    classAreas: 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/SAMPLES/AREA/',
    outputs: 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/SAMPLES/TRAINING/'
  };
  
  var rgb = ['swir1_median', 'nir_median', 'red_median'];
  var years = param.yearsPreview;
  var grid = param.gridName;
  var regionId = param.regionId;
  
  
  // Get the output version based on the cycle
  var version = getVersion(param.ciclo);
  var vMuestras = version.version_input_muestras;
  var vAreas = version.version_input_areas;
  
  
  // Create mask based on the region vector and sheet
  var region = getRegion(assets.regions, assets.grids, regionId, grid);
  var vector = region.vector;
  
  
  // Import assets based on the region
  var mosaicPath = assets.mosaics;
  
  var mosaic = getMosaic(mosaicPath, assets.mosaics22,regionId, param.variables, grid, vector);


  var country = region.vector.first().get('pais').getInfo().toUpperCase();
  country = country .replace('Ç', 'C').replace(' ', '_');
  var countryRegion = country + '-' + regionId;

  
  var stablePixels = ee.Image(
    assets.stablePixels + 'stablePixels-' + countryRegion + '-' + vMuestras)
    .updateMask(region.rasterMask).rename('reference');


  var classAreas = ee.FeatureCollection(
    assets.classAreas + 'classArea-' +countryRegion + '-' + vAreas);
  
  
  var classAreasDictionary = classAreas.first().toDictionary();
  

  var classNames = classAreasDictionary.keys()
    .filter(ee.Filter.stringContains('item', 'ID'));
  

  // Generate training samples based on the area of each cover class
  var classIds = classNames.map(
    function(name) {
      var classId = ee.String(name).split('D').get(1);
      return ee.Number.parse(classId);
    }
  );
  

  // Calculate areas of each class and total 
  var areas = classNames.map( function(name) {
    return classAreasDictionary.get(name);
  });
  
  var totalArea = areas.reduce(ee.Reducer.sum());


  // Calculate weighted number of samples and generate training points
  var pointsPerClass = areas.map(
    function(area) {
      return getPointsByArea(
        area, totalArea, param.sampleSize, param.minSamples);
    });
  
  var training = getSamples(stablePixels, mosaic, classIds, pointsPerClass);
  var points = training.points;
  

  // Send images to the map
  //
  var rgbMosaic = getMosaic(mosaicPath,assets.mosaics22, regionId, rgb, grid, vector);
  
  addLayersToMap(points, stablePixels, rgbMosaic, years, vector);
  
  
  // Export assets and statistics to GEE and Drive
  if(grid && grid !== '') regionId = regionId + '-' + grid;
  var outputs = assets.outputs;
  var data = training.data;
  exportSamples(data, outputs, country, regionId, version.version_out);
    

  // Display information in console
  var zipped = classNames.zip(areas).zip(pointsPerClass);

  zipped = zipped.map(function(item){
    item = ee.List(item);
    var item0 = ee.List(item.get(0));
    var id = ee.String(item0.get(0)).replace('ID', 'Class '); 
    var area = ee.String(item0.get(1));
    var points = ee.String(item.get(1));
    
    return ee.String(id)
      .cat(', Area: ').cat(area)
      .cat(', Samples: ').cat(points);
  });
  
  zipped = zipped.filter(ee.Filter.stringContains('item', ': 0.0,').not());
  
  
  var samples = zipped.map(function(item){
    var points = ee.String(item).split('Samples: ');
    points = ee.List(points).get(1);
    return ee.Number.parse(points);
  });
  
  var global = ee.Dictionary.fromLists(
    ['Total area', 'Total samples: '],
    [totalArea, samples.reduce(ee.Reducer.sum())]
  );
  
  print('Stable Pixels', stablePixels);
  // print('Mosaics', mosaic);
  print('Area', classAreas.first());
  print('General Stats', global);
  print('Class Stats', zipped);
      
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
      version_input_areas: 1,
      version_input_muestras: 1,
      version_out:1
    },
    'ciclo-2': {
      // Cycle II
      version_input_areas: 2,
      version_input_muestras: 2,
      version_out:2
    }
  };
  
  return version[cicle];
}

/**
 * Global constants
 */
function ALL_YEARS() {
  return [
    1985, 1986, 1987, 1988, 1989, 1990, 1991, 1992, 1993, 1994, 1995,
    1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005, 2006, 
    2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017,
    2018, 2019, 2020, 2021, 2022, 2023, 2024, 2025
  ];
}




/**
 * Function to generate region of interest (ROI) based on
 * the classification region or a millionth grid contained in it
 */
function getRegion(regionPath, gridPath, regionId, gridName){
  
  var region = ee.FeatureCollection(regionPath)
        .filterMetadata("id_regionC", "equals", regionId);
  
  if(gridName && gridName !== '') {
    var grid = ee.FeatureCollection(gridPath)
      .filterMetadata("name", "equals", gridName)
      .first();
      
    grid = grid.set('pais', region.first().get('pais'));
    
    region = ee.FeatureCollection(ee.Feature(grid));
  }
  
  // Generate the raster
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
 * Function to filter mosaics
 * Allows filtering mosaics by region code and 250,000 grid,
 * Also manages the selection of indices that will be used to generate the
 * training points.
 */
function getMosaic(paths, paths22, regionId, variables, gridName,regionVector) {
  
  // Import altitude data
  var altitude = ee.Image('JAXA/ALOS/AW3D30_V1_1')
    .select('AVE')
    .rename('altitude');
      
  var slope = ee.Terrain.slope(altitude).int8()
    .rename('slope');
    

  // hand
  var hand30_100 = ee.ImageCollection('users/gena/global-hand/hand-100');
  var srtm = ee.Image("USGS/SRTMGL1_003");
  var hand30_1000 =  ee.Image("users/gena/GlobalHAND/30m/hand-1000");
  var hand90_1000 = ee.Image("users/gena/GlobalHAND/90m-global/hand-1000");
  var hand30_5000 = ee.Image("users/gena/GlobalHAND/30m/hand-5000");
  var fa = ee.Image("users/gena/GlobalHAND/90m-global/fa");
  var jrc = ee.Image("JRC/GSW1_0/GlobalSurfaceWater");
  var HS_fa = ee.Image("WWF/HydroSHEDS/15ACC");
  var HS_fa30 = ee.Image("WWF/HydroSHEDS/30ACC");
  var demUk = ee.Image("users/gena/HAND/test_uk_DSM");
    
  // smoothen HAND a bit, scale varies a little in the tiles
  hand30_100 = hand30_100.mosaic().focal_mean(0.1);
    
  // potential water (valleys)
  var thresholds = [0,1,2,5,10];
  var HANDm = ee.List([]);
  thresholds.map(function(th) {
    var water = hand30_100.lte(th)
      .focal_max(1)
      .focal_mode(2, 'circle', 'pixels', 5).mask(swbdMask);
      
    HANDm = HANDm.add(water.mask(water).set('hand', 'water_HAND_<_' + th + 'm'));
  });
  
  // exclude SWBD water
  var swbd = ee.Image('MODIS/MOD44W/MOD44W_005_2000_02_24').select('water_mask');
  Map.addLayer(swbd, {}, 'swbd mask', false);
  var swbdMask = swbd.unmask().not().focal_median(1);
  
  // water_hand	water (HAND < 5m)
  var HAND_water = ee.ImageCollection(HANDm);
  
  // exports.
  hand30_100  = hand30_100.rename('hand30_100');
  hand30_1000 = hand30_1000.rename('hand30_1000');
  hand30_5000 = hand30_5000.rename('hand30_5000');
  hand90_1000 = hand90_1000.rename('hand90_1000');
  HAND_water  = HAND_water.toBands()
    .rename(['water_HAND_0m', 'water_HAND_1m', 'water_HAND_2m', 'water_HAND_5m', 'water_HAND_10m']);
          
  var Hand_bands =  hand30_100
    .addBands(hand30_1000)
    .addBands(hand30_5000)
    .addBands(hand90_1000)
    .addBands(HAND_water);
                                

  // shademask2
  var shademask2 = ee.Image("projects/mapbiomas-raisg/MOSAICOS/shademask2_v3")
    .rename('shademask2');
  

  // slppost
  var slppost = ee.Image("projects/mapbiomas-raisg/MOSAICOS/slppost2_30_v3")
    .rename('slppost');
  

  
  // Manage Landsat mosaics
  var mosaicRegion = regionId.toString().slice(0, 3);
  if (mosaicRegion ==='211' ||  mosaicRegion ==='205'){mosaicRegion='210'}
  var workspace_c3_v2 = ee.ImageCollection(paths);
  var workspace_NG = ee.ImageCollection(paths22);
  
  joinedMosaics = workspace_c3_v2.filterMetadata('region_code', 'equals', parseInt(mosaicRegion));
  var jm = workspace_NG.filterMetadata('region_code', 'equals', parseInt(mosaicRegion));
  
  joinedMosaics=joinedMosaics.merge(jm)
  var joinedMosaics = joinedMosaics
    .map(function(image){
      return ee.Image
        .cat(image, altitude, slope, Hand_bands, slppost,shademask2)
        .int();
    });
    
  
  
  // select variables
  if(variables.length > 0) return joinedMosaics.select(variables);
  
  else return joinedMosaics;

}
 


/**
 * Function to calculate number of training samples based on the area
 * occupied by each class
 */
function getPointsByArea(singleArea, totalArea, sampleSize, minSamples) {
  return ee.Number(singleArea)
    .divide(totalArea)
    .multiply(sampleSize)
    .round()
    .int16()
    .max(minSamples);
}



/**
 * Function to implement the collection of points for all years in the param.year list
 * defined in the user parameters.
 */
function getSamples(stablePixels, mosaic, classIds, pointsPerClass) {
  
  var years = ee.List(ALL_YEARS());
  
  var keys = years.map( function(year) {
    var stringYear = ee.String(year);
    return ee.String('samples-').cat(stringYear);
  });
  
  var points = stablePixels
    .addBands(ee.Image.pixelLonLat())
    .stratifiedSample({
        numPoints: 0,
        classBand: 'reference',
        region: stablePixels.geometry(),
        scale: 30,
        seed: 1,
        geometries: true,
        dropNulls: true,
        classValues: classIds, 
        classPoints: pointsPerClass
    });

  var yearMosaic;
  
  var trainingSamples = years.map( function(year) {
    yearMosaic = mosaic
      .filterMetadata('year', 'equals', year)
      .mosaic();
    
    var training = stablePixels
      .addBands(yearMosaic)
      .sampleRegions({
        collection: points,
        properties: ['reference'],
        scale: 30,
        tileScale:param.tileScale,
        geometries: true
      });
    
    return training;
    
  });
  
  return {
    data: ee.Dictionary.fromLists(keys, trainingSamples),
    points: points
  };

}

/**
 * Function to send visualization to the map
 * 
 */
function addLayersToMap(training, stablePixels, mosaic, years, region) {
  // var trainingYear = ee.FeatureCollection(training.get('SAMPLES-' + year));
  var PALETTE = [
    'ffffff', '129912', '1f4423', '006400', '00ff00', '687537', '76a5af',
    '29eee4', '77a605', '935132', 'bbfcac', '45c2a5', 'b8af4f', 'f1c232', 
    'ffffb2', 'ffd966', 'f6b26b', 'f99f40', 'e974ed', 'd5a6bd', 'c27ba0',
    'fff3bf', 'ea9999', 'dd7e6b', 'aa0000', 'ff99ff', '0000ff', 'd5d5e5',
    'dd497f', 'b2ae7c', 'af2a2a', '8a2be2', '968c46', '0000ff', '4fd3ff'
  ];

  years.forEach(function(year) {
    var filtered = mosaic.filterMetadata('year', 'equals', year)
      .mosaic()
      .clip(region);
      
    Map.addLayer(
      filtered,
      {
        bands: ['swir1_median', 'nir_median', 'red_median'],
        gain: [0.08, 0.06, 0.2]
      },
      'MOSAIC ' + year.toString(), false
    );
  });

  var styledPoints = ee.FeatureCollection(training).map(
    function(point) {
      var classId = point.get('reference'),
          color = ee.List(PALETTE).get(classId);
      return point.set({ style: { color: color } });
    }
  );
  
  Map.addLayer(stablePixels, {
    min: 0,
    max: 34,
    palette: PALETTE
  }, 'STABLE PIXELS');

  Map.addLayer(
    region.style({
      fillColor: '00000066'
    }), {}, 'REGION'
  );
  
  Map.addLayer(
    styledPoints.style({
      styleProperty: "style",
      width: 1.5,
    }), {}, 'TRAINING SAMPLES'
  );
}

/**
 * Function to export the training samples as GEE assets
 */
function exportSamples(samples, outputDir, country, regionId, version) {
  var years = ALL_YEARS();
  years.forEach( function(year) {
    var sampleYear = samples.get('samples-' + year),
        yearInt = parseInt(year, 10);
    var collection = ee.FeatureCollection(sampleYear)
      .map( function(feature) {
        return feature.set('year', yearInt);
      });

    // Export samples
    var filename = 'samples-' + country + '-' + regionId + '-' + 
      year + '-'+ version;
    Export.table.toAsset(
      collection,
      filename,
      outputDir + filename
    );
    // Export.table.toDrive(collection, filename + 'DRIVE', outputDir+ filename + 'DRIVE')
  });
}
```
