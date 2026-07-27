```javascript
// -- -- -- -- 05_incidence
// post-processing filter: filter spurious transitions by using the number of changes, connection number, and mode reducer
// barbara.silva@ipam.org.br and dhemerson.costa@ipam.org.br
// joaquim.pereira@ipam.org.br Update for Suriname Collection 1
// Import mapbiomas color schema 
var vis = {
    min: 0,
    max: 62,
    palette:require('users/mapbiomas/modules:Palettes.js').get('classification8')
};

// Set root directory 
var root = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/classification/';
var out = 'projects/mapbiomas-suriname/assets/LAND-COVER/COLLECTION-1/GENERAL/classification-ft/';

// Set metadata
var inputVersion = '2'; //gapfill asset
var outputVersion = '1';
var thresholdEvents = 13;

// Define input file
var inputFile = 'GUYANA-50202-' + inputVersion;

// Load input classification
var classificationInput = ee.Image(root + inputFile);
print('Input classification', classificationInput);
Map.addLayer(classificationInput.select(['classification_2024']), vis, 'Input classification');

// Aggregate MapBiomas classes in level 2
var originalClasses = [
    3,       // Forest
    11, 12,  // Wetlands, Grasslands
    21,      // Mosaic
    25,      // Non-vegetated areas
    33,      // Water
    27       // Non-observed
];

var aggregatedClasses = [
    2,       // Forest
    2, 2,    // Wetlands, Grasslands
    1,       // Mosaic
    1,       // Non-vegetated areas
    1,       // Water
    7        // Non-observed
];

var classificationAggregated = ee.Image([]);

// Remove 'forest' class from incidents filter (avoid over-estimation)
var classification_remap = classificationInput.updateMask(classificationInput.neq(3));

// Set the list of years to be filtered
ee.List.sequence({'start': 1985, 'end': 2025}).getInfo()
    .forEach(function(year) {
      
        // Get year [i]
        var classificationYear = classification_remap.select(['classification_' + year])
            // Remap classes
            .remap(originalClasses, aggregatedClasses)
            .rename('classification_' + year);
            
        // Insert into aggregated classification
        classificationAggregated = classificationAggregated.addBands(classificationYear);
    });

classificationAggregated = classificationAggregated.updateMask(classificationAggregated.neq(0));

// Compute number of classes and changes
var numChanges = classificationAggregated.reduce(ee.Reducer.countRuns()).subtract(1).rename('number_of_changes');
Map.addLayer(numChanges, {palette: ["#C8C8C8", "#FED266", "#FBA713", "#cb701b", "#a95512", "#662000", "#cb181d"],
                                  min: 0, max: 15}, 'number of changes', false);

// Get the count of connections
var connectedNumChanges = numChanges.connectedPixelCount({
    'maxSize': 100,
    'eightConnected': true
});

// Compute the mode of the pixel values in the time series
var modeImage = classification_remap.reduce(ee.Reducer.mode());

// Get border pixels (high geolocation RMSE) to be masked by the mode (7 pixels = 0,6 ha)
var borderMask = connectedNumChanges.lte(7).and(numChanges.gt(10));
borderMask = borderMask.updateMask(borderMask.eq(1));

// Get borders to rectify
var rectBorder = modeImage.updateMask(borderMask);
var rectAll = modeImage.updateMask(connectedNumChanges.gt(7).and(numChanges.gte(thresholdEvents)));

// Blend masks
var incidentsMask = rectBorder.blend(rectAll).toByte();

// Apply the corrections
var correctedClassification = classificationInput.blend(incidentsMask);

Map.addLayer(correctedClassification.select(['classification_2024']), vis, 'Corrected classification');
print('Output classification', correctedClassification);

// Export as GEE asset
Export.image.toAsset({
    'image': correctedClassification,
    'description': inputFile + '_incidence_v' + outputVersion,
    'assetId': out +  inputFile + '_incidence_v' + outputVersion,
    'pyramidingPolicy': {
        '.default': 'mode'
    },
    'region': correctedClassification.geometry(),
    'scale': 30,
    'maxPixels': 1e13
});
```
