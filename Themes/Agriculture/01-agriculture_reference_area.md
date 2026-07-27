```javascript
// EXPORT AUXILIARY MASKS FOR AGRICULTURE CLASSIFICATION
// Suriname
// Exports:
// 1) candidate agriculture auto
// 2) accumulated all references no buffer - edited
// 3) accumulated all references with square buffer - edited
// 4) water mask combined
// 5) stable wetland MapBiomas
//
// Manual editing:
// - include areas in accumulated all references no buffer
// - exclude areas from accumulated all references no buffer
//
// Important:
// - manual polygons DO NOT affect candidate auto
// - square buffer is applied to the EDITED accumulated reference layer

// ---------------------------------------------------------------------
// OPTIONAL POLYGONS DRAWN IN THE EDITOR
// ---------------------------------------------------------------------

// EXAMPLES:
// var IncludeReferenceAreas = ee.FeatureCollection([ee.Feature(inclusion_areas)]);
// var ExcludeReferenceAreas = ee.FeatureCollection([ee.Feature(exclusion_areas)]);

// Leave empty if not using
var IncludeReferenceAreas = ee.FeatureCollection([inclusion_areas]);
var ExcludeReferenceAreas = ee.FeatureCollection([exclusion_areas]);

// ---------------------------------------------------------------------
// PARAMETERS
// ---------------------------------------------------------------------

var param = {
  country: 'SURINAME',
  region: 80201,
  version: 1,

  mapbiomasYears: {
    startYear: 1985,
    endYear: 2023
  },

  classes: {
    agriculture: 18,
    wetland: 11,
    water: 33
  },

  stableParams: {
    percentage: 100
  },

  candidateParams: {
    nearReferenceBuffer: 1000, // meters
    minAgreement: 1,
    useDynamicWorld: false
  },

  exportReferenceBuffer: {
    useSquareBuffer: true,
    distance: 100 // meters
  },

  waterMask: {
    usar: true,

    mapbiomas: {
      usar: true,
      minFrequency: 0.5
    },

    gsw: {
      usar: true,
      minOccurrence: 50
    }
  },

  wetlandMask: {
    usar: true,
    percentage: 100
  },

  mosaicParams: {
    year: 2025
  },

  export: {
    toAssetCandidateAuto: true,
    toAssetAccumulatedNoBuffer: true,
    toAssetAccumulatedBufferSquare: true,
    toAssetWaterMask: true,
    toAssetStableWetland: true,

    toDriveCandidateAuto: false,
    toDriveAccumulatedNoBuffer: false,
    toDriveAccumulatedBufferSquare: false,
    toDriveWaterMask: false,
    toDriveStableWetland: false,

    driveFolder: 'EXPORT-MAPBIOMAS'
  }
};


/**
 * ------------------------------------------------------------
 * ASSETS
 * ------------------------------------------------------------
 */
var assetRegionClasVector =
  'projects/mapbiomas-suriname/assets/Archive_Suriname/classification_regions_suriname';

var mapbiomasCollection =
  'projects/mapbiomas-raisg/COLECCION6/INTEGRACION/mapbiomas_raisg_panamazonia_collection6_integration-0-2';

var outputBase =
  'projects/mapbiomas-suriname/assets/AGRICULTURE/COLLECTION-1/MASKS/';

var outputCandidateAuto =
  outputBase + 'AGRICULTURE-CANDIDATE-AUTO-' +
  param.country + '-' + param.region + '-' + param.version;

var outputAccumulatedNoBuffer =
  outputBase + 'AGRICULTURE-ACCUMULATED-NOBUFFER-' +
  param.country + '-' + param.region + '-' + param.version;

var outputAccumulatedBufferSquare =
  outputBase + 'AGRICULTURE-ACCUMULATED-BUFFER-' +
  param.country + '-' + param.region + '-' + param.version;

var outputWaterMask =
  outputBase + 'WATER-MASK-COMBINED-' +
  param.country + '-' + param.region + '-' + param.version;

var outputStableWetland =
  outputBase + 'WETLAND-STABLE-MAPBIOMAS-' +
  param.country + '-' + param.region + '-' + param.version;


/**
 * ------------------------------------------------------------
 * REGION
 * ------------------------------------------------------------
 */
var setVersion = function(item) { return item.set('version', 1); };

var SelRegion = ee.FeatureCollection(assetRegionClasVector)
  .filter(ee.Filter.eq('id_regionC', param.region));

print('Selected region', SelRegion);

Map.addLayer(SelRegion, {}, 'classification region', false);

var regionMask = SelRegion
  .map(setVersion)
  .reduceToImage(['version'], ee.Reducer.first())
  .selfMask();


/**
 * ------------------------------------------------------------
 * HELPERS
 * ------------------------------------------------------------
 */
function rangeBands(start, end) {
  var list = [];
  for (var y = start; y <= end; y++) {
    list.push('classification_' + y);
  }
  return list;
}

function getStableClass(image, classId, startYear, endYear, percentage) {
  var bandNames = rangeBands(startYear, endYear);
  var nBands = bandNames.length;
  var threshold = Math.ceil(nBands * (percentage / 100));

  var result = ee.Image(0);

  bandNames.forEach(function(bandName) {
    result = result.add(image.select(bandName).eq(classId));
  });

  return result.gte(threshold).selfMask();
}

function getFrequencyMask(image, classId, startYear, endYear, minFrequency) {
  var bandNames = rangeBands(startYear, endYear);
  var nBands = bandNames.length;

  var freq = ee.Image(0);

  bandNames.forEach(function(bandName) {
    freq = freq.add(image.select(bandName).eq(classId));
  });

  freq = freq.divide(nBands);

  return freq.gte(minFrequency).selfMask();
}

// Normalize drawn geometries or feature collections into FeatureCollection
function normalizeToFeatureCollection(obj) {
  var typeName = ee.Algorithms.ObjectType(obj);

  return ee.FeatureCollection(
    ee.Algorithms.If(
      ee.String(typeName).compareTo('FeatureCollection').eq(0),
      obj,
      ee.Algorithms.If(
        ee.String(typeName).compareTo('Feature').eq(0),
        ee.FeatureCollection([obj]),
        ee.Algorithms.If(
          ee.String(typeName).compareTo('Geometry').eq(0),
          ee.FeatureCollection([ee.Feature(obj)]),
          ee.FeatureCollection([])
        )
      )
    )
  );
}

function fcToMask(fc) {
  fc = normalizeToFeatureCollection(fc);

  return ee.Image().byte().paint({
    featureCollection: fc,
    color: 1
  }).selfMask();
}

function applyManualEdits(referenceImage) {
  var edited = referenceImage;

  var includeFC = normalizeToFeatureCollection(IncludeReferenceAreas);
  var excludeFC = normalizeToFeatureCollection(ExcludeReferenceAreas);

  var includeMask = ee.Image(0);
  var excludeMask = ee.Image(0);

  if (includeFC.size().getInfo() > 0) {
    includeMask = fcToMask(includeFC);
    edited = edited.unmask(0).where(includeMask, 1);
  }

  if (excludeFC.size().getInfo() > 0) {
    excludeMask = fcToMask(excludeFC);
    edited = edited.unmask(0).where(excludeMask, 0);
  }

  edited = edited.selfMask().updateMask(regionMask);

  return {
    edited: edited,
    includeMask: includeMask.selfMask(),
    excludeMask: excludeMask.selfMask()
  };
}

function applySquareBufferToReference(referenceImage, bufferDistanceMeters) {
  var binary = referenceImage.gt(0);

  var buffered = binary.focal_max({
    kernel: ee.Kernel.square({
      radius: bufferDistanceMeters,
      units: 'meters',
      normalize: false
    }),
    iterations: 1
  });

  return buffered.selfMask().updateMask(regionMask);
}


/**
 * ------------------------------------------------------------
 * MAPBIOMAS COLLECTION
 * ------------------------------------------------------------
 */
var integration = ee.Image(mapbiomasCollection).updateMask(regionMask);


/**
 * ------------------------------------------------------------
 * LANDSAT MOSAIC FOR VISUAL INSPECTION
 * ------------------------------------------------------------
 */
var mosaic = ee.ImageCollection('projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2')
  .merge(
    ee.ImageCollection('projects/mapbiomas-raisg/MOSAICOS/mosaics-2')
      .filter(ee.Filter.gte('year', 2022))
  )
  .filter(ee.Filter.eq('year', param.mosaicParams.year))
  .filterBounds(SelRegion)
  .mosaic()
  .clip(SelRegion);

Map.addLayer(
  mosaic,
  {
    bands: ['swir1_median', 'nir_median', 'red_median'],
    gain: [0.08, 0.06, 0.2]
  },
  'Landsat mosaic ' + param.mosaicParams.year,
  false
);


/**
 * ------------------------------------------------------------
 * MAPBIOMAS AGRICULTURE REFERENCES
 * ------------------------------------------------------------
 */
var stableAgriMapbiomas = getStableClass(
  integration,
  param.classes.agriculture,
  param.mapbiomasYears.startYear,
  param.mapbiomasYears.endYear,
  param.stableParams.percentage
).updateMask(regionMask);

Map.addLayer(
  stableAgriMapbiomas,
  {palette: ['#00ff00']},
  'stable agriculture MapBiomas',
  false
);

var agriAccumMapbiomas = integration.eq(param.classes.agriculture)
  .reduce('max')
  .selfMask()
  .updateMask(regionMask);

Map.addLayer(
  agriAccumMapbiomas,
  {palette: ['#8ddf3f']},
  'accumulated agriculture MapBiomas',
  true
);


/**
 * ------------------------------------------------------------
 * STABLE WETLAND MASK - MAPBIOMAS
 * ------------------------------------------------------------
 */
var stableWetlandMapbiomas = getStableClass(
  integration,
  param.classes.wetland,
  param.mapbiomasYears.startYear,
  param.mapbiomasYears.endYear,
  param.wetlandMask.percentage
).updateMask(regionMask);

Map.addLayer(
  stableWetlandMapbiomas,
  {palette: ['#7e57c2']},
  'stable wetland MapBiomas',
  false
);


/**
 * ------------------------------------------------------------
 * WATER MASKS
 * ------------------------------------------------------------
 */
var waterMapbiomas = getFrequencyMask(
  integration,
  param.classes.water,
  param.mapbiomasYears.startYear,
  param.mapbiomasYears.endYear,
  param.waterMask.mapbiomas.minFrequency
).updateMask(regionMask);

Map.addLayer(
  waterMapbiomas,
  {palette: ['#0000ff']},
  'water mask MapBiomas frequency',
  false
);

var gsw = ee.Image('JRC/GSW1_4/GlobalSurfaceWater');
var gswOccurrence = gsw.select('occurrence').clip(SelRegion);

var waterGSW = gswOccurrence
  .gte(param.waterMask.gsw.minOccurrence)
  .selfMask()
  .updateMask(regionMask);

Map.addLayer(
  waterGSW,
  {min: 0, max: 100, palette: ['ffffff', '99d9ff', '0000ff']},
  'water mask GSW occurrence',
  false
);

var waterMaskCombined = ee.Image(0);

if (param.waterMask.usar) {
  if (param.waterMask.mapbiomas.usar) {
    waterMaskCombined = waterMaskCombined.where(waterMapbiomas, 1);
  }

  if (param.waterMask.gsw.usar) {
    waterMaskCombined = waterMaskCombined.where(waterGSW, 1);
  }

  waterMaskCombined = waterMaskCombined.selfMask().updateMask(regionMask);
}

Map.addLayer(
  waterMaskCombined,
  {palette: ['#00bfff']},
  'water mask combined',
  false
);


/**
 * ------------------------------------------------------------
 * POTAPOV CROPLAND REFERENCES
 * ------------------------------------------------------------
 */
var crop2003 = ee.ImageCollection('users/potapovpeter/Global_cropland_2003')
  .mosaic().clip(SelRegion).gt(0);

var crop2007 = ee.ImageCollection('users/potapovpeter/Global_cropland_2007')
  .mosaic().clip(SelRegion).gt(0);

var crop2011 = ee.ImageCollection('users/potapovpeter/Global_cropland_2011')
  .mosaic().clip(SelRegion).gt(0);

var crop2015 = ee.ImageCollection('users/potapovpeter/Global_cropland_2015')
  .mosaic().clip(SelRegion).gt(0);

var crop2019 = ee.ImageCollection('users/potapovpeter/Global_cropland_2019')
  .mosaic().clip(SelRegion).gt(0);

var cropStable = crop2003
  .and(crop2007)
  .and(crop2011)
  .and(crop2015)
  .and(crop2019)
  .selfMask()
  .updateMask(regionMask);

Map.addLayer(
  cropStable,
  {palette: ['#4caf50']},
  'Potapov stable cropland',
  false
);

// Optional dynamics
var cropLoss = ee.ImageCollection('users/potapovpeter/Global_cropland_loss')
  .mosaic().clip(SelRegion).gt(0);

var cropGain = ee.ImageCollection('users/potapovpeter/Global_cropland_gain')
  .mosaic().clip(SelRegion).gt(0);

var cropDyn = ee.Image(0)
  .where(cropStable, 1)
  .where(cropGain, 2)
  .where(cropLoss, 3)
  .selfMask()
  .updateMask(regionMask);

Map.addLayer(
  cropDyn,
  {min: 1, max: 3, palette: ['#4caf50', '#ffd54f', '#ef5350']},
  'Potapov cropland dynamics',
  false
);


/**
 * ------------------------------------------------------------
 * ESRI / ESA / DYNAMIC WORLD
 * ------------------------------------------------------------
 */
var esri_lc = ee.ImageCollection('projects/sat-io/open-datasets/landcover/ESRI_Global-LULC_10m');
var esriCrops = esri_lc
  .mosaic()
  .eq(5)
  .selfMask()
  .updateMask(regionMask);

Map.addLayer(
  esriCrops,
  {palette: ['#fdd835']},
  'ESRI crops',
  false
);

var esa_lc = ee.ImageCollection('ESA/WorldCover/v100');
var esaCropland = esa_lc
  .first()
  .eq(40)
  .selfMask()
  .updateMask(regionMask);

Map.addLayer(
  esaCropland,
  {palette: ['#ff9800']},
  'ESA cropland',
  false
);

var dw = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1')
  .filterDate('2020-01-01', '2023-12-31')
  .filterBounds(SelRegion);

var dwMode = dw.select('label').mode().clip(SelRegion);
var dwCrops = dwMode
  .eq(4)
  .selfMask()
  .updateMask(regionMask);

Map.addLayer(
  dwCrops,
  {palette: ['#cddc39']},
  'Dynamic World crops',
  false
);


/**
 * ------------------------------------------------------------
 * ACCUMULATED ALL REFERENCES - NO BUFFER
 * ------------------------------------------------------------
 */
var accumulatedAllRefs = ee.Image(0)
  .where(stableAgriMapbiomas, 1)
  .where(agriAccumMapbiomas, 1)
  .where(cropStable, 1)
  .where(esriCrops, 1)
  .where(esaCropland, 1);

if (param.candidateParams.useDynamicWorld) {
  accumulatedAllRefs = accumulatedAllRefs.where(dwCrops, 1);
}

accumulatedAllRefs = accumulatedAllRefs
  .selfMask()
  .updateMask(regionMask);

if (param.waterMask.usar) {
  accumulatedAllRefs = accumulatedAllRefs
    .where(waterMaskCombined, 0)
    .selfMask()
    .updateMask(regionMask);
}

if (param.wetlandMask.usar) {
  accumulatedAllRefs = accumulatedAllRefs
    .where(stableWetlandMapbiomas, 0)
    .selfMask()
    .updateMask(regionMask);
}

// Manual edits ONLY here
var editResult = applyManualEdits(accumulatedAllRefs);
var accumulatedAllRefsEdited = editResult.edited;
var includeMaskApplied = editResult.includeMask;
var excludeMaskApplied = editResult.excludeMask;

// Difference layer to inspect applied edits
var manualDifference = ee.Image(0)
  .where(accumulatedAllRefsEdited.neq(accumulatedAllRefs), 1)
  .selfMask();

Map.addLayer(
  accumulatedAllRefs,
  {palette: ['#999999']},
  'accumulated all references no buffer - original',
  false
);

Map.addLayer(
  accumulatedAllRefsEdited,
  {palette: ['#ff00ff']},
  'accumulated all references no buffer - edited',
  true
);

Map.addLayer(
  manualDifference,
  {palette: ['#ffff00']},
  'manual edit difference',
  true
);

// These layers show exactly where the masks were rasterized
Map.addLayer(
  includeMaskApplied,
  {palette: ['#00ff00']},
  'include mask applied',
  false
);

Map.addLayer(
  excludeMaskApplied,
  {palette: ['#ff0000']},
  'exclude mask applied',
  false
);

// Optional vector display
if (normalizeToFeatureCollection(IncludeReferenceAreas).size().getInfo() > 0) {
  Map.addLayer(
    normalizeToFeatureCollection(IncludeReferenceAreas),
    {color: '00ff00'},
    'include reference areas',
    true
  );
}

if (normalizeToFeatureCollection(ExcludeReferenceAreas).size().getInfo() > 0) {
  Map.addLayer(
    normalizeToFeatureCollection(ExcludeReferenceAreas),
    {color: 'ff0000'},
    'exclude reference areas',
    true
  );
}

// Square buffer applied to the EDITED exported reference layer
var accumulatedAllRefsBufferSquare = ee.Image(
  ee.Algorithms.If(
    param.exportReferenceBuffer.useSquareBuffer,
    applySquareBufferToReference(
      accumulatedAllRefsEdited,
      param.exportReferenceBuffer.distance
    ),
    accumulatedAllRefsEdited
  )
);

Map.addLayer(
  accumulatedAllRefsBufferSquare,
  {palette: ['#ff66cc']},
  'accumulated all references square buffer',
  false
);


/**
 * ------------------------------------------------------------
 * CORE REFERENCE SEEDS
 * conservative core
 * ------------------------------------------------------------
 */
var referenceSeeds = ee.Image(0)
  .where(stableAgriMapbiomas, 1)
  .where(agriAccumMapbiomas, 1)
  .where(cropStable, 1)
  .selfMask()
  .updateMask(regionMask);

if (param.waterMask.usar) {
  referenceSeeds = referenceSeeds
    .where(waterMaskCombined, 0)
    .selfMask()
    .updateMask(regionMask);
}

if (param.wetlandMask.usar) {
  referenceSeeds = referenceSeeds
    .where(stableWetlandMapbiomas, 0)
    .selfMask()
    .updateMask(regionMask);
}

Map.addLayer(
  referenceSeeds,
  {palette: ['#00ffcc']},
  'reference seeds',
  false
);


/**
 * ------------------------------------------------------------
 * EXTRA AGREEMENT SOURCES
 * ------------------------------------------------------------
 */
var extraAgreement = ee.Image(0)
  .addBands(ee.Image(0).where(esriCrops, 1).rename('esri'))
  .addBands(ee.Image(0).where(esaCropland, 1).rename('esa'))
  .addBands(ee.Image(0).where(agriAccumMapbiomas, 1).rename('mapbiomas_acc'))
  .addBands(ee.Image(0).where(cropStable, 1).rename('potapov'));

if (param.candidateParams.useDynamicWorld) {
  extraAgreement = extraAgreement.addBands(
    ee.Image(0).where(dwCrops, 1).rename('dw')
  );
}

var agreementScore = extraAgreement.reduce('sum').updateMask(regionMask);

Map.addLayer(
  agreementScore.updateMask(agreementScore.gt(0)),
  {
    min: 1,
    max: param.candidateParams.useDynamicWorld ? 5 : 4,
    palette: ['fff829', 'ffce45', 'ff920a', 'ff6e19', 'ff0000']
  },
  'agreement score',
  false
);


/**
 * ------------------------------------------------------------
 * AUTOMATIC CANDIDATE AGRICULTURE
 * ------------------------------------------------------------
 */
var nearReference = referenceSeeds
  .focal_max({
    radius: param.candidateParams.nearReferenceBuffer,
    units: 'meters'
  })
  .selfMask()
  .updateMask(regionMask);

Map.addLayer(
  nearReference,
  {palette: ['#00ffff']},
  'near reference',
  false
);

var candidateAgriculture = nearReference
  .and(agreementScore.gte(param.candidateParams.minAgreement))
  .selfMask()
  .updateMask(regionMask);

if (param.waterMask.usar) {
  candidateAgriculture = candidateAgriculture
    .where(waterMaskCombined, 0)
    .selfMask()
    .updateMask(regionMask);
}

if (param.wetlandMask.usar) {
  candidateAgriculture = candidateAgriculture
    .where(stableWetlandMapbiomas, 0)
    .selfMask()
    .updateMask(regionMask);
}

Map.addLayer(
  candidateAgriculture,
  {palette: ['#ff8800']},
  'candidate agriculture auto',
  true
);


/**
 * ------------------------------------------------------------
 * EXPORTS - CANDIDATE AUTO
 * ------------------------------------------------------------
 */
if (param.export.toDriveCandidateAuto) {
  Export.image.toDrive({
    image: candidateAgriculture.toUint8(),
    description: 'AGRICULTURE-CANDIDATE-AUTO-' + param.country + '-' + param.region + '-' + param.version,
    scale: 30,
    maxPixels: 1e13,
    folder: param.export.driveFolder,
    region: SelRegion.geometry().bounds(),
    shardSize: 1024
  });
}

if (param.export.toAssetCandidateAuto) {
  Export.image.toAsset({
    image: candidateAgriculture.toUint8(),
    description: 'AGRICULTURE-CANDIDATE-AUTO-' + param.country + '-' + param.region + '-' + param.version,
    assetId: outputCandidateAuto,
    scale: 30,
    pyramidingPolicy: {
      '.default': 'mode'
    },
    maxPixels: 1e13,
    region: SelRegion.geometry().bounds()
  });
}


/**
 * ------------------------------------------------------------
 * EXPORTS - ACCUMULATED ALL REFERENCES NO BUFFER
 * ------------------------------------------------------------
 */
if (param.export.toDriveAccumulatedNoBuffer) {
  Export.image.toDrive({
    image: accumulatedAllRefsEdited.toUint8(),
    description: 'AGRICULTURE-ACCUMULATED-NOBUFFER-' + param.country + '-' + param.region + '-' + param.version,
    scale: 30,
    maxPixels: 1e13,
    folder: param.export.driveFolder,
    region: SelRegion.geometry().bounds(),
    shardSize: 1024
  });
}

if (param.export.toAssetAccumulatedNoBuffer) {
  Export.image.toAsset({
    image: accumulatedAllRefsEdited.toUint8(),
    description: 'AGRICULTURE-ACCUMULATED-NOBUFFER-' + param.country + '-' + param.region + '-' + param.version,
    assetId: outputAccumulatedNoBuffer,
    scale: 30,
    pyramidingPolicy: {
      '.default': 'mode'
    },
    maxPixels: 1e13,
    region: SelRegion.geometry().bounds()
  });
}


/**
 * ------------------------------------------------------------
 * EXPORTS - ACCUMULATED ALL REFERENCES BUFFER SQUARE
 * ------------------------------------------------------------
 */
if (param.export.toDriveAccumulatedBufferSquare) {
  Export.image.toDrive({
    image: accumulatedAllRefsBufferSquare.toUint8(),
    description: 'AGRICULTURE-ACCUMULATED-BUFFER-' + param.country + '-' + param.region + '-' + param.version,
    scale: 30,
    maxPixels: 1e13,
    folder: param.export.driveFolder,
    region: SelRegion.geometry().bounds(),
    shardSize: 1024
  });
}

if (param.export.toAssetAccumulatedBufferSquare) {
  Export.image.toAsset({
    image: accumulatedAllRefsBufferSquare.toUint8(),
    description: 'AGRICULTURE-ACCUMULATED-BUFFER-' + param.country + '-' + param.region + '-' + param.version,
    assetId: outputAccumulatedBufferSquare,
    scale: 30,
    pyramidingPolicy: {
      '.default': 'mode'
    },
    maxPixels: 1e13,
    region: SelRegion.geometry().bounds()
  });
}


/**
 * ------------------------------------------------------------
 * EXPORTS - WATER MASK COMBINED
 * ------------------------------------------------------------
 */
if (param.export.toDriveWaterMask) {
  Export.image.toDrive({
    image: waterMaskCombined.toUint8(),
    description: 'WATER-MASK-COMBINED-' + param.country + '-' + param.region + '-' + param.version,
    scale: 30,
    maxPixels: 1e13,
    folder: param.export.driveFolder,
    region: SelRegion.geometry().bounds(),
    shardSize: 1024
  });
}

if (param.export.toAssetWaterMask) {
  Export.image.toAsset({
    image: waterMaskCombined.toUint8(),
    description: 'WATER-MASK-COMBINED-' + param.country + '-' + param.region + '-' + param.version,
    assetId: outputWaterMask,
    scale: 30,
    pyramidingPolicy: {
      '.default': 'mode'
    },
    maxPixels: 1e13,
    region: SelRegion.geometry().bounds()
  });
}


/**
 * ------------------------------------------------------------
 * EXPORTS - STABLE WETLAND MAPBIOMAS
 * ------------------------------------------------------------
 */
if (param.export.toDriveStableWetland) {
  Export.image.toDrive({
    image: stableWetlandMapbiomas.toUint8(),
    description: 'WETLAND-STABLE-MAPBIOMAS-' + param.country + '-' + param.region + '-' + param.version,
    scale: 30,
    maxPixels: 1e13,
    folder: param.export.driveFolder,
    region: SelRegion.geometry().bounds(),
    shardSize: 1024
  });
}

if (param.export.toAssetStableWetland) {
  Export.image.toAsset({
    image: stableWetlandMapbiomas.toUint8(),
    description: 'WETLAND-STABLE-MAPBIOMAS-' + param.country + '-' + param.region + '-' + param.version,
    assetId: outputStableWetland,
    scale: 30,
    pyramidingPolicy: {
      '.default': 'mode'
    },
    maxPixels: 1e13,
    region: SelRegion.geometry().bounds()
  });
}
```
