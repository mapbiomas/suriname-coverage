```javascript
/**** Mangrove mosaics | Time series | Suriname + French Guiana ****/
/**** Output: one mosaic per year ****/

// ======================================================
// OPTIONAL DRAWN GEOMETRY
// ======================================================
// If you want to use a drawn geometry, draw it in the GEE map and
// replace null below with the geometry variable name, for example:
//
// var drawnGeometry = geometry;
//
// If you do not want to use a drawn geometry, keep it as null.

var drawnGeometry = null;


// ======================================================
// PARAMETERS
// ======================================================

var countryLayer;

var param = {
  startYear: 1985,
  endYear: 2025,

  cloudCover: 90,

  // Countries to be processed together
  countries: ['SURINAME', 'GUIANA_FRANCESA'],

  // Name used in export files and metadata
  outputName: 'mangrove_mosaic',

  // If true, the final mosaic is clipped to the study area geometry
  clipToStudyArea: false,

  // If true and drawnGeometry is not null, the drawn geometry will be used
  // instead of the country boundaries
  useDrawnGeometry: false,

  // Export folder
  outDir: 'projects/mapbiomas-suriname/assets/MANGROVE/MOSAIC/mangrove_mosaics',

  // Years to visualize in the map
  previewYears: [1990, 2000, 2025],

  pathRows: {
    PERU: [
      [11, 63],
      [10, 64],
      [11, 64]
    ],

    COLOMBIA: [
      [10, 53],
      [10, 54],
      [10, 55],
      [10, 56],
      [10, 57],
      [10, 58],
      [10, 59],
      [11, 55],
      [11, 59],
      [14, 51],
      [7, 51],
      [7, 52],
      [8, 51],
      [8, 52],
      [9, 52],
      [9, 53],
      [9, 54]
    ],

    VENEZUELA: [
      [2, 52],
      [3, 52],
      [7, 52],
      [5, 52],
      [6, 52],
      [4, 53],
      [2, 53],
      [7, 53],
      [5, 53],
      [3, 53],
      [1, 53],
      [7, 54],
      [7, 51],
      [8, 51],
      [232, 54],
      [233, 53],
      [233, 54]
    ],

    ECUADOR: [
      [10, 58],
      [10, 59],
      [10, 60],
      [11, 59],
      [11, 60],
      [11, 61],
      [10, 62],
      [11, 62],
      [10, 63],
      [11, 63],
      [17, 60],
      [17, 61],
      [18, 60],
      [18, 61],
      [10, 64],
      [11, 64]
    ],

    GUYANA: [
      [230, 56],
      [231, 55],
      [231, 56],
      [232, 54],
      [232, 55]
    ],

    SURINAME: [
      [230, 56],
      [228, 56],
      [228, 57],
      [229, 56]
    ],

    GUIANA_FRANCESA: [
      [226, 57],
      [227, 56],
      [227, 57],
      [228, 56],
      [228, 57]
    ]
  },

  blacklist: {
    PERU: [],

    COLOMBIA: [
      "LT04_007051_19881212",
      "LT04_007051_19881110",
      "LT04_007051_19881025",
      "LT04_007051_19881009",
      "LT04_007051_19880923",
      "LT04_007052_19890113",
      "LT04_007052_19881110",
      "LT04_007052_19881025",
      "LT04_007052_19880923",
      "LT04_008052_19881117",
      "LT04_010053_19881217",
      "LT04_010053_19881201",
      "LT04_010053_19880928",
      "LT04_010053_19880608",
      "LT04_010053_19880523",
      "LT04_010055_19881201",
      "LT04_010055_19880405",
      "LT04_010055_19880320",
      "LT04_010058_19880320",
      "LT04_010059_19880608",
      "LT04_010059_19880523",
      "LT04_010059_19880405",
      "LT04_011055_19880327",
      "LT04_011055_19880514",
      "LT04_014051_19880519",
      "LT04_014051_19881010",
      "LT04_014051_19881025",
      "LT04_014051_19880924",
      "LT04_009054_19881210",
      "LT04_009054_19881226",
      "LT04_008052_19880626",

      "LT05_007051_19951224",
      "LT05_008052_19950809",
      "LT05_010054_19951213",
      "LT05_010059_19951111",
      "LT05_010059_19951213",
      "LT05_009053_19950816",
      "LT05_009053_19950901",
      "LT05_008052_19950809",
      "LT05_008052_19950910",

      "LE07_007052_20090731",
      "LE07_007052_20090816",
      "LE07_007052_20090901",
      "LE07_007052_20091019",
      "LE07_007052_20091120",
      "LE07_007052_20091206",
      "LE07_007052_20100107",
      "LE07_008052_20100114",
      "LE07_010054_20090704",
      "LE07_010054_20090618",
      "LE07_010054_20090314",
      "LE07_010055_20090415",
      "LE07_010056_20090821",
      "LE07_010057_20091227",
      "LE07_010057_20090602",
      "LE07_010058_20090330",
      "LE07_010058_20090415",
      "LE07_010058_20090704",
      "LE07_010058_20090821",
      "LE07_010058_20091008",
      "LE07_010058_20100213",
      "LE07_010058_20091227",
      "LE07_010058_20091008",
      "LE07_010058_20090922",
      "LE07_010058_20090821",
      "LE07_010058_20090805",
      "LE07_010058_20090704",
      "LE07_010058_20091125",
      "LE07_010059_20090517",
      "LE07_010059_20090415",
      "LE07_010059_20090922",
      "LE07_011055_20090625",
      "LE07_014051_20090918",
      "LE07_009052_20090510",
      "LT05_009052_20091126",
      "LT05_009052_20100129",
      "LE07_009052_20091204",
      "LE07_008051_20100114",
      "LE07_008051_20091111",
      "LE07_008052_20091010",
      "LE07_009052_20091118",
      "LE07_009052_20091102",
      "LE07_009052_20091001",
      "LE07_009052_20090814",
      "LE07_009052_20090627",

      "LC08_010053_20230329",
      "LC08_010053_20230719",
      "LC08_010053_20230804",
      "LC08_010053_20230905",
      "LC08_010053_20230921",
      "LC08_010053_20231007",
      "LC08_010053_20231023",
      "LC08_010053_20231108",
      "LC08_010053_20231124",
      "LC09_010053_20230422",
      "LC09_010053_20230727",
      "LC09_010053_20230812",
      "LC09_010053_20230828",
      "LC09_010053_20230929",
      "LC09_010053_20231015",
      "LC09_010053_20231031",
      "LC08_010054_20230329",
      "LC08_010054_20230414",
      "LC08_010054_20230703",
      "LC08_010054_20230719",
      "LC08_010054_20230804",
      "LC08_010054_20230905",
      "LC08_010054_20231023",
      "LC08_010054_20231108",
      "LC08_010054_20231124",
      "LC09_010054_20230217",
      "LC09_010054_20230321",
      "LC09_010054_20230406",
      "LC09_010054_20230422",
      "LC09_010054_20230508",
      "LC09_010054_20230609",
      "LC09_010054_20230828",
      "LC09_010054_20230929",
      "LC09_010054_20231015",
      "LC09_010054_20231031",
      "LC09_010054_20231116",
      "LC08_010055_20230329",
      "LC08_010055_20230703",
      "LC08_010055_20230921",
      "LC09_010055_20230321",
      "LC09_010055_20230422",
      "LC09_010055_20230508",
      "LC09_010055_20230609",
      "LC09_010055_20230711",
      "LC09_010055_20230913",
      "LC09_010055_20230929",
      "LC09_010055_20231015",
      "LC09_010055_20231031",
      "LC08_010055_20230703",
      "LC08_010055_20230719",
      "LC08_010055_20230921",
      "LC09_010055_20230422",
      "LC09_010055_20230508",
      "LC09_010055_20230711",
      "LC09_010055_20230828",
      "LC09_010055_20231015",
      "LC09_010055_20231031",
      "LC08_010056_20230703",
      "LC09_010056_20230321",
      "LC08_010057_20230703",
      "LC08_010057_20230804",
      "LC09_010057_20230422",
      "LC08_010058_20230804",
      "LC08_010058_20231108",
      "LC09_010058_20230812",
      "LC09_010058_20231116",
      "LC08_010058_20230804",
      "LC08_010058_20230820",
      "LC08_010058_20231108",
      "LC09_010058_20230217",
      "LC08_010059_20230804",
      "LC08_010059_20230820",
      "LC08_010059_20231108",
      "LC09_010059_20230508",
      "LC09_010059_20230711",
      "LC09_010059_20231116",
      "LC08_010059_20230430",
      "LC08_010059_20231108",
      "LC09_010059_20230321",
      "LC09_010059_20230508",
      "LC09_010059_20230609",
      "LC09_010059_20230711",
      "LC09_010059_20230929",
      "LC08_011055_20230115",
      "LC08_011055_20230216",
      "LC08_011055_20230304",
      "LC09_011055_20230224",
      "LC09_011055_20230413",
      "LC09_011055_20230702",
      "LC09_011055_20230803",
      "LC09_011059_20230312",
      "LC08_014051_20230205",
      "LC08_014051_20230221",
      "LC08_014051_20230309",
      "LC08_014051_20230410",
      "LC08_014051_20230426",
      "LC08_014051_20230512",
      "LC08_014051_20230528",
      "LC08_014051_20230613",
      "LC08_014051_20230629",
      "LC08_014051_20230731",
      "LC08_014051_20230816",
      "LC08_014051_20230917",
      "LC08_014051_20231003",
      "LC08_014051_20231019",
      "LC08_014051_20231104",
      "LC08_014051_20231120",
      "LC09_014051_20230213",
      "LC09_014051_20230317",
      "LC09_014051_20230402",
      "LC09_014051_20230418",
      "LC09_014051_20230504",
      "LC09_014051_20230520",
      "LC09_014051_20230605",
      "LC09_014051_20230621",
      "LC09_014051_20230909",
      "LC09_014051_20230925",
      "LC09_014051_20231011",
      "LC09_014051_20231128",
      "LC09_007051_20230316",
      "LC09_007051_20230401",
      "LC09_007051_20230722",
      "LC09_007052_20231127",
      "LC09_007052_20231111",
      "LC09_007052_20231010",
      "LC09_008052_20231102",
      "LC09_008052_20230323",
      "LC09_008052_20230408",
      "LC09_008052_20230611",
      "LC09_008052_20230203"
    ],

    VENEZUELA: [],

    ECUADOR: [
      'LE07_010059_20180118',
      'LE07_010060_20181204',
      'LE07_010060_20181118',
      'LC08_010059_20181228'
    ],

    GUYANA: [],
    SURINAME: [],
    GUIANA_FRANCESA: []
  }
};


// ======================================================
// STUDY AREA
// ======================================================

var getStudyArea = function(param) {
  var countriesPath = 'projects/mapbiomas-raisg/DATOS_AUXILIARES/VECTORES/paises-5';

  var countryNames = {
    VENEZUELA: 'Venezuela',
    COLOMBIA: 'Colombia',
    ECUADOR: 'Ecuador',
    PERU: 'Perú',
    GUYANA: 'Guyana',
    SURINAME: 'Suriname',
    GUIANA_FRANCESA: 'Guyane Française'
  };

  var names = param.countries.map(function(country) {
    return countryNames[country];
  });

  var countriesFc = ee.FeatureCollection(countriesPath)
    .filter(ee.Filter.inList('name', names));

  var studyArea = countriesFc;

  if (param.useDrawnGeometry === true && drawnGeometry !== null) {
    studyArea = ee.FeatureCollection([
      ee.Feature(drawnGeometry, {
        name: param.outputName
      })
    ]);
  }

  return studyArea;
};


// ======================================================
// AUXILIARY FUNCTIONS
// ======================================================

var getYears = function(startYear, endYear) {
  var years = [];

  for (var year = startYear; year <= endYear; year++) {
    years.push(year);
  }

  return years;
};


var getMergedPathRows = function(param) {
  var merged = [];

  param.countries.forEach(function(country) {
    var list = param.pathRows[country] || [];

    list.forEach(function(pathRow) {
      var key = pathRow[0] + '_' + pathRow[1];

      var alreadyExists = merged.some(function(item) {
        return item[0] + '_' + item[1] === key;
      });

      if (!alreadyExists) {
        merged.push(pathRow);
      }
    });
  });

  return merged;
};


var getMergedBlacklist = function(param) {
  var merged = [];

  param.countries.forEach(function(country) {
    var list = param.blacklist[country] || [];

    list.forEach(function(id) {
      if (merged.indexOf(id) === -1) {
        merged.push(id);
      }
    });
  });

  return merged;
};


// ======================================================
// LANDSAT COLLECTION
// ======================================================

var getCollection = function(param) {
  var cloudCover = param.cloudCover;
  var collectionId = param.collectionId;
  var start = param.startDate;
  var finish = param.endDate;
  var bands = param.bands;
  var pathRows = param.mergedPathRows;
  var blacklist = param.mergedBlacklist || [];
  var satellite = param.satellite;

  var bandNamesTOA = ee.List([
    'blue',
    'green',
    'red',
    'nir',
    'swir1',
    'swir2',
    'BQA'
  ]);

  var main = ee.ImageCollection(collectionId)
    .filter(
      ee.Filter.and(
        ee.Filter.lte('CLOUD_COVER', cloudCover),
        ee.Filter.date(start, finish)
      )
    )
    .select(bands.get(satellite), bandNamesTOA);

  var merged = ee.ImageCollection([]);

  pathRows.forEach(function(pathRow) {
    var filtered = main.filter(
      ee.Filter.and(
        ee.Filter.eq('WRS_PATH', pathRow[0]),
        ee.Filter.eq('WRS_ROW', pathRow[1]),
        ee.Filter.inList('system:index', blacklist).not()
      )
    );

    merged = merged.merge(filtered);
  });

  return merged;
};


var applyGetCollection = function(param) {
  var year = param.year;
  var toa = {};

  var collectionIds = {
    L4: 'LANDSAT/LT04/C02/T1_TOA',
    L5: 'LANDSAT/LT05/C02/T1_TOA',
    L7: 'LANDSAT/LE07/C02/T1_TOA',
    L8: 'LANDSAT/LC08/C02/T1_TOA',
    L9: 'LANDSAT/LC09/C02/T1_TOA'
  };

  param.bands = ee.Dictionary({
    L9: ee.List([1, 2, 3, 4, 5, 6, 'QA_PIXEL']),
    L8: ee.List([1, 2, 3, 4, 5, 6, 'QA_PIXEL']),
    L7: ee.List([0, 1, 2, 3, 4, 7, 'QA_PIXEL']),
    L5: ee.List([0, 1, 2, 3, 4, 6, 'QA_PIXEL']),
    L4: ee.List([0, 1, 2, 3, 4, 6, 'QA_PIXEL'])
  });

  Object.keys(collectionIds).forEach(function(satellite) {
    param.collectionId = collectionIds[satellite];
    param.satellite = satellite;
    toa[satellite] = getCollection(param);
  });

  var ls;

  if (year < 2014) {
    ls = ee.ImageCollection(toa.L4)
      .merge(toa.L5)
      .merge(toa.L7)
      .merge(toa.L8);
  } else {
    ls = ee.ImageCollection(toa.L8)
      .merge(toa.L9);
  }

  return ls;
};


// ======================================================
// CLOUD AND SHADOW MASK
// ======================================================

var bqaFunction = function(image) {
  var dilatedCloud = 1 << 1;
  var cloud = 1 << 3;
  var shadow = 1 << 4;

  var qa = image.select('BQA');

  var mask = qa.bitwiseAnd(dilatedCloud)
    .or(qa.bitwiseAnd(cloud))
    .or(qa.bitwiseAnd(shadow));

  return image.updateMask(mask.not());
};


// ======================================================
// SPECTRAL INDICES
// ======================================================

var createIndexs = function(image) {
  var NDVI = image.expression(
    '((nir - red) / (nir + red))',
    {
      nir: image.select('nir'),
      red: image.select('red')
    }
  );

  var NDSI = image.expression(
    '((swir1 - nir) / (nir + swir1))',
    {
      swir1: image.select('swir1'),
      nir: image.select('nir')
    }
  );

  var NDWI = image.expression(
    '((green - nir) / (nir + green))',
    {
      green: image.select('green'),
      nir: image.select('nir')
    }
  );

  var MNDWI = image.expression(
    '((green - swir1) / (green + swir1))',
    {
      green: image.select('green'),
      swir1: image.select('swir1')
    }
  );

  var MMRI = image.expression(
    '(abs(MNDWI) - abs(NDVI)) / (abs(MNDWI) + abs(NDVI))',
    {
      MNDWI: MNDWI,
      NDVI: NDVI
    }
  );

  var totalPhosphorus = image.expression(
    '2.71828 ** (-0.4081 - 8.659 * (1 / (red / green)))',
    {
      green: image.select('green'),
      red: image.select('red')
    }
  );

  var totalNitrate = image.expression(
    '2.71828 ** (8.228 - 2.713 * (1 / (red + green)))',
    {
      green: image.select('green'),
      red: image.select('red')
    }
  );

  return image
    .addBands(NDVI.rename('NDVI'))
    .addBands(MNDWI.rename('MNDWI'))
    .addBands(NDSI.rename('NDSI'))
    .addBands(NDWI.rename('NDWI'))
    .addBands(MMRI.rename('MMRI'))
    .addBands(totalNitrate.rename('IM1'))
    .addBands(totalPhosphorus.rename('IM2'));
};


// ======================================================
// MOSAIC CREATION
// ======================================================

var createMosaic = function(year, param, studyArea) {
  var localParam = JSON.parse(JSON.stringify(param));

  localParam.year = year;
  localParam.startDate = year + '-01-01';
  localParam.endDate = year + '-12-31';

  localParam.mergedPathRows = getMergedPathRows(param);
  localParam.mergedBlacklist = getMergedBlacklist(param);

  var mosaicMerge = applyGetCollection(localParam)
    .map(createIndexs)
    .map(bqaFunction);

  var mosaic = mosaicMerge.median();

  var selected = mosaic.select([
    'NDVI',
    'MNDWI',
    'MMRI',
    'NDSI',
    'NDWI'
  ]);

  var im1 = mosaic
    .select('IM1')
    .unitScale(5.04893968546401e-7, 0.004990332819056622)
    .multiply(255)
    .clamp(0, 255)
    .rename('IM1');

  var im2 = mosaic
    .select('IM2')
    .unitScale(8.53118346494392e-9, 27.249109247040032)
    .multiply(255)
    .clamp(0, 255)
    .rename('IM2');

  var mosaicNew = mosaic
    .select([
      'swir2',
      'swir1',
      'nir',
      'red',
      'green',
      'BQA'
    ])
    .multiply(255)
    .addBands(
      selected
        .add(1)
        .multiply(127)
    )
    .addBands(im1)
    .addBands(im2)
    .byte();

  if (param.clipToStudyArea === true) {
    mosaicNew = mosaicNew.clip(studyArea.geometry());
  }

  return {
    image: mosaicNew,
    collection: mosaicMerge
  };
};


// ======================================================
// IMPLEMENTATION
// ======================================================

countryLayer = getStudyArea(param);

var years = getYears(param.startYear, param.endYear);

Map.addLayer(
  countryLayer,
  {},
  'STUDY AREA',
  false
);

Map.centerObject(countryLayer, 7);

print('Processing years:', years);
print('Countries:', param.countries);
print('Merged path/rows:', getMergedPathRows(param));
print('Merged blacklist:', getMergedBlacklist(param));
print('Study area:', countryLayer);


// ======================================================
// EXPORT LOOP
// ======================================================

years.forEach(function(year) {
  var result = createMosaic(year, param, countryLayer);

  var mosaic = result.image;
  var mosaicMerge = result.collection;

  var fileName = param.outputName + '-' + year;

  // Preview only selected years to avoid too many map layers
  if (param.previewYears.indexOf(year) !== -1) {
    Map.addLayer(
      mosaic.select('MMRI'),
      {
        min: 10,
        max: 220,
        palette: 'b3260f,ff9c08,2dff34,1f64ff'
      },
      'Mosaic MMRI ' + year,
      false
    );

    Map.addLayer(
      mosaic,
      {
        bands: ['swir1', 'nir', 'red'],
        min: 0,
        max: 255,
        gamma: 1.8
      },
      'Mosaic SWIR1/NIR/RED ' + year,
      false
    );

    print('Image count ' + year, mosaicMerge.size());

    print(
      'Images used in ' + year,
      mosaicMerge.aggregate_array('system:index')
    );
  }

  Export.image.toAsset({
    image: mosaic
      .toByte()
      .set({
        year: year,
        version: 6,
        country: param.outputName,
        countries: param.countries.join(','),
        start_date: year + '-01-01',
        end_date: year + '-12-31'
      }),

    description: fileName,

    assetId: param.outDir + '/' + fileName,

    region: countryLayer.geometry().bounds(),

    scale: 30,

    maxPixels: 1e13
  });
});
```
