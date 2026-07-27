```javasccript
var param = { 
    code_region: 80202,  //Region de Clasificacion 
    country: 'SURINAME', 
    year: 2015,  // Solo visualizacion
    version_input: '5',
    version_output: '6',
    eightConnected: true,
    min_connect_pixel: 5
}; 


var dirinput = 'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/classification-ft'
var dirout  =   'projects/mapbiomas-suriname/assets/URBAN/COLLECTION-1/classification-ft'
var AssetMosaic='projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2'
var AssetMosaic2='projects/mapbiomas-raisg/MOSAICOS/mosaics-2'
var AssetRegions = 'projects/mapbiomas-suriname/assets/classification_regions_suriname';
// var AssetRegionsRaster = 'projects/mapbiomas-raisg/DATOS_AUXILIARES/RASTERS/clasificacion-regiones-3';

var prefixo_out = param.country+ '-' + param.code_region + '-' + param.version_output

var region = ee.FeatureCollection(AssetRegions).filterMetadata('id_regionC', 'equals', param.code_region)
// var regionRaster = ee.Image(AssetRegionsRaster).eq(param.code_region).selfMask()
var setVersion = function(item) { return item.set('version', 1) };
var regionRaster = region
                      .map(setVersion)
                      .reduceToImage(['version'], ee.Reducer.first());
                      
var palettes = require('users/mapbiomas/modules:Palettes.js');
var vis = {
    'min': 0,
    'max': 34,
    'palette': palettes.get('classification2')
};

var mosaicRegion = param.code_region.toString().slice(0, 3);
var collect_mosaic = ee.ImageCollection(AssetMosaic).merge(ee.ImageCollection(AssetMosaic2))
            .filterMetadata('region_code', 'equals', Number(mosaicRegion))
            .select(['swir1_median', 'nir_median', 'red_median']);
            
var mosaic = collect_mosaic.filterMetadata('year', 'equals', param.year);
            
            
var class4FT = ee.ImageCollection(dirinput)
                      .filterMetadata('code_region', 'equals', param.code_region)
                      // .filterMetadata('paso', 'equals', 'S04-3') 
                      .filterMetadata('version', 'equals', param.version_input)
                      .first()
print(class4FT)


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
    2021, 2022, 2023, 2024];    
    
// get band names list 
var bandNames = ee.List(
    years.map(
        function (year) {
            return 'classification_' + String(year);
        }
    )
);

// add connected pixels bands
var imageFilledConnected = class4FT.addBands(
    class4FT
        .connectedPixelCount(100, param.eightConnected)
        .rename(bandNames.map(
            function (band) {
                return ee.String(band).cat('_connected')
            }
        ))
);

print(imageFilledConnected)
class4FT= imageFilledConnected;

var palettes = require('users/mapbiomas/modules:Palettes.js');
var pal = palettes.get('classification2');
var vis = {
      bands: 'classification_1985',
      min:0,
      max:34,
      palette: pal,
      format: 'png'
    };
var vis2 = {
      min:0,
      max:34,
      palette: pal,
      format: 'png'
    };
var vis3 = {
      bands: 'classification_'+param.year,
      min:0,
      max:34,
      palette: pal,
      format: 'png'
    };
// Map.addLayer(class4FT.select(bandNames), vis, 'class4FT_1985', false);

var ano = '1985'
var moda_85 = class4FT.select('classification_'+ano).focal_mode(1, 'square', 'pixels')
moda_85 = moda_85.mask(class4FT.select('classification_'+ano+ '_connected').lte(param.min_connect_pixel))
var class_outTotal = class4FT.select('classification_'+ano).blend(moda_85)

// Map.addLayer(class_outTotal, vis, 'class4 MODA_1985', false);

var anos = ['1986','1987','1988','1989','1990','1991','1992','1993',
            '1994','1995','1996','1997','1998','1999','2000','2001',
            '2002','2003','2004','2005','2006','2007','2008','2009',
            '2010','2011','2012','2013','2014','2015','2016','2017',
            '2018','2019','2020','2021','2022','2023','2024'];
            
for (var i_ano=0;i_ano<anos.length; i_ano++){  
  var ano = anos[i_ano]; 
  var moda = class4FT.select('classification_'+ano).focal_mode(1, 'square', 'pixels')
  moda = moda.mask(class4FT.select('classification_'+ano+ '_connected').lte(param.min_connect_pixel))
  var class_out = class4FT.select('classification_'+ano).blend(moda)
  class_outTotal = class_outTotal.addBands(class_out)
}


class_outTotal = class_outTotal.select(bandNames).updateMask(regionRaster)
                          .set('code_region', param.code_region)
                          .set('country', param.country)
                          .set('version', param.version_output)
                          .set('description', 'spatial filter')
                          .set('step', 'S04-5');
            
print('Result', class_outTotal)

Map.addLayer(mosaic.mosaic().updateMask(regionRaster), {
      'bands': ['swir1_median', 'nir_median', 'red_median'],
      'gain': [0.08, 0.06, 0.08],
      'gamma': 0.65
  }, 'mosaic-'+param.year, false);
  
Map.addLayer(class4FT.select(bandNames), vis3, 'class-ORIGINAL'+param.year);
Map.addLayer(class_outTotal, vis3, 'class-SPATIAL FILTER'+param.year);

Export.image.toAsset({
    'image': class_outTotal,
    'description': prefixo_out,
    'assetId': dirout+'/'+ prefixo_out,
    'pyramidingPolicy': {
        '.default': 'mode'
    },
    'region': region.geometry().bounds(),
    'scale': 30,
    'maxPixels': 1e13
});
```
