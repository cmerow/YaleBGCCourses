---
title: "Introduction to Species Distribution Modeling"
author: "[Cory Merow](cmerow.github.io)"
---



<!-- <div> -->
<!-- <iframe src="05_presentation/05_Spatial.html" width="100%" height="700px"> </iframe> -->
<!-- </div> -->

[<i class="fa fa-file-code-o fa-3x" aria-hidden="true"></i> The R Script associated with this page is available here](101SDMs.R).  Download this file and open it (or copy-paste into a new script) with RStudio so you can follow along.  

# Setup


```r
library(spocc)
library(raster)
```

```
## Loading required package: sp
```

```r
library(sp)
library(rgdal)
```

```
## rgdal: version: 1.2-16, (SVN revision 701)
##  Geospatial Data Abstraction Library extensions to R successfully loaded
##  Loaded GDAL runtime: GDAL 2.1.3, released 2017/20/01
##  Path to GDAL shared files: /Library/Frameworks/R.framework/Versions/3.4/Resources/library/rgdal/gdal
##  GDAL binary built with GEOS: FALSE 
##  Loaded PROJ.4 runtime: Rel. 4.9.3, 15 August 2016, [PJ_VERSION: 493]
##  Path to PROJ.4 shared files: /Library/Frameworks/R.framework/Versions/3.4/Resources/library/rgdal/proj
##  Linking to sp version: 1.2-5
```

```r
library(ROCR)
```

```
## Loading required package: gplots
```

```
## 
## Attaching package: 'gplots'
```

```
## The following object is masked from 'package:stats':
## 
##     lowess
```

```r
library(corrplot)
```

```
## corrplot 0.84 loaded
```

```r
library(maxnet)
library(spThin)
```

```
## Loading required package: spam
```

```
## Loading required package: grid
```

```
## Spam version 1.4-0 (2016-08-29) is loaded.
## Type 'help( Spam)' or 'demo( spam)' for a short introduction 
## and overview of this package.
## Help for individual functions is also obtained by adding the
## suffix '.spam' to the function name, e.g. 'help( chol.spam)'.
```

```
## 
## Attaching package: 'spam'
```

```
## The following objects are masked from 'package:base':
## 
##     backsolve, forwardsolve
```

```
## Loading required package: fields
```

```
## Loading required package: maps
```

```
## Loading required package: knitr
```


# The worst SDM ever

The goal of this is to use the simplest possible set of operations to build an SDM. There are many packages that will perform much more refined versions of these steps, at the expense that decisions are made behind the scenes, or may be obscure to the user, especially if s/he is not big on reading help files. So before getting into the fancy tools, let's see what the bare minimum looks like.

> This is not the simplest possible code, because it requires some familiarity with the internal components of different spatial objects. The tradeoff is that none of the key operations are performed behind the scenes by specialized SDM functions. I realize this is not always pretty, but I hope for that reason it can demonstrate some coding gynmastics for beginners.


## Get presence data

```r
# get presence data
# pres=occ('Alliaria petiolata',from='gbif',limit=5000) # this can be slow
  # so just read in the result of me running this earlier
pres=read.csv('https://cmerow.github.io/YaleBGCCourses/101_assets/AP_gbif.csv')[,c('longitude','latitude')]
pres=pres[complete.cases(pres),] # toss records without coords
```

## Get environmental data

> Decision: Worldclim data describes the environmentl well in this region. The 'bioclim' variables are biologically relevant summaries of climate.


```r
# get climate data
  # the raster package has convenience function build in for worldclim
clim=getData('worldclim', var='bio', res=10)
```

The Bioclim variables in `clim.us` are:

<small>

