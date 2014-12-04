---
title       : Ecstasy
subtitle    : 
author      : Ryan Quan & Frank Chen
job         : Columbia University
framework   : io2012        # {io2012, html5slides, shower, dzslides, ...}
highlighter : highlight.js  # {highlight.js, prettify, highlight}
hitheme     : tomorrow      # 
widgets     : [mathjax]            # {mathjax, quiz, bootstrap}
mode        : selfcontained # {standalone, draft}
knit        : slidify::knit2slides
--- 



<!-- Limit image width and height -->
<style type='text/css'>
img {
    max-height: 560px;
    max-width: 964px;
}
</style>

<!-- Center image on slide -->
<script src="http://ajax.aspnetcdn.com/ajax/jQuery/jquery-1.7.min.js"></script>
<script type='text/javascript'>
$(function() {
    $("p:has(img)").addClass('centered');
});
</script>

---

## Our Question

<q>How has the composition of pills sold as ecstasy changed over time?</q>

---

## Outline

* Background
* Data Source
* Getting and Cleaning Data
* Sketches and Visualizations
* TODO

--- .segue .dark

## Background

---

## What is Ecstasy?

> * Any pill represented as MDMA on the street
> * Notoriously unreliable in content and purity
> * Can come in tablet, powder, or crystalline form

---

## Pill Form

![](assets/img/ecstasy01.jpg)

---

## Powder Form

![](assets/img/ecstasy02.jpg)

---

## Common Effects

> * Positive
	* feelings of comfort, empathy, and connection to others
	* decreased fear, anxiety, and insecurities
	* increased awareness and appreciation of music
> * Negative
	* jaw clenching, teeth grinding, cheek chewing
	* increase in body temperature, dehydration
	* hyponatremia (drinking too much water)
	* post-trip crash

---

## In The News

> * ![](assets/img/headline02.png)
> * ![](assets/img/headline03.png)
> * ![](assets/img/headline04.png)

--- 

## Word on the Street

<q>The mainstream popularity of music festivals and raves has led to an upsurge of "impure" ecstasy pills as demand of MDMA exceeds supply.</q>

--- .segue .dark

## Data Source

---

## Dead End?

> * Data on Schedule I substances is difficult to obtain
> * Very few human studies available
> * Collecting new data was not a reasonable option

---

## EcstasyData.org

