---
layout: post
title:  "Grid of histograms in R using ggplot2"
date:   2014-09-24 15:55:21
comments: true
---

In this gist I will explain a simple way to generate a grid of histograms in ```R``` using ```ggplot2``` library. I will use the ```SegmentationOriginal``` dataset from the ```AppliedPredictiveModeling``` package.

- install dependencies and load libraries.
{% highlight r %}
install.packages("AppliedPredictiveModeling")
install.packages("ggplot2")
install.packages("e1071")
data(segmentationOriginal)
library(AppliedPredictiveModeling)
library(e1071)
library(ggplot2)
{% endhighlight %}

- Suppose now that we want to want to make a grid of four columns of the dataset, namely ```SkewIntenCh1,SkewIntenCh3,AngleCh1,AreaCh1```
{% highlight r %}
# get a portion of the data.
segTrainData = subset(segmentationOriginal, Case=="Train")
columns = c("SkewIntenCh1","SkewIntenCh3","AngleCh1","AreaCh1")
df = data.frame()
for(i in seq(from=1, to=4)){
	column = segTrainData[columns[i]][,1]
	dft = data.frame(column, array(i, length(column)))
	df = rbind(df, dft)
}
colnames(df) <- c("vals", "indicator")
q <- qplot(vals, data=df, geom="histogram")
p <- q + facet_wrap(~indicator, scales="free_x")
p
{% endhighlight %}

The result will be like that:
![Result]({{ site.url }}/images/multiple_hist_R.png)

- Suppose now that we want to change the headers to reflect certain value, for example the **skewness** of each variable, we can use a labeller and change the ``indicator`` variable to reflect that value
{% highlight r %}
paste_labeller <- function(variable, value){
  paste(variable, value, sep=";")
}
df = data.frame()
for(i in seq(from=1, to=4)){
  column = segTrainData[columns[i]][,1]
  skew_val = skewness(column)
  dft = data.frame(column, array(paste_labeller("skewness", skew_val), length(column)))
  df = rbind(df, dft)
}
colnames(df) <- c("vals", "indicator")
q <- qplot(vals, data=df, geom="histogram")
p <- q + facet_wrap(~indicator, scales="free_x")
p
{% endhighlight %}
The result will be like that:
![Result]({{ site.url }}/images/multiple_hist_R-2.png)
