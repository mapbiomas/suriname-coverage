```javascript
var param = {
  country: 'SURINAME',
  regions: [80202],
  roi: 'AllMiningAreasGuyanas',
  trees: 80, 
  years: [
    //1985,
    1986,1987,1988,1989,1990,1991,1992,1993,1994,1995,1996,1997,1998,1999,
    2000,2001,2002,2003,2004,2005,2006,2007,2008,2009,2010,2011,2012,
    2013,2014,2015,
    2016,2017,2018,2019,2020,
    2021,2022,2023,2024, 2025
  ],
  variables: [
    // e.g., 'ndvi_median', 'swir1_median'
  ],
  yearsPreview: [1990, 2010, 2024, 2025], // only these years will be visualized (all hidden by default)
  driveFolder: 'RF-PRELIMINAR-CLASSIFICATION',
  samplesVersion: 1,
  outputVersion: 1,
  newAreas: [ newAreas ], // optional (e.g., new polygons to merge into ROI)
  additionalSamples: { 
    // polygons must have 'start' and 'finish' properties to control year ranges
    polygons: [ /* feature collections or features */ ],
    classes:  [27],/* e.g., 27 */ 
    points:   [2000],/* e.g., 2000 */
  }
};

var Mine = function(param) {
  var _this = this;

  var assets = {
    mosaics1: ee.ImageCollection('projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2'),
    mosaics2: ee.ImageCollection('projects/mapbiomas-raisg/MOSAICOS/mosaics-2'),
    roi: ee.FeatureCollection('users/apoiotecnicoraisg/' + param.roi),
    palette: require('users/mapbiomas/modules:Palettes.js').get('classification6'),
    regions: ee.FeatureCollection('projects/mapbiomas-suriname/assets/Archive_Suriname/classification_regions_suriname')
  };

  // Helper: add a layer always hidden
  var addHidden = function(imgOrFc, vis, name) {
    Map.addLayer(imgOrFc, vis || {}, name, false);
  };

  // Initialize pipeline
  var init = function(param) {
    var country = param.country;
    var regionIds = param.regions;
    var newAreas = param.newAreas;
    var trees = param.trees;
    var years = param.years;
    var variables = param.variables;
    var samplesVersion = param.samplesVersion;
    var outputVersion = param.outputVersion;
    var additionalSamples = param.additionalSamples;

    var outputPath = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/classification/';
    var samplesPath = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/SAMPLES/';

    var newMiningAreas = newAreas ? ee.FeatureCollection(newAreas) : null;

    // Target regions (classification regions)
    var region = assets.regions.filter(ee.Filter.inList('id_regionC', regionIds));

    // Join both mosaic sources and spatially filter by region
    var mosaics = assets.mosaics1.merge(assets.mosaics2).filter(ee.Filter.bounds(region));

    // Draw region and ROI once (hidden)
    addHidden(region.style({color:'000000', fillColor:'00000000'}), {}, 'Regions');
    var miningAreaOnce = newMiningAreas ? assets.roi.merge(newMiningAreas) : assets.roi;
    addHidden(miningAreaOnce.style({color:'cyan', fillColor:'00000000'}), {}, 'ROI (merged)');

    // Multiband classification image (one band per year)
    var classified;

    years.forEach(function(year, i) {
      var showThisYear = param.yearsPreview.indexOf(year) > -1;

      // Yearly mosaic (median) for classification
      var yearMosaic = mosaics
        .filterMetadata('year', 'equals', year)
        .median();

      // ROI for classification (merged if newAreas provided)
      var miningArea = miningAreaOnce;

      // Load training samples for the given year
      var samplesAssetName = param.country + '-' + [param.regions].join('-') + '-' + year + '-' + samplesVersion;
      var samples = _this.loadSamples(yearMosaic.clip(region), variables, samplesPath, samplesAssetName);

      // Optional: add additional samples constrained by polygons’ [start, finish] year properties
      if (additionalSamples.polygons.length > 0) {
        var newSamples = _this.resampleCover(yearMosaic, additionalSamples, year);
        samples = newSamples ? samples.merge(newSamples) : samples;
      }

      // Classify
      var bandName = 'classification_' + year;
      var classification = _this.applyClassifier(yearMosaic, variables, trees, samples, bandName, miningArea);

      // Stack yearly band
      classified = i === 0 ? classification : classified.addBands(classification);

      // Visualization (only for preview years, all layers hidden)
      if (showThisYear) {
        addHidden(
          yearMosaic,
          { min: 200, max: 5000, bands: 'swir1_median,nir_median,red_median' },
          'MOSAIC ' + year + ' (preview)'
        );

        // Style training points (example: mining=30 in dark red; adjust as needed)
        var styledPoints = samples.map(function(pt) {
          var ref = ee.Number(pt.get('reference'));
          var isMining = ref.eq(30);
          var color = ee.Algorithms.If(isMining, 'darkred', 'BBBBBB');
          return pt.set({ style: { color: color } });
        });

        addHidden(
          styledPoints.style({ styleProperty: 'style', width: 0.5 }),
          {},
          'SAMPLES ' + year + ' (preview)'
        );

        // Classification preview (example highlights class 30)
        addHidden(
          classification.updateMask(classification.eq(30)),
          { palette: 'darkred' },
          'CLASS ' + year + ' (preview)'
        );
      }
    });

    // Metadata and export
    var outputName = country + '-' + regionIds.join('-') + '-' + outputVersion;

    classified = classified.set({
      regions: regionIds,
      country: country,
      version: outputVersion,
      trees: trees,
      samples_version: samplesVersion,
      description: 'preliminary mining classification'
    });

    Export.image.toAsset({
      image: classified,
      description: outputName,
      assetId: outputPath + outputName,
      scale: 30,
      maxPixels: 1e13,
      region: region.geometry().bounds(),
      pyramidingPolicy: { '.default': 'mode' }
    });
  };

  // Load training samples for a year (ensuring bands align with selected variables)
  this.loadSamples = function(mosaic, variables, path, assetname) {
    mosaic = variables.length > 0 ? mosaic.select(variables) : mosaic;
    var bands = mosaic.bandNames();
    var contained = bands.containsAll(ee.List(variables));

    return ee.FeatureCollection(
      ee.Algorithms.If(
        contained,
        ee.FeatureCollection(path + assetname),
        null
      )
    )
    .select(bands.add('reference'))
    .filter(ee.Filter.notNull(bands));
  };

  // Count distinct classes present in the sample set
  this.checkClassesInSamples = function(mosaic, variables, samples) {
    mosaic = variables.length > 0 ? mosaic.select(variables) : mosaic;
    var bands = mosaic.bandNames();
    var contained = bands.containsAll(ee.List(variables));

    var nClasSample = ee.List(
      ee.Algorithms.If(
        contained,
        samples.reduceColumns(ee.Reducer.toList(), ['reference']).get('list'),
        null
      )
    );
    return nClasSample.reduce(ee.Reducer.countDistinct());
  };

  // Additional samples by polygons and year ranges (using polygon properties 'start' and 'finish')
  this.resampleCover = function(mosaic, additionalSamples, year) {
    var polygons = additionalSamples.polygons,
        classIds = additionalSamples.classes,
        points = additionalSamples.points,
        newSamples = [];

    polygons.forEach(function(polygon, i) {
      var start = polygon.get('start').getInfo();
      var finish = polygon.get('finish').getInfo();

      if (year <= finish && year >= start) {
        var newSample = mosaic.sample({
          numPixels: points[i],
          region: polygon.geometry(),
          scale: 30,
          projection: 'EPSG:4326',
          seed: 1,
          tileScale: 8,
          geometries: true
        })
        .map(function(item) { return item.set('reference', classIds[i]); });

        newSamples.push(newSample);
      }
    });

    return ee.FeatureCollection(newSamples).flatten();
  };

  // Train and apply RF classifier; returns a single-band image named as outputband
  this.applyClassifier = function(mosaic, variables, trees, samples, outputband, region) {
    mosaic = variables.length > 0 ? mosaic.select(variables) : mosaic;
    var bands = mosaic.bandNames();
    var contained = bands.containsAll(ee.List(variables));
    var nClasses = _this.checkClassesInSamples(mosaic, variables, samples);

    var classifier = ee.Classifier.smileRandomForest({
      numberOfTrees: trees,
      variablesPerSplit: 1
    });

    classifier = ee.Classifier(
      ee.Algorithms.If(
        contained,
        ee.Algorithms.If(
          ee.Algorithms.IsEqual(nClasses, 1),
          null,
          classifier.train(samples, 'reference', bands)
        ),
        null
      )
    );

    var image = mosaic.classify(classifier);
    image = outputband ? image.select(['classification'], [outputband]) : image;
    var maskBand = outputband ? ee.Image(27).rename(outputband) : ee.Image(27);

    return ee.Image(
      ee.Algorithms.If(
        contained,
        ee.Algorithms.If(ee.Algorithms.IsEqual(nClasses, 1), maskBand, image),
        maskBand
      )
    )
    .unmask(27)
    .clip(region)
    .toByte();
  };

  // Run
  return init(param);
};

new Mine(param);

```
