```javascript
// -- -- -- -- 06_temporal
// post-processing filter: temporal filter is used to identify transitions between classes that are implausible
// Server-side version preserving the original rule order
// Option to export with or without the last-year filter, keeping the same export name

// Import mapbiomas color schema
var vis = {
  min: 0,
  max: 62,
  palette: require('users/mapbiomas/modules:Palettes.js').get('classification8')
};

// Set root directory
var root = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/classification-ft/';
var out = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/classification-ft/';

// Set metadata
var inputVersion = '403';
var outputVersion = '404';

// Define input file
var inputFile = 'SURINAME-80201-402_incidence_v' + inputVersion;

// Preview controls
var SHOW_LAYERS = true;
var previewYear = 2025;

// Export control
// false -> export without applying run_3yr_last()
// true  -> export with applying run_3yr_last()
var APPLY_LAST_YEAR_FILTER = true;

// Load input classification
var classificationInput = ee.Image(root + inputFile);
print('Input classification', classificationInput);

// --------------------------------------------
// Helpers
// --------------------------------------------
var years = ee.List.sequence(1985, 2025);

var bandName = function(year) {
  year = ee.Number(year).toInt();
  return ee.String('classification_').cat(year.format());
};

var getBand = function(image, year) {
  return ee.Image(image).select([bandName(year)]);
};

var buildImageFromYearList = function(imageList) {
  return ee.ImageCollection.fromImages(imageList).toBands()
    .rename(years.map(function(y) { return bandName(y); }));
};

// --------------------------------------------
// Build initial classification stack
// --------------------------------------------
var classificationList = years.map(function(y) {
  y = ee.Number(y).toInt();

  return classificationInput
    .select([bandName(y)])
    .remap([3, 11, 12, 21, 25, 33],
           [3, 11, 12, 21, 25, 33])
    .rename(bandName(y));
});

var classification = buildImageFromYearList(classificationList);
print('Classification stack ready', classification);

// --------------------------------------------
// General temporal rules
// --------------------------------------------
var apply_3yr_rule_to_year = function(image, year, class_id) {
  year = ee.Number(year).toInt();
  class_id = ee.Number(class_id);

  var current = getBand(image, year);

  var valid = year.gte(1986).and(year.lte(2024));

  var result = ee.Image(ee.Algorithms.If(valid,
    current.where(
      getBand(image, year.subtract(1)).eq(class_id)
        .and(getBand(image, year).neq(class_id))
        .and(getBand(image, year.add(1)).eq(class_id)),
      class_id
    ),
    current
  ));

  return result.rename(bandName(year));
};

var apply_4yr_rule_to_year = function(image, year, class_id) {
  year = ee.Number(year).toInt();
  class_id = ee.Number(class_id);

  var current = getBand(image, year);

  var valid = year.gte(1986).and(year.lte(2023));

  var result = ee.Image(ee.Algorithms.If(valid,
    current.where(
      getBand(image, year.subtract(1)).eq(class_id)
        .and(getBand(image, year).neq(class_id))
        .and(getBand(image, year.add(1)).neq(class_id))
        .and(getBand(image, year.add(2)).eq(class_id)),
      class_id
    ),
    current
  ));

  return result.rename(bandName(year));
};

var apply_5yr_rule_to_year = function(image, year, class_id) {
  year = ee.Number(year).toInt();
  class_id = ee.Number(class_id);

  var current = getBand(image, year);

  var valid = year.gte(1986).and(year.lte(2022));

  var result = ee.Image(ee.Algorithms.If(valid,
    current.where(
      getBand(image, year.subtract(1)).eq(class_id)
        .and(getBand(image, year).neq(class_id))
        .and(getBand(image, year.add(1)).neq(class_id))
        .and(getBand(image, year.add(2)).neq(class_id))
        .and(getBand(image, year.add(3)).eq(class_id)),
      class_id
    ),
    current
  ));

  return result.rename(bandName(year));
};

var apply_general_rule_series = function(image, window, class_id) {
  image = ee.Image(image);
  window = ee.Number(window).toInt();
  class_id = ee.Number(class_id);

  var imageList = years.map(function(y) {
    y = ee.Number(y).toInt();

    return ee.Image(ee.Algorithms.If(
      window.eq(3),
      apply_3yr_rule_to_year(image, y, class_id),
      ee.Algorithms.If(
        window.eq(4),
        apply_4yr_rule_to_year(image, y, class_id),
        apply_5yr_rule_to_year(image, y, class_id)
      )
    ));
  });

  return buildImageFromYearList(imageList);
};

// --------------------------------------------
// Deforestation rules
// --------------------------------------------
var apply_3yr_deforestation_rule_to_year = function(image, year, class_ids) {
  image = ee.Image(image);
  year = ee.Number(year).toInt();
  class_ids = ee.List(class_ids);

  var prevClass = ee.Number(class_ids.get(0));
  var currClass = ee.Number(class_ids.get(1));
  var nextClass = ee.Number(class_ids.get(2));
  var outClass  = ee.Number(class_ids.get(3));

  var current = getBand(image, year);
  var valid = year.gte(1986).and(year.lte(2024));

  var result = ee.Image(ee.Algorithms.If(valid,
    current.where(
      getBand(image, year.subtract(1)).eq(prevClass)
        .and(getBand(image, year).eq(currClass))
        .and(getBand(image, year.add(1)).eq(nextClass)),
      outClass
    ),
    current
  ));

  return result.rename(bandName(year));
};

var apply_4yr_deforestation_rule_to_year = function(image, year, class_ids) {
  image = ee.Image(image);
  year = ee.Number(year).toInt();
  class_ids = ee.List(class_ids);

  var c0 = ee.Number(class_ids.get(0));
  var c1 = ee.Number(class_ids.get(1));
  var c2 = ee.Number(class_ids.get(2));
  var c3 = ee.Number(class_ids.get(3));
  var c4 = ee.Number(class_ids.get(4));

  var current = getBand(image, year);
  var valid = year.gte(1986).and(year.lte(2023));

  var result = ee.Image(ee.Algorithms.If(valid,
    current.where(
      getBand(image, year.subtract(1)).eq(c0)
        .and(getBand(image, year).eq(c1))
        .and(getBand(image, year.add(1)).eq(c2))
        .and(getBand(image, year.add(2)).eq(c3)),
      c4
    ),
    current
  ));

  return result.rename(bandName(year));
};

var apply_deforestation_rule_series = function(image, window, class_ids) {
  image = ee.Image(image);
  window = ee.Number(window).toInt();
  class_ids = ee.List(class_ids);

  var imageList = years.map(function(y) {
    y = ee.Number(y).toInt();

    return ee.Image(ee.Algorithms.If(
      window.eq(3),
      apply_3yr_deforestation_rule_to_year(image, y, class_ids),
      apply_4yr_deforestation_rule_to_year(image, y, class_ids)
    ));
  });

  return buildImageFromYearList(imageList);
};

// --------------------------------------------
// First and last year filters
// --------------------------------------------
var run_3yr_first = function(class_id, image) {
  image = ee.Image(image);
  class_id = ee.Number(class_id);

  var first_yr = getBand(image, 1985).where(
    getBand(image, 1985).neq(class_id)
      .and(getBand(image, 1986).eq(class_id))
      .and(getBand(image, 1987).eq(class_id)),
    class_id
  ).rename('classification_1985');

  var imageList = years.map(function(y) {
    y = ee.Number(y).toInt();

    return ee.Image(ee.Algorithms.If(
      y.eq(1985),
      first_yr,
      getBand(image, y).rename(bandName(y))
    ));
  });

  return buildImageFromYearList(imageList);
};

var run_3yr_last = function(class_id, image) {
  image = ee.Image(image);
  class_id = ee.Number(class_id);

  var last_yr = getBand(image, 2025).where(
    getBand(image, 2025).neq(class_id)
      .and(getBand(image, 2024).eq(class_id))
      .and(getBand(image, 2023).eq(class_id)),
    class_id
  ).rename('classification_2025');

  var imageList = years.map(function(y) {
    y = ee.Number(y).toInt();

    return ee.Image(ee.Algorithms.If(
      y.eq(2025),
      last_yr,
      getBand(image, y).rename(bandName(y))
    ));
  });

  return buildImageFromYearList(imageList);
};

// --------------------------------------------
// Exact original rule order
// --------------------------------------------
var deforestationRules = ee.List([
  ee.Dictionary({window: 4, classes: [3, 12, 12, 12, 21]}),
  ee.Dictionary({window: 4, classes: [3, 12, 12, 21, 21]}),

  ee.Dictionary({window: 3, classes: [3, 12, 21, 21]}),
  ee.Dictionary({window: 3, classes: [3, 12, 12, 21]}),
  ee.Dictionary({window: 3, classes: [3, 11, 21, 21]}),
  ee.Dictionary({window: 3, classes: [3, 11, 11, 3]}),
  ee.Dictionary({window: 4, classes: [3, 11, 11, 11, 3]}),
  ee.Dictionary({window: 3, classes: [3, 12, 21, 21]}),
  ee.Dictionary({window: 3, classes: [11, 12, 21, 21]}),
  ee.Dictionary({window: 3, classes: [12, 11, 21, 21]})
]);

var generalRules = ee.List([
  ee.Dictionary({window: 5, class_id: 3}),
  ee.Dictionary({window: 4, class_id: 3}),
  ee.Dictionary({window: 3, class_id: 3}),

  ee.Dictionary({window: 5, class_id: 12}),
  ee.Dictionary({window: 4, class_id: 12}),
  ee.Dictionary({window: 3, class_id: 12}),

  ee.Dictionary({window: 5, class_id: 11}),
  ee.Dictionary({window: 4, class_id: 11}),
  ee.Dictionary({window: 3, class_id: 11}),

  ee.Dictionary({window: 5, class_id: 21}),
  ee.Dictionary({window: 4, class_id: 21}),
  ee.Dictionary({window: 3, class_id: 21}),

  ee.Dictionary({window: 5, class_id: 33}),
  ee.Dictionary({window: 4, class_id: 33}),
  ee.Dictionary({window: 3, class_id: 33}),

  ee.Dictionary({window: 5, class_id: 25}),
  ee.Dictionary({window: 4, class_id: 25}),
  ee.Dictionary({window: 3, class_id: 25})
]);

// --------------------------------------------
// Apply rules preserving original order
// --------------------------------------------
var to_filter = ee.Image(deforestationRules.iterate(function(rule, img) {
  rule = ee.Dictionary(rule);
  img = ee.Image(img);

  return apply_deforestation_rule_series(
    img,
    rule.get('window'),
    rule.get('classes')
  );
}, classification));

print('Deforestation filters done');

to_filter = ee.Image(generalRules.iterate(function(rule, img) {
  rule = ee.Dictionary(rule);
  img = ee.Image(img);

  return apply_general_rule_series(
    img,
    rule.get('window'),
    rule.get('class_id')
  );
}, to_filter));

print('General temporal filters done');

// --------------------------------------------
// First year filters
// --------------------------------------------
to_filter = run_3yr_first(12, to_filter);
to_filter = run_3yr_first(3, to_filter);
to_filter = run_3yr_first(11, to_filter);

print('First-year filters done');

// --------------------------------------------
// Last year filter (inspection / optional export choice)
// --------------------------------------------
var filtered = run_3yr_last(21, to_filter);
print('Last-year filtered image ready');

// --------------------------------------------
// Small regeneration correction
// --------------------------------------------
var applySmallRegenerationCorrection = function(image) {
  image = ee.Image(image);

  var remapList = years.map(function(y) {
    y = ee.Number(y).toInt();

    return getBand(image, y)
      .remap([3, 11, 12, 21], [3, 3, 3, 21])
      .rename(bandName(y));
  });

  var remap_col = buildImageFromYearList(remapList);

  var reg_last = remap_col.select('classification_2025')
    .eq(3)
    .and(remap_col.select('classification_2024').eq(21));

  var reg_size = reg_last.selfMask()
    .connectedPixelCount(20, true)
    .reproject('epsg:4326', null, 30);

  var excludeReg = image.select('classification_2024')
    .updateMask(reg_size.lte(11).eq(1));

  var x25 = image.select('classification_2025').blend(excludeReg);

  var finalList = years.map(function(y) {
    y = ee.Number(y).toInt();

    return ee.Image(ee.Algorithms.If(
      y.eq(2025),
      x25.rename('classification_2025'),
      getBand(image, y).rename(bandName(y))
    ));
  });

  return buildImageFromYearList(finalList);
};

// Version without last-year filter
var output_without_last_year = applySmallRegenerationCorrection(to_filter);

// Version with last-year filter
var output_with_last_year = applySmallRegenerationCorrection(filtered);

// Choose which one will be exported, but keep the same export name
var output_to_export = ee.Image(ee.Algorithms.If(
  APPLY_LAST_YEAR_FILTER,
  output_with_last_year,
  output_without_last_year
));

print('Output to export', output_to_export);

// --------------------------------------------
// Visualization
// --------------------------------------------
if (SHOW_LAYERS) {
  Map.addLayer(classification.select('classification_' + previewYear), vis, 'before filters ' + previewYear, true);
  Map.addLayer(output_without_last_year.select('classification_' + previewYear), vis, 'after filters without last-year ' + previewYear, true);
  Map.addLayer(output_with_last_year.select('classification_' + previewYear), vis, 'after filters with last-year ' + previewYear, false);
}

// --------------------------------------------
// Export as GEE asset
// Keeps the original export naming pattern
// --------------------------------------------
Export.image.toAsset({
  image: output_to_export,
  description: inputFile + '_temporal_v' + outputVersion,
  assetId: out + inputFile + '_temporal_v' + outputVersion,
  pyramidingPolicy: {
    '.default': 'mode'
  },
  region: classification.geometry(),
  scale: 30,
  maxPixels: 1e13
});
```
