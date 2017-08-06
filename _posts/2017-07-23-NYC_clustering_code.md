---
title: "k-means clustering for NYC"
layout: post
categories: [data]
tags: [R, k-means clustering]
---





This is the underlying code R code for my [earlier post ]({{site.baseurl}}{% post_url 2017-04-08-KCities %}) analyzing NYC's neighborhoods using principal-component-analysis and k-means cluster on socioeconomic data.

### Data Prep in R


{% highlight r %}
library(feather) # you'll need feather installed in both R and python
library(dplyr)

ACS <- read_feather("ACS.feather")s
ACS <- ACS[, !duplicated(colnames(ACS))]

ACS <- rename(ACS, nta_name = GeogName , nta_code = GeoID)
{% endhighlight %}



{% highlight text %}
## Error in rename(ACS, nta_name = GeogName, nta_code = GeoID): unused arguments (nta_name = GeogName, nta_code = GeoID)
{% endhighlight %}



{% highlight r %}
ACS$MdHHIncP <- ACS$MdHHIncE / 55000  # have to construct by hand

labelnames <- c("nta_code","nta_name","Borough")
labels <- ACS[labelnames]
{% endhighlight %}



{% highlight text %}
## Error: Unknown columns 'nta_code', 'nta_name'
{% endhighlight %}



{% highlight r %}
substrRight <- function(x, n){
  substr(x, nchar(x)-n+1, nchar(x))
}

keeps <- names(ACS)[substrRight(names(ACS),1) == "P"] # drop non-percent variables
keeps <- keeps[keeps != "MnHHIncP"] #  , mean income
keeps <- keeps[keeps != "HHP"] #  , check variable sums to 100



km_data   <- ACS[keeps]
too_many_NANs <- colnames(km_data)[colSums(is.na(km_data)) > 50] # filter out variables that are too sparse

print(too_many_NANs)
{% endhighlight %}



{% highlight text %}
## [1] "AgFFHMP"    "UpdFmWrkrP"
{% endhighlight %}



{% highlight r %}
km_data <- km_data[!is.element(colnames(km_data), too_many_NANs)]
km_labels <- labels[complete.cases(km_data),]
km_data   <- km_data[complete.cases(km_data),]

rownames(km_data) <- km_labels$nta_name
{% endhighlight %}

### Run Principle Components Analyis on all the ACS data


{% highlight r %}
library(pcaPP)

pr_out <- PCAproj(km_data, scale = sd, k = 5  , scores = TRUE)

rownames(pr_out$scores) <- rownames(km_data)
{% endhighlight %}

### How much of variance is explained by each factor?


{% highlight r %}
pr_var <- pr_out$sdev ^ 2
pve <- pr_var / sum(pr_var)

plot(cumsum(pve), ylim = c(0,1), type='b')
{% endhighlight %}

![center]({{site.baseurl}}/images/2017-07-23-NYC_clustering_share/unnamed-chunk-3-1.png)

### K-means clustering on the PCA components

I set k = 3  after some trial and error where I tried to minimize the overlap between clusters inside the PCA biplot as seen below.


{% highlight r %}
km_out <- kmeans(pr_out$scores, centers = 3, nstart = 3)
results  <- data.frame(dplyr::combine(list(km_out$cluster),km_labels ))
colnames(results) <- c("cluster_code",colnames(km_labels))
names(results)[names(results) == 'nta_code'] <- 'ntacode'  # for matching to carto
{% endhighlight %}

### Plotting clusters against first two PCA dimensions



{% highlight r %}
library(factoextra)

cluster_colors <- c("#1b9e77", "#d95f02", "#7570b3")

plot(fviz_pca_ind(pr_out , label="var", addEllipses = TRUE,
            habillage=as.factor(results$cluster_code)) + ggtitle("")  +
            scale_color_manual(values= cluster_colors) +
            theme(text = element_text(size = 10),
                panel.background = element_blank(),
                panel.grid.major = element_blank(),
                panel.grid.minor = element_blank(),
                axis.line = element_line(colour = "black"),
                legend.position=c(.9,.95)))
{% endhighlight %}

![center]({{site.baseurl}}/images/2017-07-23-NYC_clustering_share/unnamed-chunk-5-1.png)


