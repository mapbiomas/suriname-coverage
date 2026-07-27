```javascript
var exclusion = exclusion;
// training for classification of urban areas COL5
// date: 14/06/2023
            
var param = {
  region: 80202,
  country: 'SURINAME',
  trees: 50,
  yearsPreview: [2023],
  _print: true,
  tileScale: 8,
  inputVersionSample: '1',
  outVersionClass: '1',
  GlobalSurfaceWater: 1,
  additionalSamples: {
    polygons: [ geometry_24, geometry_27],
    // polygons: [ ],
    classes: [ 24, 27 ], // clases en orden de las geometrias
    points: [100, 100]   //Numero de muestras a agregar en cada clase
  },
  additionalSamplesNoStable: { // MUESTRAS NO ESTABLES Muestra24_1985_1990, Muestra27_1985_1990, Muestra27_1985
    polygons: [Muestra24_1985_1990],  // El Numero de Muestras no estables es el total de pixeles que interceptan con el mosaico (no se hace sorteo)
  },
  classificationArea : {   // Para Incluir o exluir el Area de clasificacion con el Buffer 
      MaskArea :{
        col: 'COLECCION5', // COLECCION4, COLECCION5
        versionClassArea: '1',   // Area de clasificacion con el Buffer  //COLECCION 4
         },
        inclusion: inclusion1,
        exclusion: exclusion,
      shapefile : {
        useshp: false,   // Usar geometrias ya creadas en pasos posteriores usado para ciclo2
        col: 'COLECCION5', // COLECCION4, COLECCION5
        step: 'step3',  // 'step2', 'step3'
        shpVersion: '1',
             }
  }
};

// featureSpace
var featureSpace = [
  "blue_median",
  "green_median",
  "red_median",
  "red_wet",
  "nir_median",
  "nir_wet",
  "swir1_median",
  "swir2_median",
  "ndvi_median",
  "ndvi_wet",
  "wefi_wet",
  "gcvi_wet",
  "sefi_median",
  "soil_median",
  "snow_median",
  "evi2_median",
  "ndwi_mcfeeters_median",
  "mndwi_median",
  "slope",
  "slppost",
  "elevation",
  "shade_mask2",
  "nuaci_median2"
];


var Urban = function(param){
  
  this.param = param;
  
  this.inputs = {
    mosaics: [
            'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2',
            'projects/mapbiomas-raisg/MOSAICOS/mosaics-2'
    ],
    _regions : 'projects/mapbiomas-suriname/assets/classification_regions_suriname',
    _clasificacionArea: 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/MASKS/URBAN-REF-ACCUM-'+ param.country + '-' + param.region + '-' + param.classificationArea.shapefile.shpVersion,
    samples: 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/SAMPLES/',
    result: 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/classification/',
    urbana_all_ref: 'projects/mapbiomas-raisg/MOSAICOS/Urbana_all_ref_v2',
    STEP3_GEOMETRY: 'projects/mapbiomas-raisg/MUESTRAS/'+param.country+'/COLECCION5/TRANSVERSALES/URBANA/STEP3_GEOMETRY/urban-'+ param.region + '-' + param.country + '-' + param.classificationArea.shapefile.shpVersion,
    STEP2_GEOMETRY: 'projects/mapbiomas-raisg/MUESTRAS/'+param.country+'/COLECCION5/TRANSVERSALES/URBANA/STEP2_GEOMETRY/urban-'+ param.region + '-' + param.country + '-' + param.classificationArea.shapefile.shpVersion,
    years: [
       1985, 1986, 1987, 1988, 
       1989, 1990, 1991, 1992, 
       1993, 1994, 1995, 1996, 
       1997, 1998, 1999, 2000, 
       2001, 2002, 2003, 2004, 
       2005, 2006, 2007, 2008, 
       2009, 2010, 2011, 2012, 
       2013, 2014, 2015, 2016, 
       2017, 2018, 2019, 2020,
       2021, 2022, 2023, 2024
    ],
    palette: require('users/mapbiomas/modules:Palettes.js').get('classification2')
  };
  
  
  
  this.init = function(param){
    
    // Set satellite as default view
    Map.setOptions({
      mapTypeId: 'SATELLITE'
    });
    
    // Inputs and parms
    var _this = this;
    var regionId = param.region;
    var regionAsset = _this.inputs._regions;
    var country = param.country;
    var thesGSW = param.GlobalSurfaceWater
    var assetclassArea = _this.inputs._clasificacionArea;
    var urbana_all_ref = _this.inputs.urbana_all_ref;
    
    
    var additionalSamples = param.additionalSamples;
    
    var variables = featureSpace;
    var trees = param.trees;
    var _print = param._print;
    var version_input = param.inputVersionSample.toString();
    var version_out = param.outVersionClass.toString();
    var palette = _this.inputs.palette;
    
    var samplesPath = _this.inputs.samples;
    var assetMosaics = _this.inputs.mosaics;
    var years = _this.inputs.years;
    var outputPath = _this.inputs.result;
    
    
    // GEOMETRIAS de Inclusion y exclusion
    var AssetSHPIncluExclu;
    if(param.classificationArea.shapefile.step == 'step2')
    {
      AssetSHPIncluExclu = _this.inputs.STEP2_GEOMETRY 
      
    } else if(param.classificationArea.shapefile.step == 'step3'){
      AssetSHPIncluExclu = _this.inputs.STEP3_GEOMETRY
    }
    
  AssetSHPIncluExclu = AssetSHPIncluExclu.replace('COLECCION5', param.classificationArea.shapefile.col)
  print('AssetSHPIncluExclu',AssetSHPIncluExclu)
    
    // Region
    var region = _this._getRegion(regionAsset, regionId);
    var regionMask = region.rasterMask;
    var regionLayer = region.vector;
      print('Region Layer', regionLayer)
    Map.addLayer(regionLayer,{}, 'region',true)

    // Mosaics
    //var mosaic = this.getMosaic( assetMosaics, regionId);
    // Mosaics (agora usando a geometria da região)
var mosaic = this.getMosaic(assetMosaics, regionLayer.geometry());

    print('Mosaic Size',mosaic.size())
    
    assetclassArea = assetclassArea.replace('COLECCION5', param.classificationArea.MaskArea.col)
     print('assetclassArea',assetclassArea)
    var classArea = ee.Image(assetclassArea).updateMask(regionMask)
        // classArea = _this.inclus_exclu(classArea, param.classificationArea.inclusion, param.classificationArea.exclusion);
    
    if(param.classificationArea.shapefile.useshp){
      var inclusionSHP = ee.FeatureCollection(AssetSHPIncluExclu).filter(ee.Filter.eq('type', 'inclusion'));
      var exclusionSHP = ee.FeatureCollection(AssetSHPIncluExclu).filter(ee.Filter.eq('type', 'exclusion'));
      classArea = _this.inclus_exclu(classArea, inclusionSHP, exclusionSHP);
      Map.addLayer(inclusionSHP,{},'inclusionSHP',false)
      Map.addLayer(exclusionSHP,{},'exclusionSHP',false)
    }
     
    classArea = _this.inclus_exclu(classArea, param.classificationArea.inclusion, param.classificationArea.exclusion);

    // All-years training polygons
    var samplesAsset = 'urban-' + regionId + '-' + country + '-' + version_input;
    var trainingSamples = ee.FeatureCollection(samplesPath + samplesAsset);
    
    // Define classifier
    var classifier = ee.Classifier.smileRandomForest({
        numberOfTrees: trees, 
        variablesPerSplit: 1
    });
    
    // Terrain
    var dem = ee.Image('JAXA/ALOS/AW3D30_V1_1').select("AVE");  
    var slope = ee.Terrain.slope(dem).rename('slope');
    var slppost = ee.Image('projects/mapbiomas-raisg/MOSAICOS/slppost2_30_v3').rename('slppost')
    var shadeMask2 = ee.Image("projects/mapbiomas-raisg/MOSAICOS/shademask2_v3").rename('shade_mask2')
    var water = ee.Image("JRC/GSW1_2/GlobalSurfaceWater")
              .select('occurrence')
              .gte(thesGSW)
    
    var classifiedImage = ee.Image().byte();
    
     var geom = ee.FeatureCollection(
        regionLayer.geometry().bounds()
      )
      .map(function(item) {
        return item.set('version', 1);
      })
      .reduceToImage(['version'], ee.Reducer.first());
    var urbana_all_ref_raster = ee.Image(urbana_all_ref)
    
    // merges all polygons
    var regionsNoStableSample = ee.FeatureCollection(param.additionalSamplesNoStable.polygons).flatten()
    var yearst0 = regionsNoStableSample.aggregate_min('t0').getInfo();
    var yearst1 = regionsNoStableSample.aggregate_max('t1').getInfo();
    print(yearst0, yearst1)
    // Iterate by years
    years.forEach(function(year) {
      
      // Mosaics
      var yearMosaic = mosaic.filter(ee.Filter.eq('year', year))
                            .filterBounds(regionLayer)
                            .median()
                            .addBands(dem.rename('elevation'))
                            .addBands(slope)
                            .addBands(slppost)
                            .addBands(shadeMask2)
                            .updateMask(regionMask);
                            
      yearMosaic = _this.newIndex(yearMosaic, urbana_all_ref_raster, 1)    
       
      var yearMosaicSel = yearMosaic.select(featureSpace)
      
    // if(useOSM){
    //     yearMosaic = yearMosaic.updateMask(OSM_buffer)
    // }
      yearMosaicSel = yearMosaicSel.updateMask(yearMosaicSel.select('blue_median')).updateMask(classArea);
      
      
      Map.addLayer(yearMosaic, {
          bands: ['swir1_median', 'nir_median', 'red_median'],
          gain: [0.08, 0.06, 0.2]
        }, 
        'MOSAICO ' + year, 
        false
      );
      

      // Samples
      var yearSamples = trainingSamples.filterMetadata('year', 'equals', year)
        .map(function(feature){
          return _this.removeProperty(feature, 'year');
        });
        
      // Here we put additional samples
      if(additionalSamples.polygons.length > 0){
        
        var insidePolygons = ee.FeatureCollection(additionalSamples.polygons)
          .flatten()
          .reduceToImage(['id'], ee.Reducer.first());
        
        var outsidePolygons = insidePolygons.mask().eq(0).selfMask();
        outsidePolygons = geom.updateMask(outsidePolygons);
  
        
        var outsideVector = outsidePolygons.reduceToVectors({
          reducer: ee.Reducer.countEvery(),
          geometry: regionLayer.geometry().bounds(),
          scale: 30,
          maxPixels: 1e13
        });
  
        
        var newSamples = _this.resampleCover(yearMosaicSel, additionalSamples);
        
        
        yearSamples = yearSamples.filterBounds(outsideVector)
                                 .merge(newSamples);
      }
          
      // Here we put additional samples no stable
      if(param.additionalSamplesNoStable.polygons.length > 0 & year >= yearst0 & year <= yearst1){
         // filter by user defined region "userRegion" if exists
        var PolygonsYear = regionsNoStableSample
                              .filterBounds(regionLayer)
                              .filter(ee.Filter.and(ee.Filter.lte('t0', year), 
                                                    ee.Filter.gte('t1', year)
                              ));
                              
        var newNoStableSamples =  yearMosaicSel.sampleRegions(PolygonsYear, ['reference'], 30, null, 4)
        
        print(year)
        // Map.addLayer(newNoStableSamples,{},'addNssample',false)
        print(newNoStableSamples.aggregate_histogram('reference'))
        yearSamples = yearSamples.merge(newNoStableSamples);
      }
          
      // Classification
      var classified = _this.classifyRandomForests(
        yearMosaicSel, classifier, yearSamples
      );
      
      var name = 'classification_' + year.toString();
      
      classified = classified.where(water.eq(1), 27);
      classifiedImage = classifiedImage.addBands(classified.rename(name));

      
      
      // Display and exports
      Map.addLayer(classified.rename(name).updateMask(classArea), {
          min: 0, 
          max: 34,
          palette: _this.inputs.palette
        }, 
        'CLASIFICACION ' + year, 
        false
      );
      
      
    });
      Map.addLayer(classArea,
       {
        palette: 'fcff00'
       },'classArea',false
      );
    // Export image to asset
    var siteName = samplesAsset.toUpperCase() + '-RF-' + version_out;

    classifiedImage = classifiedImage.slice(1).updateMask(classArea).byte()
      .set({
        region: regionId,
        country: country,
        metodo: 'Random forest',
        version: version_out
      });
      
    if(_print) print(classifiedImage);

    Export.image.toAsset({
      image: classifiedImage,
      description: siteName,
      assetId: outputPath + siteName,
      region: regionLayer.geometry().bounds(),
      scale: 30,
      maxPixels: 1e13,
      pyramidingPolicy: {'.default': 'mode'},
    });
    
  };
  
  /**
 * Inclu exclu
 */
  this.inclus_exclu = function(capa, inclu, exclu){
             var inclusionRegions=  ee.FeatureCollection(inclu).reduceToImage(['value'], ee.Reducer.first())
                           .eq(1)
             var exclusionRegions=  ee.FeatureCollection(exclu).reduceToImage(['value'], ee.Reducer.first())
                           .eq(1)
             capa = capa.where(exclusionRegions.eq(1), 0).selfMask()        
             capa = ee.Image(0).where(capa.eq(1), 1)
                               .where(inclusionRegions.eq(1), 1).selfMask()
                               
      return capa
    };
    
    /**
   * Function for taking additional samples
   */
  this.resampleCover = function(mosaic, additionalSamples) {
    
    var polygons = additionalSamples.polygons,
        classIds = additionalSamples.classes,
        points = additionalSamples.points,
        newSamples = [];
    
    polygons.forEach(function(polygon, i) {
      
      var newSample = mosaic.sample({
        numPixels: points[ i ],
        region: polygon,
        scale: 30,
        projection: 'EPSG:4326',
        seed: 1,
        geometries: true,
        tileScale:param.tileScale
      })
      .map(function(item) { return item.set('reference', classIds[ i ]) });
      
      newSamples.push(newSample);
  
    });
    
    return ee.FeatureCollection(newSamples).flatten();
  
  };

  /**
   * Get mosaics
   * Get mosaics from collection2 asset. Then compute
   * Urbans indexes remaining.
   */
 /**
 * Carrega mosaicos por interseção espacial com a geometria da região
 * (sem depender de region_code/país).
 */
this.getMosaic = function(paths, regionGeometry) {
  // Carrega e filtra cada coleção por bounds
  var m1 = ee.ImageCollection(paths[0])
              .filterBounds(regionGeometry)
              .map(function(image) {
                var index = ee.String(image.get('system:index')).slice(0, -3);
                return image.set('index', index);
              });

  var m2 = ee.ImageCollection(paths[1])
              .filterBounds(regionGeometry)
              .map(function(image) {
                var index = ee.String(image.get('system:index')).slice(0, -3);
                return image.set('index', index);
              });

  // Une as duas coleções (se uma vier vazia, o merge ainda funciona)
  var joinedMosaics = m1.merge(m2);

  // (opcional) restringe aos anos de interesse se você quiser otimizar:
  // joinedMosaics = joinedMosaics.filter(ee.Filter.inList('year', this.inputs.years));

  return joinedMosaics;
};

  /**
 * Get new indexes
 * Get new index from images. Then compute
 * Urbans indexes .
 */
  this.newIndex = function(image, urbana_all_ref, threshold) {

      var uNTL = urbana_all_ref.gte(threshold)
      var ndvi = image.expression(
          'float(nir - red)/(nir + red)', {
              'nir': image.select('nir_median'),
              'red': image.select('red_median'),
          });

      var ndwi = image.expression(
          'float(swir1 - green)/(swir1 + green)', {
              'swir1': image.select('swir1_median'),
              'green': image.select('green_median'),
          });

      var ndbi = image.expression(
        'float(swir1 - nir)/(swir1 + nir)', {
          'nir': image.select('nir_median'),
          'swir1': image.select('swir1_median')
         });
      var nuaci = image.expression(
        'float(uNTL) * (1.0 - sqrt(pow((NDWI + 0.05), 2) + (pow((NDVI + 0.1), 2) + (pow((NDBI + 0.1), 2)))))', 
        {
          'uNTL': uNTL,
          'NDVI': ndvi,
          'NDBI': ndbi,
          'NDWI': ndwi
        }).multiply(100).add(100).byte().rename(["nuaci_median2"]);
     
     nuaci = ee.Image(1).where(nuaci, nuaci).rename(["nuaci_median2"]);


  return image.addBands(nuaci.select([0], ['nuaci_median2']));
    
};


  /**
 * Función para generar region de interés (ROI) con base en
 * las región de clasificación o una grilla millonésima contenida en ella
 */
  this._getRegion = function(regionPath, regionIds){
  
    var setVersion = function(item) { return item.set('version', 1) };
    
    var region = ee.FeatureCollection(regionPath)
                   .filter(ee.Filter.eq('id_regionC', regionIds));
    
    var regionMask = region
      .map(setVersion)
      .reduceToImage(['version'], ee.Reducer.first());
      
    return {
      vector: region,
      rasterMask: regionMask
    };
  
 };
  
  /**
   * RandomForests classifier
   */
  this.classifyRandomForests = function(mosaic, classifier, samples) {

    var bands = mosaic.bandNames();
    
    var nBands = bands.size();
    
    var points = samples.size();
    
    var nClassSamples = samples
      .reduceColumns(ee.Reducer.toList(), ['reference'])
      .get('list');
      
      
    nClassSamples = ee.List(nClassSamples)
      .reduce(ee.Reducer.countDistinct());
    
    
    var _classifier = ee.Classifier(
      ee.Algorithms.If(
        ee.Algorithms.IsEqual(nBands, 0),
        null, 
        ee.Algorithms.If(
          ee.Algorithms.IsEqual(nClassSamples, 1),
          null,
          classifier.train(samples, 'reference', bands)
        )
      )
    );

    var classified = ee.Image(
      ee.Algorithms.If(
        ee.Algorithms.IsEqual(points, 0),
        ee.Image().rename('classification'),
        ee.Algorithms.If(
          ee.Algorithms.IsEqual(nBands, 0),
          ee.Image().rename('classification'),
          ee.Algorithms.If(
            ee.Algorithms.IsEqual(nClassSamples, 1),
            ee.Image().rename('classification'),
            mosaic.classify(_classifier)
          )
        )
      )
    ).unmask(27).toByte();
    

    classified = classified
      .where(classified.neq(24), 27)
      .where(classified.eq(24), 24);

    
    return classified;
    
  };
  

  /**
   * utils methods
   */
  this.setVersion = function(item){ return item.set('version', 1) };
  
  
  
  this.removeProperty = function(feature, property) {
    var properties = feature.propertyNames();
    var selectProperties = properties.filter(ee.Filter.neq('item', property));
    return feature.select(selectProperties);
  };
  
  
  
  return this.init(param);
  
};


var Urban = new Urban(param);
```