Variable      Description
-    -
BIO1          Annual Mean Temperature
BIO2          Mean Diurnal Range (Mean of monthly (max temp – min temp))
BIO3          Isothermality (BIO2/BIO7) (* 100)
BIO4          Temperature Seasonality (standard deviation *100)
BIO5          Max Temperature of Warmest Month
BIO6          Min Temperature of Coldest Month
BIO7          Temperature Annual Range (BIO5-BIO6)
BIO8          Mean Temperature of Wettest Quarter
BIO9          Mean Temperature of Driest Quarter
BIO10         Mean Temperature of Warmest Quarter
BIO11         Mean Temperature of Coldest Quarter
BIO12         Annual Precipitation
BIO13         Precipitation of Wettest Month
BIO14         Precipitation of Driest Month
BIO15         Precipitation Seasonality (Coefficient of Variation)
BIO16         Precipitation of Wettest Quarter
BIO17         Precipitation of Driest Quarter
BIO18         Precipitation of Warmest Quarter
BIO19         Precipitation of Coldest Quarter

</small>


##  Choose domain

> Decision: We are only asking about invasion in New England, so we constrain the domain to a bounding box around New England


```r
# choose domain (just the Eastern US)
clim.us=raster::crop(clim,c(-76,-65,40,50)) # trim to a smaller region
plot(clim.us[[1]]) # plot just the 1st variable to see domain
```

