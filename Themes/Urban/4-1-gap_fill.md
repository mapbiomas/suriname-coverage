```javascript
var param = {
    code_region: 80202,
    country: 'SURINAME',
    year: 1985,  // visualization  
    version_input: '1',
    version_output: '2', // don`t change
   }
  
// input
var AssetUrbanClass = 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/classification'
var OutputAsset = 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/classification-ft/'

var AssetMosaic='projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2'
var AssetMosaic2='projects/mapbiomas-raisg/MOSAICOS/mosaics-2'

var AssetRegions = 'projects/mapbiomas-suriname/assets/classification_regions_suriname';
// var AssetRegionsRaster = 'projects/mapbiomas-raisg/DATOS_AUXILIARES/RASTERS/clasificacion-regiones-3';
 
var palettes = require('users/mapbiomas/modules:Palettes.js');
var eePalettes = require('users/gena/packages:palettes');


var regions = ee.FeatureCollection(AssetRegions)
                .filterMetadata('id_regionC', "equals", param.code_region);
var mosaicRegion = param.code_region.toString().slice(0, 3);
// var regionRaster = ee.Image(AssetRegionsRaster).eq(param.code_region).selfMask()
if (mosaicRegion=='211' || mosaicRegion=='205'){mosaicRegion='210'}
var setVersion = function(item) { return item.set('version', 1) };
var regionRaster = regions
                      .map(setVersion)
                      .reduceToImage(['version'], ee.Reducer.first());
                      
                      

var collect_mosaic = ee.ImageCollection(AssetMosaic).merge(ee.ImageCollection(AssetMosaic2))
            .filterMetadata('region_code', 'equals', Number(mosaicRegion))
            .select(['swir1_median', 'nir_median', 'red_median']);
            
var mosaic = collect_mosaic.filterMetadata('year', 'equals', param.year);


var Clasificacion = ee.ImageCollection(AssetUrbanClass)
                      .filterMetadata('region', 'equals', param.code_region)
                      .filterMetadata('version', 'equals', param.version_input)
                      .filterMetadata('metodo', 'equals', 'Random forest')
                      .mosaic()
print(Clasificacion)


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
    2021, 2022, 2023, 2024
    ];
    
Map.addLayer(Clasificacion,
            {},'Urban-original'+'-v1',false)


// GAP FILL

/**
 * User defined functions
 */
var applyGapFill = function (image) {

    // apply the gap fill form t0 until tn
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

    // apply the gap fill form tn until t0
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


// Obtiene la version de salida en base al ciclo
var version_input = param.version_input;
var version_output = param.version_output;

var image = Clasificacion;


// get band names list 
var bandNames = ee.List(
    years.map(
        function (year) {
            return 'classification_' + String(year);
        }
    )
);

//--- identi pixel mask
var classif = ee.Image();
var bandnameReg = image.bandNames();
bandnameReg.getInfo().forEach(
  function (bandName) {
    var imagey = image.select(bandName)
    var band0 = imagey
    // var band0 = imagey.where(imagey.eq(27), 27)
    //                   .where(imagey.eq(24), 24); //.updateMask(imagey.unmask().neq(27))
    var year = bandName.toString().slice(-4)
    // print(year)
    var mosaic = collect_mosaic
                            .select(['swir1_median'])
                            .filterMetadata('year', 'equals', ee.Number.parse(year, 10))
                            .mosaic();
    band0 = imagey.updateMask(mosaic.unmask().neq(0))
    
    classif = classif.addBands(band0.rename(bandName))
  }
)

image =classif.select(bandnameReg);

// generate a histogram dictionary of [bandNames, image.bandNames()]
var bandsOccurrence = ee.Dictionary(
    bandNames.cat(image.bandNames()).reduce(ee.Reducer.frequencyHistogram())
);

print(bandsOccurrence);
 
// insert a masked band 
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

// convert dictionary to image
var imageAllBands = ee.Image(
    bandNames.iterate(
        function (band, image) {
            return ee.Image(image).addBands(bandsDictionary.get(ee.String(band)));
        },
        ee.Image().select()
    )
);


// generate image pixel years
var imagePixelYear = ee.Image.constant(years)
    .updateMask(imageAllBands)
    .rename(bandNames);

// apply the gap fill
var imageFilledtnt0 = applyGapFill(imageAllBands);
var imageFilledYear = applyGapFill(imagePixelYear);



/**
* Export images to asset
*/
var imageName = param.country + '-' + param.code_region + '-' + version_output;

imageFilledtnt0 = imageFilledtnt0.select(bandNames)
                  .set('code_region', param.code_region)
                  .set('country', param.country)
                  .set('version', version_output)
                  .set('description', 'gapfill')
                  .set('step', 'S04-1')
                  
print(imageFilledtnt0);

Export.image.toAsset({
    'image': imageFilledtnt0,
    'description': imageName,
    'assetId':  OutputAsset + imageName,
    'pyramidingPolicy': {
        '.default': 'mode'
    },
    'region': regions.geometry().bounds(),
    'scale': 30,
    'maxPixels': 1e13
});

var imageNameGapFill = param.country + '-' + param.code_region + '-' + version_output + '-metadata';

Export.image.toAsset({
    'image': imageFilledYear
        .set('code_region', param.code_region)
        .set('country', param.country)
        .set('version', version_output)
        .set('description', 'gapfill metadata')
        .set('step', 'S04-1'),
    'description': imageNameGapFill,
    'assetId': OutputAsset + imageNameGapFill,
    'pyramidingPolicy': {
        '.default': 'mode'
    },
    'region': regions.geometry().bounds(),
    'scale': 30,
    'maxPixels': 1e13
});
/**
* Layers
*/

var vis = {
    'bands': ['classification_' + param.year],
    'min': 0,
    'max': 34,
    'palette': palettes.get('classification2'),
    'format': 'png'
};

Map.addLayer(mosaic.mosaic().updateMask(regionRaster), {
      'bands': ['swir1_median', 'nir_median', 'red_median'],
      'gain': [0.08, 0.06, 0.08],
      'gamma': 0.65
  }, 'mosaic-'+param.year, false);
  
Map.addLayer(
    image,
    vis,
    'classification original remap-' + param.year);

Map.addLayer(
    imageFilledtnt0,
    vis,
    'classification gap fill ' + param.year);

Map.addLayer(imageFilledYear,
    {
        'bands': ['classification_' + param.year],
        'min': 1985,
        'max': 2024,
        'palette': eePalettes.colorbrewer.YlOrBr[9],
        'format': 'png'
    },
    'image metadata',false
);

Map.addLayer(
    regions.style({
        "color": "ff0000",
        "fillColor": "ff000000"
    }),
    {
        "format": "png"
    },
    'Region ' + param.code_region,
    false);
```
