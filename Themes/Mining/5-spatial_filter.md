```javascript
/*******************************
 * Spatial Filter (no country mask)
 * - Filters mosaics by region geometry
 * - Applies connected-pixel mode filter per year
 * - Exports using the same region geometry
 *******************************/

var param = { 
  code_region: 80202,          // classification region
  pais: 'SURINAME',
  tema: 'MINING',
  year: 2022,                  // visualization only
  version_input: 11112,
  version_output: 11113,
  eightConnected: true,
  min_connect_pixel: 5
}; 

// ---- Paths
var dirinput      = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/classification-ft';
var dirout        = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/classification-ft';
var AssetMosaic   = [
  'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2',
  'projects/mapbiomas-raisg/MOSAICOS/mosaics-2'
];
var AssetRegions  = 'projects/mapbiomas-suriname/assets/Archive_Suriname/classification_regions_suriname';

// ---- Region vector & mask
var region = ee.FeatureCollection(AssetRegions)
  .filterMetadata('id_regionC', 'equals', param.code_region);

var regionRaster = region
  .map(function (f) { return f.set('version', 1); })
  .reduceToImage(['version'], ee.Reducer.first());

// ---- Load classification (P04/Pxx) to be spatial-filtered
var class4FT = ee.Image(dirinput + '/' + param.pais + '-' + param.code_region + '-' + param.version_input);
print('Input classification (class4FT)', class4FT);

// ---- Year list & band names
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

var bandNames = ee.List(
  years.map(function (y) { return 'classification_' + String(y); })
);

// ---- Connected-pixel count per band (add *_connected bands)
var imageFilledConnected = class4FT.addBands(
  class4FT.connectedPixelCount(100, param.eightConnected)
    .rename(
      bandNames.map(function (b) { return ee.String(b).cat('_connected'); })
    )
);

class4FT = imageFilledConnected;

// ---- Palettes & viz
var palettes = require('users/mapbiomas/modules:Palettes.js');
var pal = palettes.get('classification2');

var visYear = {
  bands: 'classification_' + param.year,
  min: 0,
  max: 34,
  palette: pal,
  format: 'png'
};

// ---- Mosaics filtered by region geometry (not by region_code/country)
var mosaics = ee.ImageCollection(AssetMosaic[0])
  .merge(ee.ImageCollection(AssetMosaic[1]))
  .filterBounds(region.geometry())
  .filterMetadata('year', 'equals', param.year)
  .select(['swir1_median', 'nir_median', 'red_median']);

// ---- Spatial mode filter per year (masking small patches)
var class_outTotal;

// First year (1985)
var y0 = '1985';
var moda_85 = class4FT.select('classification_' + y0)
  .focal_mode(1, 'square', 'pixels');

moda_85 = moda_85.mask(
  class4FT.select('classification_' + y0 + '_connected')
    .lte(param.min_connect_pixel)
);

class_outTotal = class4FT.select('classification_' + y0).blend(moda_85);

// Remaining years
var anos = [
  '1986','1987','1988','1989','1990','1991','1992','1993',
  '1994','1995','1996','1997','1998','1999','2000','2001',
  '2002','2003','2004','2005','2006','2007','2008','2009',
  '2010','2011','2012','2013','2014','2015','2016','2017',
  '2018','2019','2020','2021','2022','2023','2024','2025'
];

for (var i_ano = 0; i_ano < anos.length; i_ano++) {
  var ano = anos[i_ano];
  var moda = class4FT.select('classification_' + ano)
    .focal_mode(1, 'square', 'pixels');

  moda = moda.mask(
    class4FT.select('classification_' + ano + '_connected')
      .lte(param.min_connect_pixel)
  );

  var class_out = class4FT.select('classification_' + ano).blend(moda);
  class_outTotal = class_outTotal.addBands(class_out);
}

// ---- Keep only classification bands, mask & metadata
class_outTotal = class_outTotal
  .select(bandNames)
  .updateMask(regionRaster)
  .set('country', param.pais)
  .set('version', param.version_output)
  .set('description', 'spatial filter')
  .set('step', 'S04-5');

print('Result (class_outTotal)', class_outTotal);

// ---- Layers (mosaic and classifications), masked by region
Map.addLayer(
  mosaics.mosaic().updateMask(regionRaster),
  {
    bands: ['swir1_median', 'nir_median', 'red_median'],
    gain: [0.08, 0.06, 0.08],
    gamma: 0.65
  },
  'mosaic-' + param.year,
  false
);

Map.addLayer(
  class4FT.select(bandNames).updateMask(regionRaster),
  visYear,
  'class-ORIGINAL ' + param.year,
  false
);

Map.addLayer(
  class_outTotal.updateMask(regionRaster),
  visYear,
  'class-SPATIAL FILTER ' + param.year,
  true
);

// ---- Export (uses the SAME REGION geometry)
var prefixo_out = param.pais + '-' + param.code_region + '-' + param.version_output;

Export.image.toAsset({
  image: class_outTotal,
  description: prefixo_out,
  assetId: dirout + '/' + prefixo_out,
  pyramidingPolicy: { '.default': 'mode' },
  region: region.geometry(),     // << use region geometry (no country)
  scale: 30,
  maxPixels: 1e13
});

  


```
