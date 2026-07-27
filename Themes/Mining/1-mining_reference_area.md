```javascript
var param = {
  country: 'SURINAME',
  region: 80201,
  classIds: [30],
  years: [//1985, 
   //1986,1987,1988,1989,
   1990,1991,1992,1993,1994,1995,1996,1997,1998,1999,
   2000,2001,2002,2003,2004,2005,
   2006,2007,2008,2009,2010,2011,2012,
   2013,2014,2015,
   //2016,2017,2018,2019,2020,
   //2021,2022,
   //2023
   ],
  output: 'accumulated', // 'accumulated','stable'
  vectorBuffer: 120,    // 120
  outputVersion: 0,
  newAreas: null,
  exclusion: null,
  additionalMiningAreas: mining
};


var aggregate = function(param) {
  
  
  var _this = this;
  
  
  var inputs = {
    imageAsset: 'projects/mapbiomas-raisg/COLECCION6/INTEGRACION/mapbiomas_raisg_panamazonia_collection6_integration-0-2',
    referenceAsset: 'users/apoiotecnicoraisg/AllMiningAreasGuyanas',
    outputPath: 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/MASKS',
    countriesAsset: 'projects/mapbiomas-raisg/DATOS_AUXILIARES/VECTORES/paises-4',
    //regionsAsset: 'projects/mapbiomas-guianas/assets/Guyana_boundary',
    regionsAsset: 'projects/mapbiomas-suriname/assets/Archive_Suriname/classification_regions_suriname',
    sentinelAlerts: 'projects/mapbiomas-raisg/DATOS_AUXILIARES/RASTERS/sentinel-alerts',
    mosaics1: ee.ImageCollection('projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2'),
    mosaics2: ee.ImageCollection('projects/mapbiomas-raisg/MOSAICOS/mosaics-2'),
    palette: require('users/mapbiomas/modules:Palettes.js'),
    countries: {
      'VENEZUELA': 'Venezuela',
      'PERU': 'Perú',
      'COLOMBIA': 'Colombia',
      'BOLIVIA': 'Bolivia',
      'ECUADOR': 'Ecuador',
      'GUYANA': 'Guyana',
      'GUIANA_FRANCESA': 'Guiana Francesa',
      'SURINAME': 'Suriname'
    }
  };

  
  var init = function(param) {
    
    var country = param.country;
    var regionId = param.region;
    var classIds = param.classIds;
    var agg = param.aggregate;
    var years = param.years;
    var outputVersion = param.outputVersion;
    var newAreas = param.newAreas;
    var additionalMining = param.additionalMiningAreas;
    var exclusionAreas = param.exclusion;
    var boundVector = param.boundVector || true;
    var buffer = param.vectorBuffer;
    var output = param.output;
    
    var imagePath = inputs.imageAsset;
    var alertPath = inputs.sentinelAlerts;
    var mosaicsPath = inputs.mosaics1;
    var mosaicsPath2 = inputs.mosaics2;
    var countriesAsset = inputs.countriesAsset;
    var regionsAsset = inputs.regionsAsset;
    var paletteValues = inputs.palette;
    var countriesNames = inputs.countries;
    var referenceAsset = inputs.referenceAsset;
    var outputPath = inputs.outputPath;
    
    
    
    var bandNames = years
      .map(function(year) { return 'classification_' + year });
    
    var countryLayer = ee.FeatureCollection(countriesAsset)
      .filter(ee.Filter.eq('name', countriesNames[country]));
  
    var countryRaster = countryLayer
      .reduceToImage(['OBJECTID'], ee.Reducer.first());
    
    var regionLayer = ee.FeatureCollection(regionsAsset)
      .filter(ee.Filter.eq('id_regionC', regionId))
      Map.addLayer(regionLayer,{}, 'region',true);
    //print(regionLayer)
    
    var regionRaster = regionLayer
  .map(function(f){ return f.set('version', 1); })
  .reduceToImage(['version'], ee.Reducer.first());
    
    var image = ee.Image(imagePath)
      .select(bandNames)
      .updateMask(regionRaster);
      
    var sentinelAlerts = ee.Image(alertPath)
      .updateMask(regionRaster);
      
    var mosaics = mosaicsPath.merge(mosaicsPath2)
     // .filter(
    //    ee.Filter.and(
    //      ee.Filter.eq('country', country)
    //    )
     // )
      .filterBounds(regionLayer.geometry());
      
      
    // compute accumulated
    var accumulated = _this.acumulatedPixels(image, classIds);
    var stables = _this.stablePixels(image, classIds);
    
    if(exclusionAreas) {
      exclusionLayer = ee.FeatureCollection(exclusionAreas)
        .reduceToImage(['id'], ee.Reducer.first())
        .unmask(27)
        .clip(regionLayer);
  
      stables = stables.updateMask(exclusionLayer.eq(27));
      accumulated = accumulated.updateMask(exclusionLayer.eq(27));
    }
    
    var vectorAccumulated = _this.getVector(accumulated, regionLayer, boundVector, buffer);
    vectorAccumulated = newAreas ? vectorAccumulated.merge(newAreas) : vectorAccumulated;
    
    
    // compute additional mining areas
    var customMining;
    if(additionalMining) {
      customMining = ee.FeatureCollection(additionalMining)
        .reduceToImage(['id'], ee.Reducer.first());
      
      customMining = customMining.clipToBoundsAndScale({
        geometry: additionalMining.geometry(),
        scale: 30
      })
      .byte().rename('constant');
    }
    
    
    // Display to map
    Map.setOptions('SATELLITE');
    
    years.forEach(function(year) {
      var yearMosaic = mosaics
        .filter(ee.Filter.eq('year', year))
        .filterBounds(regionLayer.geometry())
        .mosaic()
        .clip(regionLayer.geometry())
        .updateMask(regionRaster);
        
      Map.addLayer(
        yearMosaic,
        { min: 200, max: 5000, bands: 'swir1_median,nir_median,red_median' },
        'MOSAIC ' + year,
        false
      );
    });

    Map.addLayer(
      sentinelAlerts, 
      { bands: ['alert'], min:0, max:4, palette: '000000,00ffff,ffe599,ff9900,ff0000'},
      'SENTINEL ALERTS 2019-2022',
      false
    );
    
    
    var palette = paletteValues.get('classification6');
    var raster;
    var mainstables;
    var vector = ee.FeatureCollection(referenceAsset);
    vector = newAreas ? vector.merge(newAreas) : vector;

    
    if(country !== 'ECUADOR' && country !== 'VENEZUELA') {
      raster = vector
        .map(function(item) { return item.set('version', 1)})
        .reduceToImage(['version'], ee.Reducer.first());
      
      if(output === 'accumulated') { accumulated = accumulated.updateMask(raster) }
      if(output === 'stable') { 
        mainstables = stables
          .updateMask(raster).byte().rename('constant');
          
        stables = additionalMining
          ? ee.ImageCollection([mainstables, customMining]).mosaic()
          : mainstables;
      }

      Map.addLayer(ee.FeatureCollection(vector.union())
        .style({color: 'cyan', fillColor:'00000000'}), {}, 'MINING SITE REFFERENCES');
    }
    
    
    if(country === 'VENEZUELA' || country === 'ECUADOR') {
      if(output === 'accumulated') { 
        vectorAccumulated = vector ? vectorAccumulated.merge(vector) : vectorAccumulated;
      }
      if(output === 'stable') { 
        mainstables = stables.byte().rename('constant');
          
        stables = additionalMining
          ? ee.ImageCollection([mainstables, customMining]).mosaic()
          : mainstables;
      }
      
    }
    
    
    if(output === 'accumulated') {
      Map.addLayer(accumulated, {palette: 'darkred'}, 'ACCUMULATED');
      Map.addLayer(vectorAccumulated.style({color: 'fff', fillColor: 'ffffff00'}), {}, 'ROI', false);
    }
    if(output === 'stable') {
      Map.addLayer(stables, {palette: 'darkred'}, 'STABLE', false);
    }
    
    
    
    
    // Export both accumulated and stables
    var firstYear = years[0];
    var lastYear = years[years.length - 1];
    var fileName = country + '-' + regionId + '-' + firstYear + '-' + lastYear  + '-' + outputVersion;
    fileName = fileName.replace('_', '-');
    var scale = 30;
    var crs = 'EPSG:4326';
    var maxPixels = 1e13;
    var region = regionLayer.geometry().bounds();
    
    if(output === 'accumulated') {
      Export.table.toAsset({
        collection: vectorAccumulated,
        description: 'ACCUMULATED-VECTOR-' + fileName,
        assetId: outputPath + '/ACCUMULATED-VECTOR-' + fileName
      });
      Export.image.toAsset({
        image: accumulated,
        description: 'ACCUMULATED-' + fileName,
        assetId: outputPath + '/ACCUMULATED-' + fileName,
        region: region,
        scale: scale,
        maxPixels: maxPixels,
        crs: crs
      });
    }
    
    if(output === 'stable') {
      Export.image.toAsset({
        image: stables,
        description: 'STABLE-' + fileName,
        assetId: outputPath + '/STABLE-' + fileName,
        region: region,
        scale: scale,
        maxPixels: maxPixels,
        crs: crs
      });
    }
    
  };
  
  
  
  
  /**
   * Stable pixels method
   */
  this.stablePixels = function(image, classes) {
  
    var bandNames = image.bandNames();
    
    var images = classes.map(function(classId) {
      
      var previousBand = image.select([bandNames.get(0)]).eq(classId);
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
      
      singleClass = singleClass.updateMask(singleClass.eq(1)).multiply(classId);
      return singleClass;
        
    });
  
    // blend all images
    var allStable = ee.Image();
    for(var i = 0; i < classes.length; i++) allStable = allStable.blend(images[i]);
  
    return allStable;
  };


  
  
  /**
   * Accumulated pixels method
   */
  this.acumulatedPixels = function(image, classIds) {
    var bands = image.bandNames();

    var single = bands.map(function(band) {
      return image
        .select([band])
        .remap(classIds, ee.List.repeat(1, classIds.length), 0);
    });
    
    var sumImage = ee.ImageCollection(single).sum();
    
    return sumImage
      .where(sumImage.gt(0), 30)
      .updateMask(sumImage.gt(0));
  };
  
  
  
  
  /**
   * Convert to vector method
   */
  this.getVector = function(image, geometry, bounds, buffer) {
    var vector = image
      .addBands(image)
      .reduceToVectors({
        geometry: geometry.geometry().bounds(),
        crs: 'EPSG:4326',
        scale: 30,
        eightConnected: false,
        labelProperty: 'mining',
        reducer: ee.Reducer.mean(),
        maxPixels: 1e13
      });
    
    if(bounds) {
      if(buffer) {
        return ee.FeatureCollection(
          vector.map(function(item) {
            return ee.Feature(item).geometry().buffer(buffer).bounds();
          })
        )
        .union(30);
      }
      else {
        return ee.FeatureCollection(
          vector.map(function(item) {
            return ee.Feature(item).geometry().bounds();
          })
        )
        .union(30);
      }
    }
    else {
      return vector;
    }
  };


  
  
  return init(param);
  
};


new aggregate(param)
```
