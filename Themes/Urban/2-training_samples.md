```javascript
// training for classification of urban areas col5
// date: 14/06/2023

var param = { 
  regionId: 80201, // region de clasificacion de urbana
  country: "SURINAME",
  yearsPreview: [2019, 2020, 2023, 2024],
  samples: [
    1000,              // Urban 
    1000               // no-Urban 
  ],
  version_out: '1',  // Version de samples a exportar
  remapStablePixel : {
    polygons: [to_urban, to_no_urban, remap_27_to_0],  // para REMAP pixeles estables OJO; Urbana:24, No urbana:27, No Observado:0 
  },
  
  classificationArea : {   // Para Incluir o exluir el Area de clasificacion con el Buffer 
      MaskArea :{
        col: 'COLECCION5', // COLECCION4, COLECCION5
        versionClassArea: '1',   // Area de clasificacion con el Buffer  //COLECCION 4
         },
        // versionClassArea: '1',   // Area de clasificacion con el Buffer  //COLECCION 4
        inclusion: inclusion,
        exclusion: exclusion,
     // Para importar shp o geometria guardada de ciclos anteriores
      shapefile : {
        useshp: false,   // Usar geometrias ya creadas en pasos posteriores usado para ciclo2
        col: 'COLECCION5', // COLECCION4, COLECCION5
        step: 'step3',  // 'step2', 'step3'
        shpVersion: '1',
             }
        },
  exportassets: {
    geometries : false   // Para exportar inclusion y exclusion de classificationArea
  }
};

/**
 * Años a procesar
 */
var years = [
     1985, 1986, 1987, 1988, 1989, 
     1990, 1991, 1992, 1993, 1994, 
     1995, 1996, 1997, 1998, 1999, 
     2000, 2001, 2002, 2003, 2004, 
     2005, 2006, 2007, 2008, 2009, 
     2010, 2011, 2012, 2013, 2014, 
     2015, 2016, 2017, 2018, 2019, 
     2020, 2021, 2022, 2023, 2024
  ]
  
  
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

var SampleUrban = function(param) {

/**
 * Input data
 * Assets paths, years and another necessary input data
 */
this.inputs = {
  mosaics: [
            'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2',
            'projects/mapbiomas-raisg/MOSAICOS/mosaics-2'
          ],
  _regions: 'projects/mapbiomas-suriname/assets/classification_regions_suriname',
  _samples: 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/SAMPLES/',
  _clasificacionArea: 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/MASKS/URBAN-REF-ACCUM-'+ param.regionId + '-' + param.country + '-' + param.classificationArea.shapefile.shpVersion,

  STEP3_GEOMETRY: 'projects/mapbiomas-raisg/MUESTRAS/'+param.country+'/COLECCION5/TRANSVERSALES/URBANA/STEP3_GEOMETRY/urban-'+ param.regionId + '-' + param.country + '-' + param.classificationArea.shapefile.shpVersion,
  STEP2_GEOMETRY: 'projects/mapbiomas-raisg/MUESTRAS/'+param.country+'/COLECCION5/TRANSVERSALES/URBANA/STEP2_GEOMETRY/urban-'+ param.regionId + '-' + param.country + '-' + param.classificationArea.shapefile.shpVersion,
  
  references: {
      coleccion3: 'projects/mapbiomas-raisg/public/collection3/mapbiomas_raisg_panamazonia_collection3_integration_v2',
      coleccion4: 'projects/mapbiomas-raisg/COLECCION6/INTEGRACION/mapbiomas_raisg_panamazonia_collection6_integration-0-2',
      urbana_all_ref: 'projects/mapbiomas-raisg/MOSAICOS/Urbana_all_ref_v1',
      peru: 'projects/mapbiomas-public/assets/peru/collection1/mapbiomas_peru_collection1_integration_v1',
      colombia: '',
      ecuador: '',
      bolivia: 'projects/mapbiomas-raisg/PRODUCTOS/BOLIVIA/COLECCION4/INTEGRACION/integracion-pais/BOLIVIA-4',
      guyanas: '',
      venezuela: '',
  },
  palette: require('users/mapbiomas/modules:Palettes.js').get('classification2')
};


/**
 * Initialize the app
 */
this.init = function(param) {
  var _this = this;
  var assetMosaics = _this.inputs.mosaics;
  var assetclassArea = 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/MASKS/URBAN-REF-ACCUM-'+ param.country + '-' + param.regionId + '-' + param.classificationArea.shapefile.shpVersion;
  var assetcoleccion3 = _this.inputs.references.coleccion3;
  var assetcoleccion4 = 'projects/mapbiomas-raisg/COLECCION6/INTEGRACION/mapbiomas_raisg_panamazonia_collection6_integration-0-2';
  var PeruCol1 = _this.inputs.references.peru;

  var urbana_all_ref = _this.inputs.references.urbana_all_ref;

  var regionAsset = _this.inputs._regions;
  var samplesAsset = _this.inputs._samples;
  var palette = _this.inputs.palette;

    // Create mask based on region vector
  var regionId = param.regionId;
  var yearsPreview = param.yearsPreview;
  var nSamples = param.samples;
  
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
  
  var region = _this._getRegion(regionAsset, regionId);
  var regionMask = region.rasterMask;
  var regionLayer = region.vector;
  Map.addLayer(regionLayer,{},'regionLayer')
  
  var country = param.country.split(' ').join('-').toUpperCase();
  var version_out = param.version_out
  
  // Get mosaics
  var mosaics = _this.getMosaic(assetMosaics, regionId);
  print('mosaics',mosaics.size());

  // Get stable pixels from collection 4
    // var collection4 = ee.Image(assetcoleccion4)
    //                     .updateMask(regionMask);
                        
    // var Per_col1 = ee.Image(PeruCol1)
    
    // collection4 = collection4.blend(Per_col1)
    var collection4;
    if (param.country ==='PERU'){
      collection4 =ee.Image('projects/mapbiomas-raisg/PRODUCTOS/PERU/COLECCION4/INTEGRACION/integracion-pais/Integracion-PERU-6')
                   .updateMask(regionMask)
    }
    if (param.country ==='BOLIVIA'){
      collection4 =ee.Image('projects/mapbiomas-raisg/PRODUCTOS/BOLIVIA/COLECCION4/INTEGRACION/integracion-pais/BOLIVIA-4')
                   .updateMask(regionMask)
    } else {
      collection4 = ee.Image(assetcoleccion4)
                      .updateMask(regionMask)
    }
  
  var classes = ee.List.sequence(1, 34).getInfo();
  var stablePixels = _this.getStablePixels(collection4, classes);
  
  print(stablePixels);

  // Urban sampling points
  var colorId, stableReference;    
  var nodata = stablePixels.neq(24);  // revisar
  var urban  = stablePixels.eq(24);  // revisar
  stableReference = ee.Image(0).updateMask(regionMask);

  
  stableReference = stableReference
      .where(urban.eq(1), 24)
      .where(nodata.eq(1), 27)
      .updateMask(regionMask)
      .rename("reference");
  
  
 // Exclusión de clases con areas delimitadas con geometrías
  var polygons = param.remapStablePixel.polygons;
  stableReference = _this.remapWithPolygons(stableReference, polygons);
  
  
  assetclassArea = assetclassArea.replace('COLECCION5', param.classificationArea.MaskArea.col)
  print('assetclassArea',assetclassArea)
  var classArea = ee.Image(assetclassArea).updateMask(regionMask)
  classArea = _this.inclus_exclu(classArea, param.classificationArea.inclusion, param.classificationArea.exclusion);
  
  if(param.classificationArea.shapefile.useshp){
  var inclusionSHP = ee.FeatureCollection(AssetSHPIncluExclu).filter(ee.Filter.eq('type', 'inclusion'));
  var exclusionSHP = ee.FeatureCollection(AssetSHPIncluExclu).filter(ee.Filter.eq('type', 'exclusion'));
  classArea = _this.inclus_exclu(classArea, inclusionSHP, exclusionSHP);
  Map.addLayer(inclusionSHP,{},'inclusionSHP',false)
  Map.addLayer(exclusionSHP,{},'exclusionSHP',false)
 }
  var GeometriesIE = ee.FeatureCollection([param.classificationArea.inclusion, param.classificationArea.exclusion]).flatten()
 
    // print('GeometriesIE',GeometriesIE)
    
  stableReference = stableReference.updateMask(classArea);
  
  var points = stableReference
    .addBands(ee.Image.pixelLonLat())
    .stratifiedSample({
        numPoints: 0,
        classBand: 'reference',
        region: regionLayer.geometry().bounds(),
        scale: 30,
        seed: 1,
        geometries: true,
        dropNulls: true,
        classValues: [24, 27], 
        classPoints: [ nSamples[1], nSamples[0] ]
  });
  
 print('Points class 24:',points.filter(ee.Filter.eq('reference', 24)).size());
 print('Points class 27:',points.filter(ee.Filter.eq('reference', 27)).size());
 
  //iterate by years
  Map.setOptions('SATELLITE');
  
  // Terrain
    var dem = ee.Image('JAXA/ALOS/AW3D30_V1_1').select("AVE");  
    var slope = ee.Terrain.slope(dem).rename('slope');
    var slppost = ee.Image('projects/mapbiomas-raisg/MOSAICOS/slppost2_30_v3').rename('slppost')
    var shadeMask2 = ee.Image("projects/mapbiomas-raisg/MOSAICOS/shademask2_v3").rename('shade_mask2')
    var urbana_all_ref_raster = ee.Image(urbana_all_ref)
    
  var SamplesList = ee.List([]);
  years.forEach(function(year){
        
      var mosaic = mosaics
          .filter(ee.Filter.eq('year', Number(year)))
          .filterBounds(regionLayer)
          .median()
          .addBands(dem.rename('elevation'))
          .addBands(slope)
          .addBands(slppost)
          .addBands(shadeMask2)
          .updateMask(regionMask);
      
      mosaic = _this.newIndex(mosaic, urbana_all_ref_raster, 1)
      
      var mosaicSel = mosaic.updateMask(mosaic.select('blue_median')).select(featureSpace).updateMask(classArea);
      print('MOSAICO'+year,mosaicSel.bandNames())
      var trainingSamples = _this.getSamples(stableReference, mosaicSel, points);
      var training = trainingSamples.training;
      
      SamplesList = SamplesList.add(training.map(function(feature){
                  return feature.set('year', year);
                }));
                                  
      // Export samples to asset
      // var fileName = 'Urban' + '-' + regionId + '-' + country + '-' + year + '-' + version_out;
      // var assetId = samplesAsset + fileName;
      
      // Export.table.toAsset(training, fileName, assetId);

      if(yearsPreview.indexOf(year) > -1) {
        
          Map.addLayer(
            mosaic, 
            {
              bands: ['swir1_median', 'nir_median', 'red_median'],
              min: 200,
              max: 5000
              // gain: [0.08, 0.06, 0.2]
            }, 
            'MOSAICO ' + year,
            false
          );
          Map.addLayer(
            mosaic.select('nuaci_median2').updateMask(regionMask), 
            {
              bands: ['nuaci_median2'],
              min:0,"max":200,
              palette:["504eff","1ffff4","32ff23","fff71d","ffc31d","ff0000"]
            }, 
            'nuaci_median2 ' + year,
            false
          );
      }  

  });
  
  SamplesList = ee.FeatureCollection(SamplesList).flatten()
  // print(SamplesList.limit(2))
  // print('SamplesList',SamplesList.size())
  
  // Export samples to asset
  var fileName = 'urban' + '-' + regionId + '-' + country  + '-' + version_out;
  var assetId = samplesAsset + fileName;
  Export.table.toAsset(SamplesList, fileName, assetId);
  
  
  var assetIdG = 'projects/mapbiomas-raisg/MUESTRAS/'+country+'/COLECCION5/TRANSVERSALES/URBANA/STEP2_GEOMETRY/' + fileName;

  if(param.exportassets.geometries){
    
     if(classificationArea.shapefile.useshp === false) {
      // GeometriesIE= ee.FeatureCollection(AssetSHPIncluExclu).merge(GeometriesIE)
      Export.table.toAsset(GeometriesIE, 'geom-'+fileName, assetIdG);
      } else {
      GeometriesIE= ee.FeatureCollection(AssetSHPIncluExclu).merge(GeometriesIE)
      Export.table.toAsset(GeometriesIE, 'geom-'+fileName, assetIdG);
       }
      
    }
  
  Map.addLayer(classArea,
       {
        palette: 'fcff00'
       },'classArea',false
      );
  Map.addLayer(stableReference,
       {
        min: 0,
        max: 34,
        palette: _this.inputs.palette
       },'stableReference',false
      );

  var pts = ee.FeatureCollection(points);

  // Layers
    var eeColors = ee.List(_this.inputs.palette);
    
    var trainingPointsColor = pts.map(
        function (feature) {
    
            var c = feature.get("reference");
    
            return feature.set({
                "style": {
                    "color": eeColors.get(c),
                    "pointSize": 4
                }
            });
        }
    );
    
    Map.addLayer(trainingPointsColor.style({
        "styleProperty": "style"
    }), {}, 'points',false);
};



/**
 * Get mosaics
 * Get mosaics from collection2 asset. Then compute
 * Urbans indexes remaining.
 */
this.getMosaic = function(paths, regionId) {

    // Additional variables
   // var shademask2_v1 = shademask2_v1.rename('shade_mask2')
    
    var mosaicRegion = regionId.toString().slice(0, 3);
    // añadimos una excepcion para Bolivia
    if (regionId===21101 || regionId===21102){mosaicRegion='210'}
    
    var mosaics = paths.map( function(path) {
            
            var mosaic = ee.ImageCollection(path)
              .filter(ee.Filter.eq('region_code', Number(mosaicRegion)))
              .map(function(image) {
                var index = ee.String(image.get('system:index')).slice(0, -3);
                return image
                  .set('index', index);
              });
            
            if(mosaic.size().getInfo() !== 0) return mosaic;
            
          });
          
    mosaics = mosaics.filter( 
            function(m) { return m !== undefined }
          );
    
    // var joinedMosaics = mosaics[0];
    var joinedMosaics = mosaics[0].merge(mosaics[1]);
    // if(mosaics.length === 2) {
        
    //         var join = ee.Join.inner(),
    //             joiner = ee.Filter.equals({
    //               leftField: 'index',
    //               rightField: 'index'
    //             });
                
    //         var joinedCollection = join.apply(mosaics[0], mosaics[1], joiner);
            
    //         joinedMosaics = ee.ImageCollection(
    //           joinedCollection.map( function(feature) {
    //             var primary = feature.get('primary'),
    //                 secondary = feature.get('secondary');
                    
    //             return ee.Image.cat(primary, secondary);
    //           })
    //         );
    //       }
          
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
 * Get stable pixels
 * Get stable pixels from mapbiomas collection 2
 * Then cross over reference datasets 
 */
this.getStablePixels = function (image, classes) {
  
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
};


/**
 * Get reference raster data
 */
this.getRasterReference = function(inputs) {
  
  var setVersion = function(item) { return item.set('version', 1) };
  
  // Colombia
  var colombia = ee.FeatureCollection(inputs.colombia)
    .reduceToImage(['ID'], ee.Reducer.first());
  
  var colombiaMask = colombia
    .where(colombia.eq(0), 6).where(colombia.eq(1), 27)
    .rename('reference')
    .uint8();
  
  
  // Ecuador
  var ecuador = ee.FeatureCollection(inputs.ecuador)
    .reduceToImage(['CODIGO'], ee.Reducer.first());
  
  var ecuadorMask = ecuador
    .where(ecuador.eq(1), 6).where(ecuador.eq(2), 27)
    .rename('reference')
    .uint8();
  
  
  // Peru
  var peru =  ee.FeatureCollection(inputs.peru)
    .reduceToImage(['CODIGO'], ee.Reducer.first());
    
  var peruMask = peru
    .where(peru.eq(1), 6).where(peru.eq(2), 27)
    .rename('reference')
    .uint8();

  
  // Bolivia
  var bolivia = ee.FeatureCollection(inputs.bolivia)
    .reduceToImage(['CODIGO'], ee.Reducer.first());
  
  var boliviaMask = bolivia
    .where(bolivia.eq(1), 6).where(bolivia.eq(2), 27)
    .rename('reference')
    .uint8();

  // guyanas
  var guyanas = ee.Image(inputs.guyanas);
  
  var guyanasMask = guyanas.where(guyanas.gt(20), 6)
    .rename('reference')
    .uint8();
    

  // Join all
  var flooded = ee.ImageCollection([
    colombiaMask, boliviaMask, ecuadorMask, peruMask, guyanasMask
  ]);
  
  return flooded.mosaic();
  
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
 * Función para delimitar áreas de excusión en las que no se tomarán 
 * muestra de entrenamiento. 
 * Estas áreas pueden incluirse como polígonos desde las herramientas de 
 * dibujo o como una colección de tipo ee.FeatureCollection() ubicada en la ruta
 * establecida en el parámetro exclusion.shape.
 */
this.excludeAreas = function(image, shapePath, shapeName) {
  var exclusionRegions;
  
  var shapes = shapePath !== '' && shapeName !== '';
    
  if(shapes)
    exclusionRegions = ee.FeatureCollection(shapePath + shapeName);
  
  else exclusionRegions = null;

  
  // Excluir todas las areas definidas
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
 * Función para remapear, de manera interactiva, zonas delimitadas por polígonos
 * Estos polígonos se dibujan con las herramientas de dibujo de GEE
 * y se definen como ee.FeatureCollection()
 */
this.remapWithPolygons = function(stablePixels, polygons) {
  
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
      stablePixels = stablePixels.mask( stablePixels.neq(0) );
    });
  } else stablePixels = stablePixels;
  
  return stablePixels;
  
}



/**
 * Get sample points
 */
this.getSamples = function(reference, mosaic, points) {
  
    var training = reference
      .addBands(mosaic)
      .sampleRegions({
          collection: points,
          properties: ['reference'],
          scale: 30,
          geometries: true,
          tileScale: 4
    });
    
    return {
      points: points, 
      training: training 
    };
    
};

return this.init(param);

};


var Samples = new SampleUrban(param);
```
