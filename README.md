
<!-- README.md is generated from README.Rmd. Please edit that file -->

# IdentifyingConservationGaps

<!-- badges: start -->
<!-- badges: end -->

This repository documents the data and R code used to reproduce the
terrestrial analyses and **Fig. 1** in the manuscript *Identifying
Conservation Gaps: A Framework for Evaluating National Contributions to
the 30x30 Target*.

Fig. 1 shows how Denmark’s terrestrial areas contribute to the **30%
land target**, distinguishing between:

- areas that fully qualify as contributing protected areas,
- areas requiring individual assessment,
- areas with insufficient legal protection,
- areas compromised by infrastructure, forestry, or agriculture, and
- unprotected land.

This README focuses on the spatial data processing steps used to derive
the terrestrial map and summary statistics for Fig. 1.

## Software and packages

All spatial calculations were done in R using the
[`terra`](https://cran.r-project.org/package=terra) package.
Visualisations were generated with
[`ggplot2`](https://cran.r-project.org/package=ggplot2) and
[`tidyterra`](https://cran.r-project.org/package=tidyterra). The
terrestrial outline of Denmark was retrieved using the
[`geodata`](https://cran.r-project.org/package=geodata) package.

``` r
library(geodata)
library(ggplot2)
library(terra)
library(tidyterra)
```

## Spatial template and national boundary

All rasters used in the analysis are aligned to a common template
defining the extent, resolution and coordinate reference system.

``` r
Template <- terra::rast("Data/Template.tif")

DK <- geodata::gadm(country = "Denmark", level = 0, path = getwd(), version = "4.0") |> 
  terra::project(terra::crs(Template))
```

We then calculate the terrestrial area of Denmark in square kilometres:

``` r
Area_DK <- terra::expanse(DK, unit = "km")
```

Which is 43,144.85 square kilometers.

## Build up

In many of the layers we need to correct using buildup, which was
extracted from the landcover layer in
[basemap](envs.au.dk/en/research-areas/society-environment-and-resources/land-use-and-gis/basemap)

Here, `BuildUp` is a raster where 0 = built-up (infrastructure, urban
areas), 1 = other land. This raster is prepared in the data
pre-processing steps (not shown here).

``` r
BuildUp <- terra::rast("Data/Rast_BuildUp_Croped.tif")
```

## Protected areas contributing to the 30% target

In the manuscript, areas that **fully contribute** to the 30% land
target are those protection schemes that meet all criteria C1–C5 for
protected areas (Tables 1 and 5 in the manuscript). On land, these are:

- **Approved Nature National Parks**
- **Areas owned by private nature foundations**
- **State-owned unmanaged / untouched forests**

Below we show how these layers are read, harmonised and combined into a
single raster representing all areas that contribute to the 30%
terrestrial target.

To track overlaps between protection schemes, each scheme is encoded as
a binary layer with a **distinct base value** (1, 10, 100). When these
layers are summed, each combination of schemes yields a unique value
(e.g. 1 = Nature National Park only; 11 = Nature National Park + private
foundation; 111 = all three schemes).

### Approved Nature National Parks

We read the raster of approved Nature National Parks and convert it to a
binary layer where 0 = not an approved Nature National Park, 1 =
approved Nature National Park.

``` r
NNP_approved <- as.numeric(terra::rast("Data/Rast_Nature_National_Parks_Croped.tif")) + 1
NNP_approved <- terra::ifel(is.na(NNP_approved), 0, NNP_approved)
```

### Private nature foundations

Similarly, we read the raster of areas owned by private nature
foundations (`Fondsejede`) and encode it as

0 = not foundation-owned, 10 = foundation-owned.

``` r
Fondsejede <- as.numeric(terra::rast("Data/Rast_Fondsejede_Croped.tif")) + 10
Fondsejede <- terra::ifel(is.na(Fondsejede), 0, Fondsejede)
```

### Unmanaged / untouched forests

We then read the raster of state-owned unmanaged / untouched forests
(`UroertSkov_NST`) and encode it as

0 = not unmanaged / untouched forest, 100 = unmanaged / untouched
forest.

``` r
UroertSkov_NST <- as.numeric(terra::rast("Data/Rast_Urort_Skov_Croped.tif")) + 99
UroertSkov_NST <- terra::ifel(is.na(UroertSkov_NST), 0, UroertSkov_NST)
```

### Generate composite layer of contributing protected areas

We now add the three layers to obtain a **composite raster** `PA30`,
where the value of each pixel uniquely identifies which combination of
schemes covers that pixel.

``` r
PA30 <- NNP_approved + Fondsejede + UroertSkov_NST

cls <- data.frame(
  ID    = c(1,   10,          11,             100,            101,            110,                  111),
  class = c("NNP_approved",
            "Fondsejede",
            "NNP_Fondsejede",
            "UroertSkov_NST",
            "NNP_Uroert",
            "Fondsejede_Uroert",
            "NNP_Fondsejede_Uroert")
)

PA30 <- terra::ifel(PA30 == 0, NA, PA30)
```

We then remove built-up areas and mask the result to the terrestrial
area of Denmark.

``` r
PA30 <- terra::ifel(BuildUp == 0, NA, PA30)

levels(PA30) <- cls

PA30 <- terra::mask(PA30, DK)
```

Finally, we save the composite layer as a Cloud Optimised GeoTIFF:

``` r
SpeciesPoolR::write_cog(PA30, "FinalLayers/PA30.tif")
```

This layer corresponds to the **turquoise category** (“Protected areas
that fully meet the criteria”) in Fig. 1 of the manuscript.

## Production areas (areas compromising biodiversity)

In Fig. 1, areas where biodiversity is directly compromised by human
land uses (e.g. intensive agriculture, production forestry,
infrastructure) are shown in **orange** (“Areas compromised by
infrastructure, forestry, or agriculture”).

These areas are identified by combining:

- agricultural land (fields in rotation and permanent grasslands not
  covered by Article 3 protection),
- production forest (tree-covered areas not designated as unmanaged
  forest or Article 3 areas), and
- infrastructure (roads, buildings and other built-up areas).

In the analysis pipeline, these inputs are assembled into a raster layer
where pixels used primarily for these land uses are flagged as
**compromised** and subsequently excluded from contributing to the 30%
target. The resulting raster is then used to derive the orange category
in Fig. 1.

### Reading the layers in

We have a layer (subclasses) that has among its categories fields in
rotation (INT_AGG), grasslands not covered by Article 3 protection
(“PGR_out_of_P3”) and production forest (Drevet Skov), we fuse this
classes into one layer

``` r
Subclasses<- rast("Data/Subclasses.tif")

ProductionAreas <- Template

ProductionAreas <- terra::ifel(Subclasses %in% c("Drevet Skov", "INT_AGG", "PGR_out_of_P3"), 1, 0)
```

We then add the Buildup that we read at the beggining of this document.

``` r
ProductionAreas <- terra::ifel(!is.na(Subclasses) & BuildUp == 0, 1, ProductionAreas)

ProductionAreas <- terra::mask(ProductionAreas, DK)

cls <- data.frame(id=0:1, Protection=c("other",stringr::str_wrap("Production areas", 10)))
levels(ProductionAreas) <- cls
```

``` r
SpeciesPoolR::write_cog(ProductionAreas, "FinalLayers/ProductionAreas.tif")
```

## Insuffient legal protection

## Requires individual assesment