> * Independent lab pill testing program
> * Run by [Erowid Center](https://www.erowid.org/) and sponsored by [Dancesafe](http://www.dancesafe.org/)
> * Analyze chemical composition of user-submitted and confiscated pills
> * Publish results [online](http://www.ecstasydata.org/results.php) as a means of harm reduction

---

## Caveats

> * Sample bias
> * Chemical composition presented as ratio, not mass percent
> * Not concerned with therapeutic dose of individual substances

--- .segue .dark

## Getting and Cleaning Data

---

## Data Scraping

* Do everything the dumb way for one year…
	* Played around with the query URL to test the parameters
	

```r
url <- sprintf("http://www.ecstasydata.org/results_xml.php?sold_as_ecstasy=yes&
Y1=%s&Y2=%s&max=500", year, year)
```

* Obtained raw XML for 2013-2014 using the `XML` package


```r
raw <- xmlToList(xmlParse(year.params.url))
```

---

## Data Wrangling

* ID and active components
	* Looped through active components using `lapply` and put them in a dataframe
* Active components (list element) were trapped in the columnar format "X:Y"
	* Use `colsplit` to separate composition:proportion


```r
id.actives <- id.actives[complete.cases(id.actives), ]
id.actives[, 2:3] <- colsplit(id.actives$composition, ":", 
c("composition", "proportion"))
```

---

## Data Wrangling

* ID and attributes (name, location, mass, year)
	* Additional attributes by ID were looped via `lapply` and put in a dataframe
	* `NULL` locations broke `lapply` loop, fixed with an `if(is.null)` statement
	

```r
if (is.null(raw.data$tablet$location)) {
		raw.data$tablet$location <- "NA"
}
```

---

## Data Wrangling

* Combine actives and attributes by ID
* melt / cast dataframe using reshape2 package

---

## Complete Script (All Years)

* Scrape data for 1999-2014 via functionalization of all previous operations
	* Script would break due to unescaped ampersands (`EntityRef Error`)
* Used `gsub` to substitute entity reference which allowed the script to run


```r
doc <- getURL(url)
doc <- gsub("&", "&amp;", doc)
```

---
	
## Cleaning

* Combined all yearly datasets into one large, melted dataset
* Removed trailing whitespaces, coerced all non-numeric chars to `NA`


```r
ecstasy$composition <- gsub(" $","", ecstasy$composition, perl = TRUE)
ecstasy$location <- gsub("\n", "", ecstasy$location)

ecstasy$proportion <- as.numeric(ecstasy$proportion)
```

---

## Cleaning

* Applied bin categorical weights based on normalized composition
	1. Pure MDMA $(=1)$
	2. More MDMA $(\ge 0.5)$
	3. Less MDMA $(< 0.5)$
	4. No MDMA $(=0)$
	5. Unknown

---

## Cleaning
	

```r
if (!("MDMA" %in% pill$composition)) {
        mdma[i] <- 4 # No MDMA
}
mdma.prop <- pill[pill$composition == "MDMA", "proportion"] / sum(pill$proportion)
if (is.nan(mdma.prop) | is.na(mdma.prop)) {
    mdma[i] <- 5
	} else if (mdma.prop > 0.5 & mdma.prop < 1) {
    	mdma[i] <- 2
	} else if (mdma.prop <= 0.5 & mdma.prop > 0) {
    	mdma[i] <- 3
	} else if (mdma.prop == 1) {
    	mdma[i] <- 1
	} else {
    	mdma[i] <- 5
	}
}
```

--- .segue .dark

## Sketches and Visualizations

---

## Sketches

![sketch](assets/img/ecstasy_sketch.jpeg)

---

## Sketches

* Simple interactive dotplot
* 3-D visualization of deviations...how to do 4-D? Nah.
* Mini circles in a big circle? Sort of like a pie chart…
* Stacked stream, area, or bar?
* Dunkin’ donuts chart

--- 

## Visualizations

`ggplot2` - Stacked Area Plot


```r
ggplot(mdmaPlot, aes(year, proportion)) +
    geom_area(aes(col = mdma, fill = mdma), position = "stack") +
    scale_fill_brewer(type = "div", palette = 1) +
    theme_classic()
```

---

## MDMA Content in Ecstasy Pills, 1999-2014

![plot of chunk ggplot](assets/fig/ggplot-1.png) 

---

## Visualizations

`rCharts` - NVD3 Stacked Area Chart


```r
mdmaPlotSubset <- subset(as.data.frame(mdmaPlot), !(mdma == "Unknown"))
n1 <- nPlot(proportion ~ year, group = "mdma", 
            data = mdmaPlotSubset, type = "stackedAreaChart")
n1
```

---

## MDMA Content in Ecstasy Pills, 1999-2014

<iframe src=' assets/fig/nvd3plot-1.html ' scrolling='no' frameBorder='0' seamless class='rChart nvd3 ' id=iframe- chart12819322c8263 ></iframe> <style>iframe.rChart{ width: 100%; height: 400px;}</style>

---

## Visualizations

`D3.js` - [Stacked Area Plot with Hover-Overs](http://plnkr.co/qLBXCBID4QwiUjNvgeb7)

---

## Visualizations

`D3.js` - [Slider Dot Plot](http://embed.plnkr.co/uQ9TgDVX6tce7MUFUOnm/preview)

--- .segue .dark

## TODO

---

## Modifications

> * Find an appropriate color palette for this dataset
> * Highlight stacked area regons on hover-over to improve focus 
> * Dot plot hover-overs to encourage user to learn more about a drug

---

## Additional Visualizations

> * Word cloud of common colors and branding for ecstasy pills
> * Chloropleth map to display hotspots for ecstasy sales
> * Tracking the purity of a single "brand" over time 

---

## Lessons Learned

> * Paired programming is very efficient and fun
> * A sketch of the end-product is good for motivation
> * Ask for the owner's permission before scraping data
> * Offer to make visualizations for small organizations
> * Escape those damn ampersands - "&amp;"!

--- .segue .dark

## Thank You!

---