![](101SDMs_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

##  Prep data

> Decision: Correlated predictors can make it difficult to interpret model coefficients or response curves. So we'll remove the most correlated predictores


```r
# check for correlated predictors
cors=cor(values(clim.us),use='complete.obs') # evaluate correlations
corrplot(cors,order = "AOE", addCoef.col = "grey",number.cex=.6) # plot correlations
```

![](101SDMs_files/figure-html/unnamed-chunk-6-1.png)<!-- -->


```r
clim=clim[[c('bio1','bio2','bio14')]] # keep just reasonably uncorrelated ones
clim.us=clim.us[[c('bio1','bio2','bio14')]] # keep just reasonably uncorrelated ones
cors=cor(values(clim.us),use='complete.obs') # evaluate correlations
corrplot(cors,order = "AOE", addCoef.col = "grey",number.cex=.6)# plot correlations
```

![](101SDMs_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

Ok, tolerable.



```r
# scale each predictor to mean=0, variance=1
clim.means=apply(values(clim.us),2,mean,na.rm=T) # means
clim.sds=apply(values(clim.us),2,sd,na.rm=T) # standard devations
name=names(clim.us)
values(clim.us)=sapply(1:nlayers(clim.us),function(x) (values(clim.us)[,x]-clim.means[x])/clim.sds[x]) 
# z-scores
names(clim.us)=name

# get environment at pres points
coordinates(pres)=c('longitude','latitude') # set coords to allow extraction (next line)
pres.data=data.frame(raster::extract(clim.us,pres)) # extract data at pres locations
coordinates(pres.data)=coordinates(pres) # make sure the data have coords associated
pres.data=pres.data[complete.cases(pres.data@data),] # toss points without env data
```


```r
plot(clim.us) # view 
```

![](101SDMs_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

##  Sample background

> Decision: The species is equally likely to be anywhere on the landscapes, so we'll compare presences to a random sample of background points.


```r
	## save the data table
# sample background (to compare against presences)
all.background=which(complete.cases(values(clim.us))) # find cells on land
bg.index=sample(all.background,min(length(all.background),10000)) # take random sample of land
bg.data=data.frame(values(clim.us)[bg.index,]) # get the env at these cells
coordinates(bg.data)=coordinates(clim.us)[bg.index,] # define spatial object
```

> Decision: Linear and quadratic terms are sufficient to describe the species' response to the environment. 


```r
# prep data for use in glm()
all.data=rbind(data.frame(pres=1,pres.data@data),data.frame(pres=0,bg.data@data))

# specify formula (quickly to avoid writing out every name)
(form=paste('pres/weight~', # lhs of eqn.
            paste(names(all.data)[-1], collapse = " + "),'+', # linear terms
            paste("I(", names(all.data)[-1], "^2)", sep = "", collapse = " + "))) # qudratic terms
```

```
## [1] "pres/weight~ bio1 + bio2 + bio14 + I(bio1^2) + I(bio2^2) + I(bio14^2)"
```


## Statistical model


```r
all.data$weight = all.data$pres + (1 - all.data$pres) * 10000 # these allow you to fit a Point Process
mod.worst=glm(form,data=all.data,family=poisson(link='log'),weights=weight) # fit the model
#mod.worst=maxnet(all.data$pres,all.data[-1])
summary(mod.worst) # show coefficients
```

```
## 
## Call:
## glm(formula = form, family = poisson(link = "log"), data = all.data, 
##     weights = weight)
## 
## Deviance Residuals: 
##     Min       1Q   Median       3Q      Max  
## -3.7578  -0.2080  -0.0770  -0.0355   5.1352  
## 
## Coefficients:
##              Estimate Std. Error z value Pr(>|z|)    
## (Intercept) -15.20385    0.28785 -52.819  < 2e-16 ***
## bio1          2.33396    0.27664   8.437  < 2e-16 ***
## bio2          0.92237    0.07316  12.607  < 2e-16 ***
## bio14        -0.86362    0.07654 -11.283  < 2e-16 ***
## I(bio1^2)     0.28801    0.09338   3.084  0.00204 ** 
## I(bio2^2)     0.26184    0.04068   6.436 1.23e-10 ***
## I(bio14^2)    0.42733    0.05970   7.158 8.19e-13 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for poisson family taken to be 1)
## 
##     Null deviance: 8340.1  on 3353  degrees of freedom
## Residual deviance: 6854.8  on 3347  degrees of freedom
## AIC: 7606.8
## 
## Number of Fisher Scoring iterations: 17
```


## Inspect response curves


```r
# check response curves
  # these marginal response curves are evaluated at the means of the non-focal predictor
clim.ranges=apply(values(clim.us),2,range,na.rm=T) # upper and lower limits for each variable
dummy.mean.matrix=data.frame(matrix(0,ncol=nlayers(clim.us),nrow=100)) #makes prediction concise below
names(dummy.mean.matrix)=colnames(clim.ranges) # line up names for later reference
response.curves=lapply(1:nlayers(clim.us),function(x){ # loop over each variable
  xs=seq(clim.ranges[1,x],clim.ranges[2,x],length=100) # x values to evaluate the curve
  newdata=dummy.mean.matrix # data frame with right structure
  newdata[,x]=xs # plug in just the values for the focal variable that differ from mean
  ys=predict(mod.worst,newdata=newdata) # predictions
  return(data.frame(xs=xs,ys=ys)) # define outputs
})# ignore warnings
```


```r
str(response.curves) #structure of the object used for plotting
```

```
## List of 3
##  $ :'data.frame':	100 obs. of  2 variables:
##   ..$ xs: num [1:100] -1.79 -1.74 -1.7 -1.65 -1.61 ...
##   ..$ ys: num [1:100] -18.5 -18.4 -18.3 -18.3 -18.2 ...
##  $ :'data.frame':	100 obs. of  2 variables:
##   ..$ xs: num [1:100] -4.75 -4.68 -4.61 -4.54 -4.47 ...
##   ..$ ys: num [1:100] -13.7 -13.8 -13.9 -14 -14.1 ...
##  $ :'data.frame':	100 obs. of  2 variables:
##   ..$ xs: num [1:100] -2.52 -2.46 -2.4 -2.34 -2.28 ...
##   ..$ ys: num [1:100] -10.3 -10.5 -10.7 -10.9 -11 ...
```


```r
  # plot the curves
par(mfrow=c(2,2),mar=c(4,5,.5,.5)) # # rows and cols for plotting
for(i in 1:nlayers(clim.us)){ # loop over layers
  plot(response.curves[[i]]$xs,response.curves[[i]]$ys,
       type='l',bty='n',las=1,xlab=colnames(clim.ranges)[i],ylab='occurence rate',ylim=c(-20,20))
  pres.env.range=range(pres.data[names(clim.us)[i]]@data)
  abline(v=pres.env.range,col='red',lty=2)
}
```

![](101SDMs_files/figure-html/unnamed-chunk-15-1.png)<!-- -->


## Map predictions

> Decision: When predicting, its ok to extrapolate beyond 


```r
# predict to US
pred.r=raster::predict(clim.us,mod.worst, index=1,type="response")
pred.r=pred.r/sum(values(pred.r),na.rm=T) # normalize prediction (sum to 1)
plot(log(pred.r)) # plot raster
plot(pres,add=T) # plot points
```

![](101SDMs_files/figure-html/unnamed-chunk-16-1.png)<!-- -->


## Evaluate performance

```r
# evaluate
pred.at.fitting.pres=raster::extract(pred.r,pres.data) # get predictions at pres locations
pred.at.fitting.bg=raster::extract(pred.r,bg.data) # get predictions at background locations
rocr.pred=ROCR::prediction(predictions=c(pred.at.fitting.pres,pred.at.fitting.bg),
                          labels=c(rep(1,length(pred.at.fitting.pres)),rep(0,length(pred.at.fitting.bg)))) # define the prediction object needed by ROCR
perf.fit=performance(rocr.pred,measure = "tpr", x.measure = "fpr") # calculate perfomance 
plot(perf.fit) # plot ROC curve
abline(0,1) # 1:1 line indicate random predictions 
```

![](101SDMs_files/figure-html/unnamed-chunk-17-1.png)<!-- -->

```r
(auc_ROCR <- performance(rocr.pred, measure = "auc")@y.values[[1]]) # get AUC
```

```
## [1] 0.9431062
```

## Transfer to new conditions

```r
# transfer to Europe
# choose domain (just europe)
clim.eu=raster::crop(clim,c(-10,55,30,75))
values(clim.eu)=sapply(1:nlayers(clim.eu),function(x) (values(clim.eu)[,x]-clim.means[x])/clim.sds[x])
names(clim.eu)=names(clim.us)
# z-scores (to make values comparable to the scaeld values for fitting)
transfer.r=raster::predict(clim.eu,mod.worst, index=1,type="response")
transfer.r=transfer.r/sum(values(transfer.r),na.rm=T) # normalize prediction (sum to 1)
plot(log(transfer.r)) # plot preds
plot(pres,add=T) # plot presences 
```

![](101SDMs_files/figure-html/unnamed-chunk-18-1.png)<!-- -->



# Improvements

## Thin presences, Stratify sampling

## Sampling bias


```r
bias.bg=read.csv('/Users/ctg/Dropbox/Projects/Workshops/YaleBGCCourses/101_assets/Bias_IPANE_allPoints.csv')
coordinates(bias.bg)=c(2,3)
```


## Model comparison

## Other algorithms: glmnet


<!-- # # evaluate transfer -->
<!-- # pred.at.transfer.pres=raster::extract(transfer.r,pres.data) -->
<!-- #   # sample background in transfer region -->
<!-- # all.background=which(complete.cases(values(clim.us))) -->
<!-- # bg.index=sample(all.background,10000) -->
<!-- # bg.data=data.frame(values(clim.us)[bg.index,]) -->
<!-- # coordinates(bg.data)=coordinates(clim.us)[bg.index,] -->
<!-- #  -->
<!-- # transfer.bg= -->
<!-- # pred.at.fitting.bg=raster::extract(transfer.r,bg.data) -->
<!-- # rocr.pred=ROCR::prediction(predictions=c(pred.at.fitting.pres,pred.at.fitting.bg), -->
<!-- #                           labels=c(rep(1,length(pred.at.fitting.pres)),rep(0,length(pred.at.fitting.bg)))) -->
<!-- # perf.fit=performance(rocr.pred,measure = "tpr", x.measure = "fpr") -->
<!-- # plot(perf.fit) -->
<!-- # abline(0,1) -->
<!-- # (auc_ROCR <- performance(rocr.pred, measure = "auc")@y.values[[1]]) -->
<!-- #  -->