### Plotting projection of ACS variables onto first two PCA dimensions.

Income, immigration status, and private/public sector worker status seem particularly relevant.


{% highlight r %}
fviz_pca_var(pr_out, alpha.var = "contrib") + theme_minimal()
{% endhighlight %}

![center]({{site.baseurl}}/images/2017-07-23-NYC_clustering_share/unnamed-chunk-6-1.png)

Plotting neighborhoods onto the PCA dimensions.

{% highlight r %}
fviz_pca_ind(pr_out, labelsize = 2) + theme_minimal()
{% endhighlight %}

![center]({{site.baseurl}}/images/2017-07-23-NYC_clustering_share/unnamed-chunk-7-1.png)

### Map the clusters

First I need to pull neighborhood polygons off of my CartoDB account.


{% highlight r %}
library(rgdal)
library(CartoDB)
library(maptools)
invisible(cartodb("srimmele",api.key = Sys.getenv("CARTO_KEY")));

# CartoDB the_geom columns are always the following proj string
crs <- "+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs"


nta_URL <- cartodb.collection("table_2016_09_nta_socrata", columns=c("cartodb_id","the_geom","ntacode"), omitNull=TRUE,method="GeoJSON", urlOnly=TRUE)

nta.poly <- rgdal::readOGR(nta_URL,p4s=crs,layer = 'OGRGeoJSON', verbose = FALSE)
nta.poly@data$plot_order <- as.integer(rownames(nta.poly@data))  # to preserve the plotting order of the polygons

nta.poly@data  <- merge(nta.poly@data,results, by = "ntacode", all.x = TRUE)
nta.poly@data <- dplyr::arrange(nta.poly@data,plot_order)

write.csv(results,'NYC_clusters.csv')
{% endhighlight %}





{% highlight r %}
### gpclib was finicky and I needed to use some plyr function to get it to work properly
### library(gpclib) >  gpclibPermit() , only need to do once
require("ggplot2")
require("plyr")
require("gpclib")
gpclibPermit()
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
nta.poly@data$id <- nta.poly@data$plot_order
nta.poly.points <- fortify(nta.poly, region="id")
nta.poly.df <- join(nta.poly.points, nta.poly@data, by="id")
nta.poly.df <- dplyr::arrange(nta.poly.df,order)
{% endhighlight %}


{% highlight r %}
ggplot(nta.poly.df, aes(x = long , y = lat,group=group, fill = cluster_code)) + geom_polygon() + coord_map() + theme(legend.position="none") + scale_fill_gradientn(colors = cluster_colors)
{% endhighlight %}

![center]({{site.baseurl}}/images/2017-07-23-NYC_clustering_share/unnamed-chunk-10-1.png)

According to the k-means clustering algorithm, we see three distinct areas (3 by construction), and they map pretty well against my prior expectations of neighborhood character. I would describe them as follows:

  * Lower Manhattan , Downtown Brooklyn , Astoria. Maybe a wealthy/ rapidly gentrifying core. (orange)
  * Upper Manhattan and Lower-East-Side , Central Brooklyn , Central Queens , The Bronx. Maybe the high-immigrant / lower-income areas. Centers of ongoing gentrification. (green)
  * Staten Island, Far Brooklyn and Queens, Riverdale and the far Bronx. Predominantly white and/or middle class areas. (purple)



### 3D Scatter plot of three variables along original (non-PCA) dimensions

Another way to visualize these socioeconomic differences is with a 3d scatter plot that makes the clusters more apparent. I chose to plot  immigrant status, income, and industry of residents for each neighborhood. I picked these variables based on their importance to the PCA analysis.



{% highlight r %}
results$cluster_code <- as.factor(results$cluster_code)

plotly::plot_ly(type = 'scatter3d',  km_data, x = ~PrvWSWrkrP, y= ~Fb1P, z = ~MdHHIncP , color = results$cluster_code, colors = cluster_colors, name = "blah", hoverinfo = 'text', text = ~paste('NTA: ', rownames(km_data),
                                              '</br> % Private Sector Workers: ' , PrvWSWrkrP,
                                              '</br> % Foreign-born: ' , Fb1P,
                                              '</br> Income (% City Median): ', round(MdHHIncP, 2)))
{% endhighlight %}
