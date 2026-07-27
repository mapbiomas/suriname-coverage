```javascript
var param = {
  country: 'SURINAME',
  input: 'SURINAME-80202-11111',
  output: 'SURINAME-80202-11112',
  inputCollection: 'classification',
  regionIds: [ 80202 ],
  years: [
    1985, 1986, 1987, 1988, 1989, 1990, 1991, 1992, 1993, 1994, 1995, 1996,
    1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005, 2006, 2007, 2008,
    2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017, 2018, 2019, 2020,
    2021, 2022, 2023, 2024, 2025
  ],
  yearsPreview: [ 2000, 2015, 2025],
  version: 2
};





// params
var country         = param.country;
var input           = param.input;
var output          = param.output;
var inputCollection = param.inputCollection;
var regionIds       = param.regionIds;
//var regionCodes     = regionIds.map(function(item){ return item.toString() });

var yearsPreview    = param.yearsPreview;
var optionalFilters = param.optionalFilters;
var years           = param.years;
var version         = param.version;

var palette         = require('users/mapbiomas/modules:Palettes.js').get('classification2');
palette[1]          = palette[6];
var vis             = { min: 0, max: 34, palette: palette, format: 'png'};



// inputs
var outputDir   = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/classification-ft/';
var regionsPath = 'projects/mapbiomas-suriname/assets/Archive_Suriname/classification_regions_suriname';//change to French Guiana when necessary
var inputPath   = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/classification/';
var mosaicsPath = 'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2';
var mosaicsPath2 = 'projects/mapbiomas-raisg/MOSAICOS/mosaics-2';
var image       = ee.Image(inputPath + input);
var mosaics     = ee.ImageCollection(mosaicsPath).merge(ee.ImageCollection(mosaicsPath2));

var regions     = ee.FeatureCollection(regionsPath);
 var region = regions.filter(ee.Filter.inList('id_regionC', regionIds))

var mosaicRegion = param.regionIds.toString().slice(0, 3);
if (mosaicRegion ==='211' || mosaicRegion ==='205' ){mosaicRegion='210'  }

//var filteredMosaics = mosaics
//  .filter(ee.Filter.eq('region_code', parseInt(mosaicRegion)))
  // .filter(
  //   ee.Filter.or(
  //     ee.Filter.eq('country', country)
  //   )
  // )
//  .select(['swir1_median', 'nir_median', 'red_median']);


var filteredRegions = region
  // .filter(
  //   ee.Filter.or(
  //     ee.Filter.inList('id_region', regionIds),
  //     ee.Filter.inList('id_regionc', regionIds)
  //   )
  // );
  print('regions2',filteredRegions,region.limit(2))
  // Use region geometry instead of region_code to avoid tiling gaps
  var filteredMosaics = mosaics
  .filterBounds(filteredRegions.geometry())
  .select(['swir1_median', 'nir_median', 'red_median']);
var regionsRaster = filteredRegions
  .map(function(item) { return item.set('version', 1) })
  .reduceToImage(['version'], ee.Reducer.first());
  



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
        currentImage = currentImage.unmask(previousImage.select([0]));

        return currentImage.addBands(previousImage);
      },
      ee.Image(image.select([bandNames.get(0)]))
    );

  imageFilledt0tn = ee.Image(imageFilledt0tn);


  // apply the gap fill form tn until t0
  var bandNamesReversed = bandNames.reverse();

  var imageFilledtnt0 = bandNamesReversed.slice(1)
    .iterate(
      function (bandName, previousImage) {

        var currentImage = imageFilledt0tn.select(ee.String(bandName));
        previousImage = ee.Image(previousImage);

        currentImage = currentImage
          .unmask(previousImage
            .select(previousImage.bandNames().length().subtract(1))
          );

        return previousImage.addBands(currentImage);

      },
      ee.Image(imageFilledt0tn.select([bandNamesReversed.get(0)]))
    );


  imageFilledtnt0 = ee.Image(imageFilledtnt0).select(bandNames);

  return imageFilledtnt0;
};




/**
 * Implementation
 */
var bandNames = ee.List(
  years.map(
    function (year) { return 'classification_' + year.toString() }
  )
);




// Inserta pixel 0 para mask
var classif = ee.Image();
var bandnameReg = image.bandNames();
bandnameReg.getInfo().forEach(
  function (bandName) {
    var year = parseInt(bandName.split('_')[1], 10);
    
    var nodata = ee.Image(27);
    
    var mosaic =  filteredMosaics
      .filterMetadata('year', 'equals', year);
  
    var mosaicBand = mosaic
      .select('swir1_median')
      .mosaic()
      .updateMask(regionsRaster);
    
    nodata = nodata.updateMask(mosaicBand);
    
    var selected = image.select(bandName);

    var newImage = ee.Image(0)
      .updateMask(regionsRaster)
      .where(nodata.eq(27), 27)
      .where(selected.eq(30).or(selected.eq(1)), 30);
    
    var band0 = newImage.updateMask(newImage.unmask().neq(0));

    classif = classif.addBands(band0.rename(bandName));
    
  }
);
print(image,'image1')
print(classif,'classif')
image = classif.select(bandnameReg);
print(image,'image2')



// generate a histogram dictionary of [bandNames, image.bandNames()]
var bandsOccurrence = ee.Dictionary(
  bandNames
    .cat(image.bandNames())
    .reduce(ee.Reducer.frequencyHistogram())
);




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
 * visualizations
 */
Map.setOptions('SATELLITE');


yearsPreview.forEach(function(year) {
  
  var selector = 'classification_' + year;
  
  var mos = filteredMosaics.filterMetadata('year', 'equals', year).mosaic();
  mos = mos.mask(regionsRaster);
  
  Map.addLayer(
    mos, 
    {
      bands: ['swir1_median', 'nir_median', 'red_median'],
      gain: [0.08, 0.06, 0.2]
    }, 
    'MOSAICO ' + year,
    false
  );
  
  Map.addLayer(
    image,//.eq(30).selfMask().multiply(30),
    {
      'bands': [selector],
      'min': 0,
      'max': 34,
      'palette': palette,
      'format': 'png'
    },
    'ORIGINAL ' + year,
    false
  );
  
  Map.addLayer(
    imageFilledtnt0,//.eq(30).selfMask().multiply(30),
    {
      'bands': [selector],
      'min': 0,
      'max': 34,
      'palette': palette,
      'format': 'png'
    },
    'GAP FILL ' + year
  );
  
});





/**
  * Export images to asset
  */
imageFilledtnt0 = imageFilledtnt0.select(bandNames)
  .set({
    code_region: regionIds,
    pais: country,
    version: version,
    descripcion: 'gapfill',
    cover: 'MINING'
  });
  
print('INPUT: ' + input, image);
print('OUTPUT: ' + output, imageFilledtnt0);

Export.image.toAsset({
  image: imageFilledtnt0,
  description: output,
  assetId:  outputDir + output,
  pyramidingPolicy: { '.default': 'mode' },
  region: filteredRegions.geometry().bounds(),
  scale: 30,
  maxPixels: 1e13
});
```
