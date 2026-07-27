```javascript
/**************************************
 * Temporal Filter (region-only)
 * - Server-side version to avoid browser freezing
 * - Preserves the original temporal filter logic
 **************************************/

var param = { 
  code_region: 60201,        // classification region
  pais: 'GUYANE_FRANCAISE',
  tema: 'MINING',
  year: 2022,                // visualization only
  version_input: 2515,       // input version of classification (temporal filter IN)
  version_output: 2516,      // output version (temporal filter OUT)
  exportOpcion: {
    DriveFolder: 'DRIVE-EXPORT',
    exportClasifToDrive: false,
    exportEstadistica:  false
  },
  exclusion: {               // classes/years to exclude from temporal filter
    clases: [],
    years:  []
  }
};

// ---------------- Priority order (middle/first/last) ----------------
var ordem_exec_first  = [27];
var ordem_exec_last   = [30];
var ordem_exec_middle = [30, 27, 30, 27];

// ---------------- Paths ----------------
var dirinput = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/classification-ft';
var dirout   = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/classification-ft/';
var AssetMosaic = [
  'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2',
  'projects/mapbiomas-raisg/MOSAICOS/mosaics-2'
];
var AssetRegions = 'projects/mapbiomas-suriname/assets/French-Guiana_Accumulated_Coastline__Regions_2025';

// ---------------- Region vector & mask ----------------
var region = ee.FeatureCollection(AssetRegions)
  .filterMetadata('id_regionC', 'equals', param.code_region);

Map.addLayer(region, {}, 'region', false);

var regionRaster = region
  .map(function (f) { return f.set('version', 1); })
  .reduceToImage(['version'], ee.Reducer.first())
  .selfMask();

// Prefix for exports
var prefixo_out = param.pais + '-' + param.code_region + '-';

// ---------------- Palettes & vis ----------------
var palettes = require('users/mapbiomas/modules:Palettes.js');
var vis = {
  min: 0,
  max: 34,
  palette: palettes.get('classification2')
};

// ---------------- Mosaics by region geometry (no region_code) ----------------
var mosaic = ee.ImageCollection(AssetMosaic[0])
  .merge(ee.ImageCollection(AssetMosaic[1]))
  .filterBounds(region.geometry())
  .filterMetadata('year', 'equals', param.year)
  .select(['swir1_median', 'nir_median', 'red_median']);

Map.addLayer(
  mosaic,
  { bands: ['swir1_median', 'nir_median', 'red_median'], min: 200, max: 5000 },
  'MOSAIC ' + param.year,
  false
);

// ---------------- Load input classification ----------------
var Clasificacion_TD = ee.Image(
  dirinput + '/' + param.pais + '-' + param.code_region + '-' + param.version_input
);
print('Input (Clasificacion_TD)', Clasificacion_TD);

// ---------------- Years & band names ----------------
var years = [
  1985, 1986, 1987, 1988, 1989, 1990, 1991, 1992,
  1993, 1994, 1995, 1996, 1997, 1998, 1999, 2000,
  2001, 2002, 2003, 2004, 2005, 2006, 2007, 2008,
  2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016,
  2017, 2018, 2019, 2020, 2021, 2022, 2023, 2024,
  2025
];

var yearsList = ee.List(years);

var bandNames = ee.List(
  years.map(function (y) { return 'classification_' + String(y); })
);

var bandNamesExclude = ee.List(
  param.exclusion.years.map(function (y) { return 'classification_' + String(y); })
);

// ---------------- Helpers ----------------
var bandName = function(year) {
  year = ee.Number(year).toInt();
  return ee.String('classification_').cat(year.format());
};

var getBand = function(image, year) {
  return ee.Image(image).select([bandName(year)]);
};

var buildImageFromYearList = function(imageList) {
  return ee.ImageCollection.fromImages(imageList).toBands()
    .rename(bandNames);
};

// ---------------- Ensure all target bands exist ----------------
var bandsOccurrence = ee.Dictionary(
  bandNames.cat(Clasificacion_TD.bandNames()).reduce(ee.Reducer.frequencyHistogram())
);

var bandsDictionary = bandsOccurrence.map(function (key, value) {
  return ee.Image(
    ee.Algorithms.If(
      ee.Number(value).eq(2),
      Clasificacion_TD.select([key]).byte(),
      ee.Image().rename([key]).byte().updateMask(Clasificacion_TD.select(0))
    )
  );
});

var imageAllBands = ee.Image(
  bandNames.iterate(function (b, acc) {
    return ee.Image(acc).addBands(bandsDictionary.get(ee.String(b)));
  }, ee.Image().select())
);

// ---------------- Replace masked with 0 inside region ----------------
var classif0List = yearsList.map(function(year) {
  year = ee.Number(year).toInt();
  var bname = bandName(year);
  var im = imageAllBands.select([bname]);
  var filled = ee.Image(0).updateMask(regionRaster);
  filled = filled.where(im.mask(), im);
  return filled.rename(bname);
});

var classif0 = buildImageFromYearList(classif0List);
Clasificacion_TD = classif0.select(bandNames).selfMask();

// ---------------- Temporal rules (3/4/5 year windows) ----------------
var mask3_to_year = function(valor, ano, img) {
  valor = ee.Number(valor);
  ano = ee.Number(ano).toInt();
  img = ee.Image(img);

  var current = getBand(img, ano);

  var m = getBand(img, ano.subtract(1)).eq(valor)
    .and(getBand(img, ano).neq(valor))
    .and(getBand(img, ano.add(1)).eq(valor));

  var fix = current.mask(m.eq(1)).where(m.eq(1), valor);
  return current.blend(fix).rename(bandName(ano));
};

var mask4_to_year = function(valor, ano, img) {
  valor = ee.Number(valor);
  ano = ee.Number(ano).toInt();
  img = ee.Image(img);

  var current = getBand(img, ano);
  var next1 = getBand(img, ano.add(1));

  var m = getBand(img, ano.subtract(1)).eq(valor)
    .and(getBand(img, ano).neq(valor))
    .and(getBand(img, ano.add(1)).neq(valor))
    .and(getBand(img, ano.add(2)).eq(valor));

  var fixY  = current.mask(m.eq(1)).where(m.eq(1), valor);
  var fixY1 = next1.mask(m.eq(1)).where(m.eq(1), valor);

  return current.blend(fixY).blend(fixY1).rename(bandName(ano));
};

var mask5_to_year = function(valor, ano, img) {
  valor = ee.Number(valor);
  ano = ee.Number(ano).toInt();
  img = ee.Image(img);

  var current = getBand(img, ano);
  var next1 = getBand(img, ano.add(1));
  var next2 = getBand(img, ano.add(2));

  var m = getBand(img, ano.subtract(1)).eq(valor)
    .and(getBand(img, ano).neq(valor))
    .and(getBand(img, ano.add(1)).neq(valor))
    .and(getBand(img, ano.add(2)).neq(valor))
    .and(getBand(img, ano.add(3)).eq(valor));

  var fixY  = current.mask(m.eq(1)).where(m.eq(1), valor);
  var fixY1 = next1.mask(m.eq(1)).where(m.eq(1), valor);
  var fixY2 = next2.mask(m.eq(1)).where(m.eq(1), valor);

  return current.blend(fixY).blend(fixY1).blend(fixY2).rename(bandName(ano));
};

var window3years = function(img, valor) {
  img = ee.Image(img);
  valor = ee.Number(valor);

  var imageList = yearsList.map(function(y) {
    y = ee.Number(y).toInt();
    var original = getBand(img, y).rename(bandName(y));

    return ee.Image(ee.Algorithms.If(
      y.eq(1985),
      original,
      ee.Algorithms.If(
        y.gte(1986).and(y.lte(2024)),
        mask3_to_year(valor, y, img),
        original
      )
    ));
  });

  return buildImageFromYearList(imageList);
};

var window4years = function(img, valor) {
  img = ee.Image(img);
  valor = ee.Number(valor);

  var imageList = yearsList.map(function(y) {
    y = ee.Number(y).toInt();
    var original = getBand(img, y).rename(bandName(y));

    return ee.Image(ee.Algorithms.If(
      y.eq(1985),
      original,
      ee.Algorithms.If(
        y.gte(1986).and(y.lte(2023)),
        mask4_to_year(valor, y, img),
        original
      )
    ));
  });

  return buildImageFromYearList(imageList);
};

var window5years = function(img, valor) {
  img = ee.Image(img);
  valor = ee.Number(valor);

  var imageList = yearsList.map(function(y) {
    y = ee.Number(y).toInt();
    var original = getBand(img, y).rename(bandName(y));

    return ee.Image(ee.Algorithms.If(
      y.eq(1985),
      original,
      ee.Algorithms.If(
        y.gte(1986).and(y.lte(2022)),
        mask5_to_year(valor, y, img),
        original
      )
    ));
  });

  return buildImageFromYearList(imageList);
};

// ---------------- First/last year rules ----------------
var mask3first = function(valor, img) {
  valor = ee.Number(valor);
  img = ee.Image(img);

  var m = getBand(img, 1985).neq(valor)
    .and(getBand(img, 1986).eq(valor))
    .and(getBand(img, 1987).eq(valor));

  var fix = getBand(img, 1985).mask(m.eq(1)).where(m.eq(1), valor);
  var firstBand = getBand(img, 1985).blend(fix).rename('classification_1985');

  var imageList = yearsList.map(function(y) {
    y = ee.Number(y).toInt();

    return ee.Image(ee.Algorithms.If(
      y.eq(1985),
      firstBand,
      getBand(img, y).rename(bandName(y))
    ));
  });

  return buildImageFromYearList(imageList);
};

var mask3last = function(valor, img) {
  valor = ee.Number(valor);
  img = ee.Image(img);

  var m = getBand(img, 2023).eq(valor)
    .and(getBand(img, 2024).eq(valor))
    .and(getBand(img, 2025).neq(valor));

  var fix = getBand(img, 2025).mask(m.eq(1)).where(m.eq(1), valor);
  var lastBand = getBand(img, 2025).blend(fix).rename('classification_2025');

  var imageList = yearsList.map(function(y) {
    y = ee.Number(y).toInt();

    return ee.Image(ee.Algorithms.If(
      y.eq(2025),
      lastBand,
      getBand(img, y).rename(bandName(y))
    ));
  });

  return buildImageFromYearList(imageList);
};

// ---------------- Apply temporal filtering ----------------
var filtered = Clasificacion_TD;

// First-year priority classes
filtered = ee.Image(ee.List(ordem_exec_first).iterate(function(id_class, img) {
  return mask3first(ee.Number(id_class), ee.Image(img));
}, filtered));

// Last-year priority classes
filtered = ee.Image(ee.List(ordem_exec_last).iterate(function(id_class, img) {
  return mask3last(ee.Number(id_class), ee.Image(img));
}, filtered));

// Middle years (3, then 4+5, then 3 again)
filtered = ee.Image(ee.List(ordem_exec_middle).iterate(function(id_class, img) {
  return window3years(ee.Image(img), ee.Number(id_class));
}, filtered));

filtered = ee.Image(ee.List(ordem_exec_middle).iterate(function(id_class, img) {
  img = ee.Image(img);
  id_class = ee.Number(id_class);
  img = window4years(img, id_class);
  img = window5years(img, id_class);
  return img;
}, filtered));

filtered = ee.Image(ee.List(ordem_exec_middle).iterate(function(id_class, img) {
  return window3years(ee.Image(img), ee.Number(id_class));
}, filtered));

// Re-select target bands
var classif_FT = filtered.select(bandNames);

// ---------------- Exclusions: classes ----------------
if (param.exclusion.clases.length > 0) {
  var keep = ee.List([]);
  param.exclusion.clases.forEach(function (clase) {
    var maskThis = Clasificacion_TD.eq(clase).selfMask();
    keep = keep.add(Clasificacion_TD.updateMask(maskThis).selfMask());
  });
  keep = ee.ImageCollection(keep).max();
  Map.addLayer(keep.updateMask(regionRaster), {}, 'excluded_classes', false);
  classif_FT = classif_FT.blend(keep);
  print('Classes excluded from temporal filter', param.exclusion.clases);
}

// ---------------- Exclusions: years ----------------
if (param.exclusion.years.length > 0) {
  var yearEx = Clasificacion_TD.select(bandNamesExclude);
  classif_FT = classif_FT.addBands(yearEx, null, true);
  print('Years excluded from temporal filter', param.exclusion.years);
}

// Limit to region
filtered = classif_FT.select(bandNames).updateMask(regionRaster);

// ---------------- Visualization ----------------
Map.addLayer(
  Clasificacion_TD.updateMask(regionRaster),
  { bands: 'classification_' + param.year, min: 0, max: 34, palette: palettes.get('classification2') },
  'original ' + param.year,
  false
);

Map.addLayer(
  filtered.updateMask(regionRaster),
  { bands: 'classification_' + param.year, min: 0, max: 34, palette: palettes.get('classification2') },
  'filtered ' + param.year,
  true
);

// ---------------- Metadata & Export ----------------
filtered = filtered
  .set('code_region', param.code_region)
  .set('country', param.pais)
  .set('version', param.version_output)
  .set('description', 'temporal filter')
  .set('step', 'S04-2');

print('Filtered (temporal)', filtered);

// Export to GEE
Export.image.toAsset({
  image: filtered,
  description: prefixo_out + param.version_output,
  assetId: dirout + prefixo_out + param.version_output,
  pyramidingPolicy: { '.default': 'mode' },
  region: region.geometry(),
  scale: 30,
  maxPixels: 1e13
});

// Optional Drive export
if (param.exportOpcion.exportClasifToDrive) {
  Export.image.toDrive({
    image: filtered.toInt8(),
    description: prefixo_out + 'DRIVE-' + param.version_output,
    folder: param.exportOpcion.DriveFolder,
    scale: 30,
    maxPixels: 1e13,
    region: region.geometry().bounds()
  });
}

/**
 * Coverage statistics by year & class (optional)
 */
function getAreas(image, regionFC) {
  var pixelArea = ee.Image.pixelArea();
  var reducer = {
    reducer: ee.Reducer.sum(),
    geometry: regionFC.geometry(),
    scale: 30,
    maxPixels: 1e13
  };
  var bns = image.bandNames();
  var classIds = ee.List.sequence(0, 34);

  bns.evaluate(function (bands, error) {
    if (error) {
      print(error.message);
      return;
    }

    var yearsAreas = [];

    bands.forEach(function (band) {
      var year = ee.String(band).split('_').get(1),
          yearImage = image.select([band]);

      var covers = classIds.map(function (cid) {
        cid = ee.Number(cid).int8();
        var cover = yearImage.eq(cid).multiply(pixelArea).divide(1e6);
        return cover.reduceRegion(reducer).get(band);
      }).add(year);

      var keys = classIds.map(function (it) {
        it = ee.Number(it).int8();
        var s = ee.String(it);
        s = ee.Algorithms.If(it.lt(10), ee.String('ID0').cat(s), ee.String('ID').cat(s));
        return ee.String(s);
      }).add('year');

      var dict = ee.Dictionary.fromLists(keys, covers);
      yearsAreas.push(ee.Feature(null, dict));
    });

    yearsAreas = ee.FeatureCollection(yearsAreas);
    Export.table.toDrive({
      collection: yearsAreas,
      description: 'ESTADISTICAS-DE-COBERTURA-' + prefixo_out + param.version_output,
      fileFormat: 'CSV',
      folder: 'P02_2-FiltroTempor-CLASSIFICATION'
    });
  });
}

// Run stats if requested
if (param.exportOpcion.exportEstadistica) {
  getAreas(filtered, region);
}
```
