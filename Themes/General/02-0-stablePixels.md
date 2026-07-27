```javascript
/** 
 * STEP 2: CALCULATION OF STABLE PIXELS AND EXCLUSION AREAS v3.2 
 * 12/2021
 * ----------------------------------------------------------------------------------------------
 * UPDATE FOR SURINAME AND ENGLISH VERSION 07/2025
 * joaquim.pereira@ipam.org.br
 */  
 
var param = {
  regionId: 80201, //80201,80202
  yearsPreview: [ 1986,1990,2000,2018,2023],
  remap: {
    from: [3, 4, 5, 6, 9, 11, 12, 13, 14, 15, 18, 19, 20, 21, 22, 23, 24, 25, 26, 29, 30, 31, 32, 33, 34, 61],
    to:   [3, 3, 3, 3, 3, 11, 12, 12, 21, 21, 21, 21, 21, 21, 25, 25, 25, 25, 33, 25, 25, 33, 25, 33, 33, 61]
  },
  exclusion : {
    years: [
      //1987, 1989 // You can use a particular year
    ],
    classes: [ ],
    polygons: [ 
     //natural85, natural86, natural87, geometry
     Urban
    ],
    shape: '',
  },
  years:[],
  driveFolder: 'DRIVE-EXPORT',
  ciclo: 'ciclo-1'//ciclo-1 : version 1, ciclo-2 : version 2
};




/**
 * APPLICATION IMPLEMENTATION
 * Self-invoked expression that executes step 2 of the methodology
 * ----------------------------------------------------------------------------------------------
 */
(function init(param) {
  

  var assets = {
    basePath: 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/MASKS/',
    regions:  ee.FeatureCollection('projects/mapbiomas-suriname/assets/Archive_Suriname/classification_regions_suriname'),
    regionsRaster: 'projects/mapbiomas-raisg/DATOS_AUXILIARES/RASTERS/clasificacion-regiones-6',
    mosaics: 'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2', //1985-2025
    image: ee.Image('projects/mapbiomas-raisg/COLECCION6/INTEGRACION/mapbiomas_raisg_panamazonia_collection6_integration-0-2'),
  };
  
  
  // Get the output version based on the cycle
  var version = getVersion(param.ciclo);

  
  // Create mask based on the region vector
  var regionId = param.regionId;
  var region = getRegion(assets.regions, assets.regionsRaster, regionId);
  var regionMask = region.rasterMask;
    
    
  var country = region.vector.first().get('pais').getInfo().toUpperCase();
  country = country .replace('Ú', 'U').replace(' ', '_');
  var countryRegion = country + '-' + regionId;


  // Exclusion of areas
  var shapePath = assets.basePath + country + '/';
  var shapeName = param.exclusion.shape;
  var fullRegion = excludeAreas(regionMask, shapePath, shapeName);
  
  
  // Extract classification, ignoring years with inconsistencies.
  var image = ee.Image(assets.image).updateMask(fullRegion);
  image = selectBands(image, param.exclusion.years);
  print('Years used', image.bandNames());

  
  // Class remapping
  var originalClasses = param.remap.from;
  var newClasses = param.remap.to;
  image = remapBands(image, originalClasses, newClasses);
  

  // Generate stable pixels
  var classes = ee.List.sequence(1, 34);
  classes = classes.removeAll(param.exclusion.classes).getInfo();
  var stablePixels = getStablePixels(image, classes);
  

  // Exclusion of classes in areas delimited with geometries
  var polygons = param.exclusion.polygons;
  stablePixels = remapWithPolygons(stablePixels, polygons);
  
  
  // Import mosaics for visualization
  var assetsMosaics = [ assets.mosaics ];
  var variables = ['nir_median', 'swir1_median', 'red_median'];
  var mosaics = getMosaic(assetsMosaics, param.regionId, variables, '');
    

  // Show images on the map
  var assetData = {
    asset: assets.image,
    region: region,
    years: param.yearsPreview    
  };
  
  addLayersToMap(stablePixels, mosaics, assetData);


  // Export assets to GEE and Google Drive
  var imageName = 'stablePixels-'+ countryRegion + '-' + version;
  var assetId = assets.basePath + imageName;
  var driveFolder = param.driveFolder;
  var vector = region.vector;

  var props = {
    code_region: param.regionId,
    pais: country,
    version: version.toString(),
    paso: 'P02'
  };

  stablePixels = stablePixels.set(props);
  exportImage(stablePixels, imageName, assetId, vector, driveFolder);
  
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
    'ciclo-1': 1,
    'ciclo-2': 2
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
 * Function to delimit exclusion areas where no training
 * samples will be taken. 
 * These areas can be included as polygons from the drawing tools or as a collection of type
 * ee.FeatureCollection() located at the path established in the exclusion.shape parameter.
 */
function excludeAreas(image, shapePath, shapeName) {
  var exclusionRegions;
  
  var shapes = shapePath !== '' && shapeName !== '';
    
  if(shapes)
    exclusionRegions = ee.FeatureCollection(shapePath + shapeName);
  
  else exclusionRegions = null;

  
  // Exclude all defined areas
  if(exclusionRegions !== null) {
    var setVersion = function(item) { return item.set('version', 1) };
  
    exclusionRegions = exclusionRegions
      .map(setVersion)
      .reduceToImage(['version'], ee.Reducer.first())
      .eq(1);
    
    return image.where(exclusionRegions.eq(1), 0)
      .selfMask();
  } 
  else return image;
}
    



/**
 * Function to interactively remap areas delimited by polygons
 * These polygons are drawn with the GEE drawing tools
 * and are defined as ee.FeatureCollection()
 */
function remapWithPolygons(stablePixels, polygons) {
  
  if(polygons.length > 0) {
    polygons.forEach(function( polygon ) {
      
      var excluded = polygon.map(function( layer ){
        
        var area = stablePixels.clip( layer );
        var from = ee.String(layer.get('original')).split(',');
        var to = ee.String(layer.get('new')).split(',');
        
        from = from.map( function( item ){
          return ee.Number.parse( item );
        });
        to = to.map(function(item){
          return ee.Number.parse( item );
        });
        
        return area.remap(from, to);
      });
        
      excluded = ee.ImageCollection( excluded ).mosaic();
      stablePixels = excluded.unmask( stablePixels ).rename('reference');
      stablePixels = stablePixels.mask( stablePixels.neq(27) );
    });
  } else stablePixels = stablePixels;
  
  return stablePixels;
  
}




/**
 * Function to select the bands based on the years defined in
 * the parameters
 */
function selectBands(image, years) {
  var bandNames = [];
  
  years.forEach(function(year) {
    bandNames.push('classification_' + year);
  });
  
  return ee.Image(
    ee.Algorithms.If(
      years.length === 0, 
      image, 
      image.select(image.bandNames().removeAll(bandNames))
    )  
  );
}




/**
 * Function to generate region of interest (ROI) based on
 * the classification region or a millionth grid contained in it
 */
function getRegion(regionPath, regionImagePath, regionId){
  
  var region = ee.FeatureCollection(regionPath)
    .filterMetadata("id_regionC", "equals", regionId);
  
  //var regionMask = ee.Image(regionImagePath).eq(regionId).selfMask();
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
 * Also manages the selection of indices to be used to generate the
 * training points.
 */
function getMosaic(paths, regionId, variables, gridName) {
  
  // Import altitude data
  var altitude = ee.Image('JAXA/ALOS/AW3D30_V1_1')
    .select('AVE')
    .rename('altitude');
      
  var slope = ee.Terrain.slope(altitude).int8()
    .rename('slope');
  
  // Manage Landsat mosaics
  var mosaicRegion = regionId.toString().slice(0, 3);
  
  if (mosaicRegion ==='211' ){mosaicRegion='210'  }
  //if (mosaicRegion ==='211' ){mosaicRegion='210'  }
  var mosaics = paths.map( function(path) {
    
    var mosaic = ee.ImageCollection(path)
      .filterMetadata('region_code', 'equals', parseInt(mosaicRegion))
      .map(function(image) {
        var index = ee.String(image.get('system:index')).slice(0, -3);
        return image.set('index', index);
      });
    
    if(gridName && gridName !== '')
      mosaic = mosaic
        .filterMetadata('grid_name', 'equals', gridName);
    else
      mosaic = mosaic;
    
    if(mosaic.size().getInfo() !== 0) return mosaic;
    
  });
  
  
  mosaics = mosaics.filter( function(m) { return m !== undefined });
    
  var joinedMosaics = mosaics[0];

  if(mosaics.length === 2) {

    var join = ee.Join.inner(),
        joiner = ee.Filter.equals({
          leftField: 'index',
          rightField: 'index'
        });
        
    var joinedCollection = join.apply(mosaics[0], mosaics[1], joiner);
    
    joinedMosaics = ee.ImageCollection(
      joinedCollection.map( function(feature) {
        var primary = feature.get('primary'),
            secondary = feature.get('secondary');
            
        return ee.Image.cat(primary, secondary, altitude, slope);
      })
    );
  }
  
  // select variables
  if(variables.length > 0) return joinedMosaics.select(variables);
  
  else return joinedMosaics;

}




/**
 * Function for extraction of stable pixels
 * This function takes two parameters. The classification image and the classes that
 * you want to obtain as output
 */
function getStablePixels(image, classes) {
  
  var bandNames = image.bandNames(),
      images = [];

  classes.forEach(function(classId){
      var previousBand = image
        .select([bandNames.get(0)]).eq(classId);
          
      var singleClass = ee.Image(
        bandNames.slice(1)
          .iterate(
            function( bandName, previousBand ) {
              bandName = ee.String( bandName );
              return image
                .select(bandName).eq(classId)
                .multiply(previousBand);
            },
            previousBand
          )
      );
      
      singleClass = singleClass
        .updateMask(singleClass.eq(1))
        .multiply(classId);
      
      images.push(singleClass);
  });
  
  
  // blend all images
  var allStable = ee.Image();
  
  for(var i = 0; i < classes.length; i++) 
    allStable = allStable.blend(images[i]);

  return allStable;
} 




/**
 * Function to plot results on the map
 */
function addLayersToMap(stablePixels, mosaics, originalImage) {
  
  var palette = require('users/mapbiomas/modules:Palettes.js')
    .get('classification8');
    
  var region = originalImage.region;
    
  var image = ee.Image(originalImage.asset)
    .updateMask(region.rasterMask);
    
  var bands;
  
  if(originalImage.years.length === 0) {
    bands = image.bandNames();
  } 
  else {
    bands = ee.List([]);
    originalImage.years.forEach(function(year){
      bands = bands.add('classification_' + year.toString());
    });
  }
  
  bands.evaluate(function(bandnames){

    bandnames.forEach(function(bandname){
      
      // Mosaics
      var year = parseInt(bandname.split('_')[1], 10);
      
      var mosaic = mosaics.filterMetadata('year', 'equals', year)
        .mosaic()
        .updateMask(region.rasterMask);
        
      Map.addLayer(
        mosaic,
        {
          bands: ['swir1_median', 'nir_median', 'red_median'],
          gain: [0.08, 0.06, 0.2]
        },
        'MOSAIC ' + year.toString(), false
      );

      // Classifications
      Map.addLayer(
        image,
        {
          bands: bandname,
          min: 0, max: 62,
          palette: palette
        },
        bandname.toUpperCase().replace('TION_', 'CION ')
      );
      
    });
    
    
    // Region
    Map.addLayer(region.vector.style({
      fillColor: '00000066', color: 'FCBA03'
    }), {}, 'REGION ' + param.regionId);
    
    
    // Stable pixels
    Map.addLayer(
      stablePixels,
      {
        min: 0,
        max: 62,
        palette: palette
      },
      'Stable Pixels'
    );

  });
}




/**
 * Functions to export results to GEE and Drive
 */
function exportImage(image, imageName, imageId, region, driveFolder) {
  Export.image.toAsset({
    image: image.toInt8(),
    description: imageName,
    assetId: imageId,
    scale: 30,
    pyramidingPolicy: {
      '.default': 'mode'
    },
    maxPixels: 1e13,
    region: region.geometry().bounds()
  });
  
  if(driveFolder !== '' && driveFolder !== undefined) {
    Export.image.toDrive({
      image: image.toInt8(),
      description: imageName + '-DRIVE',
      folder: driveFolder,
      scale: 30,
      maxPixels: 1e13,
      region: region.geometry().bounds()
    });
  }
}
```
