# suriname-coverage

## Overview
Scripts of all the themes used for Suriname. Look below for a step by step guide.





---
## Guideline Structures

### General
```
├── step-1-samples/                           # Create stable and training areas
│   ├── 02-0-stablePixels                     # Stable pixel identification
│   ├── 03-1-training-areas                   # Class area sampling
│   └── 03-2-training-points                  # Training point generation
├── step-2-classifications/                   # Random Forest classification
│   ├── 04-0-preliminary-classification       # Classification process
├── step-3-filters/                           # Post-classification filters
│   ├──04-1-gapfill                           # Gap filling
│   ├── 05-incidence-filter                   # Incidence filter
│   ├── 06-temporalFilter                     # Temporal filter
│   ├── 07-frequencyFilter                    # Frequency filter
│   ├── 08-noFalseRegrowth                    # Post processing filter 
│   └── 09-spatialFilter                      # Spatial filter
```
### Urban
```
├── step-1-samples/                           
│   ├── 1-urban_reference_area                 # Gather and accumulated reffrences
│   ├── 2-training_samples                     # Class area sampling
├── step-2-classifications/                    # Random Forest classification
│   ├── 3-classification                       # Classification process
├── step-3-filters/                            # Post-classification filters
│   ├──4-1-gap_fill.md                         # Gap filling
│   ├── 4-2-temporal_filter                    # Temporal filter
│   ├── -3-frequency_filter_lastyear_24.       # Frequency filter Urban
│   ├── 4-4-frequency_filter_firstyear_27.     # Frequency filter Non observed area
│   └── 4-5-spatial_filter                     # Spatial filter
```


### Mining
```
├── step-1-samples/                           
│   ├── 1-mining_reference_area                # Gather and accumulate reffrences
│   ├── 2-training_samples                     # Class area sampling
├── step-2-classifications/                    # Random Forest classification
│   ├── 3-classification.md                    # Classification process
├── step-3-filters/                            # Post-classification filters
│   ├──4-gap_fill                              # Gap filling
│   ├── 5-spatial_filter                       # Spatial filter
│   ├── 6-temporal_filter                      # Temporal filter
│   └── 7-exclusion-area                       # Draw Exclusions 
```
