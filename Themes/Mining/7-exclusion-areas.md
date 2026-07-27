```javascript
var country       = 'SURINAME';

var collection    = 'clasification-ft';

var assetName     = 'SURINAME-80202-11114';

var outputName    = 'SURINAME-80202-11115';
var pais = 'SURINAME';
var version_output = 11114;
var code_region = 80202
var mosaicRegions = [802];

var years         = [1985, 
  1986,1987,1988,1989,
  1990,
  1991,1992,1993,1994,1995,1996,1997,1998,1999,
  2000,
  2001,2002,2003,
  2004,
  2005,
  2006,2007,2008,2009,
  2010,
  2011,2012,
  2013,2014,2015,
  2016,2017,2018,2019,2020,
  2021,2022,
  2023,2024,2025];

var preview       = [2010, 2025];

var remapZones   = [ polygons ];




var commonPath   = 'projects/mapbiomas-suriname';

var assetPath    = 'projects/mapbiomas-suriname/assets/MINING/COLLECTION-1/classification-ft';

var vMGRegions   = commonPath + '/assets/Archive_Suriname/classification_regions_suriname';

var mosaicsPath = 'projects/nexgenmap/MapBiomas2/LANDSAT/PANAMAZON/mosaics-2';
var mosaicsPath2 = 'projects/mapbiomas-raisg/MOSAICOS/mosaics-2';





var image        = ee.Image(assetPath + '/' + assetName);

var mosaics      = ee.ImageCollection(mosaicsPath).merge(ee.ImageCollection(mosaicsPath2));

var mgRegions    = ee.FeatureCollection(vMGRegions);


      


Map.setOptions('SATELLITE');

ee.List(years)
  .evaluate(function(years) {
    
    var bands = years.map(function(year) {
      return 'classification_' + year;
    });
    
    var fixed = years.map(function(year) {
      
      var selected = image.select('classification_' + year);
      
      var remaped = remapWithPolygons(selected, remapZones)
        .rename('classification_' + year);
      
      
      if(preview.indexOf(year) >= 0) {
        var yearMosaic  = getMosaic(mosaics, mosaicRegions, year);
        
        Map.addLayer(
          yearMosaic,
          { min: 200, max: 5000, bands: 'swir1_median,nir_median,red_median' },
          'MOSAICO ' + year,
          false
        );
        
        Map.addLayer(
          selected.eq(30).selfMask(),
          {
            min: 27,
            max: 30,
            palette: 'gold'
          },
          'ORIGINAL ' + year,
          false
        );
        Map.addLayer(
          remaped.eq(30).selfMask(),
          {
            min: 27,
            max: 30,
            palette: 'gold'
          },
          'REMAPED ' + year
        );
        
      }
      
      return remaped;
      
    });
    
    
    var finalImage = ee.ImageCollection(fixed).toBands().rename(bands)
                    .set('code_region', code_region)
                    .set('pais', pais)
                    .set('version', version_output)
                    .set('descripcion', 'exclusion_areas')
                    .set('paso', 'S07-1');
    
    
    Export.image.toAsset({
      image: finalImage,
      description: outputName, 
      assetId: assetPath + '/' + outputName,
      pyramidingPolicy: {
        '.default': 'mode'
      },
      region: image.geometry().bounds(),
      scale: 30,
      crs: 'EPSG:4326',
      maxPixels: 1e13
    });
    
  });






/**
 * 
 * remapWithPolygons
 * @image
 * @polygons
 */
function remapWithPolygons(image, polygons) {
  
  if(polygons.length > 0) {
    polygons.forEach(function( polygon ) {
      
      var excluded = polygon.map(function(layer){
        var area = image.clip( layer );

        var from = ee.String(layer.get('from'))
          .split(',')
          .map(function(item) { return ee.Number.parse(item) });
          
        var to = ee.String(layer.get('to'))
          .split(',')
          .map(function(item){ return ee.Number.parse(item) });
        
        return area.remap(from, to);
      });
        
      excluded = ee.ImageCollection(excluded).mosaic();
      image = excluded.unmask(image).rename('reference');
      image = image.mask(image.neq(0));
    });
  }
  else image = image;

  return image;
  
}





/**
 *
 * getMosaics
 */
function getMosaic(mosaics, regionIds, year) {
  
  return mosaics
    .filter(
      ee.Filter.and(
        ee.Filter.eq('year', year),
        ee.Filter.inList('region_code', regionIds)
      )
    )
    .mosaic();
  
}
```
