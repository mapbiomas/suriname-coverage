```javascript
var param = {
  country: 'GUYANA',
  regions: [50202],
  roi: 'AllMiningAreasGuyanas',
  stables: 'STABLE-GUYANA-50202-1990-2020-0',
  accumulated: 'ACCUMULATED-GUYANA-50202-1985-2023-0',
  points: {
    mining: 5000,
    nomining: 5000
  },
  years: [
    //1985,
    1986,1987,1988,1989,1990,1991,1992,1993,1994,1995,1996,1997,1998,1999,2000,
    2001,2002,2003,2004,2005,2006,2007,2008,2009,2010,2011,2012,2013,2014,2015,2016,
    2017,2018, 2019, 2020, 2021,2022,2023,2024, 2025],
  outputVersion: 1
};





var Points = function(param) {
  
  var _this = this;
  
  
  var assets = {
    mosaics1: ee.ImageCollection('projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2'),
    mosaics2: ee.ImageCollection('projects/mapbiomas-raisg/MOSAICOS/mosaics-2'),
    mainMiningAreas: ee.FeatureCollection('users/apoiotecnicoraisg/' + param.roi),
    newMiningAreas: ee.FeatureCollection('users/apoiotecnicoraisg/' + param.roi),
    stable: ee.Image('projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/MASKS/' + param.stables),
    unstable: ee.Image('projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/MASKS/' + param.accumulated),
    palette: require('users/mapbiomas/modules:Palettes.js').get('classification6'),
    regions: ee.FeatureCollection('projects/mapbiomas-guianas/assets/Guyana_boundary'),
  };


  /**
   * Inint application
   */
  var init = function(param) {
    
    // parameters
    var country = param.country;
    var regionIds = param.regions;
    var years = param.years;
    var miningPoints = param.points.mining;
    var noMiningPoints = param.points.nomining;
    var outputVersion = param.outputVersion;
    var basePath = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/SAMPLES/';
    
    var region = assets.regions.filter(ee.Filter.inList('id_regionC', regionIds));
    var regionImage = region.reduceToImage(['id_regionC'], ee.Reducer.first());
    var fullRoi = assets.mainMiningAreas.merge(assets.newMiningAreas);
    var reference = _this.setWorkingRegion(fullRoi, assets.stable, assets.unstable);
    var mosaics = assets.mosaics1.merge(assets.mosaics2)
    .filter(ee.Filter.bounds(region));
    
    // geometry points
    var points = reference
      .addBands(ee.Image.pixelLonLat())
      .stratifiedSample({
          numPoints: 0,
          classBand: 'reference',
          region: region,
          scale: 30,
          seed: 1,
          geometries: true,
          dropNulls: true,
          classValues: [30, 27], 
          classPoints: [miningPoints, noMiningPoints]
      });
    
    // assign mosaic data to geometry points and export to assets
    years.forEach( function(year) {
      var yearMosaic = mosaics
        .filterMetadata('year', 'equals', year)
        .mosaic()
        .updateMask(regionImage);
      
      var training = reference
        .addBands(yearMosaic)
        .sampleRegions({
          collection: points,
          properties: ['reference'],
          scale: 30,
          geometries: true
        });
        
      
      Map.addLayer(
        yearMosaic,
        { min: 200, max: 5000, bands: 'swir1_median,nir_median,red_median' },
        'MOSAIC ' + year,
        false
      );
  
      var outputDir = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/SAMPLES/';
      var filename = country + '-' + regionIds.join('-') + '-' + year + '-' + outputVersion;
      Export.table.toAsset(training, filename, outputDir + filename);
    });
    
    Map.setOptions('SATELLITE');
    Map.addLayer(region.style({color: 'white', fillColor: '00000000'}), {}, 'REGIONS');
    Map.addLayer(reference, {min: 0, max: 49, palette: assets.palette}, 'REFERENCE', false);
    Map.addLayer(points, {}, 'SAMPLES', false);
    Map.addLayer(fullRoi.style({color: 'cyan', fillColor: '00000000'}), {}, 'MINING AREAS');
  };
  
  
  
  /**
   * Set the region for generating points
   */
  this.setWorkingRegion = function(roi, stable, unstable) {
    
    var stableAreas = stable.clip(roi);
    var unstableAreas = unstable.clip(roi);
    
    var workingRegion = ee.Image(27)
      .clip(roi)
      .where(unstableAreas, 0)
      .where(stableAreas, 30);
    
    return workingRegion.updateMask(workingRegion.neq(0)).rename('reference');
  };
  
  
  return init(param);
};


new Points(param);
```
