---
title: "Reproducible Research: Peer Assessment 1"
output: 
  html_document:
    keep_md: true
---


## Loading and preprocessing the data

### Decompression and Loading

The First Step to be taken in order to analyze the Data is to load it into the R environment. For this purpose, the Source files have to be uncompressed (through unzip()). After the decompression, the created .csv file is read into the environment using read.csv


```r
  unzip("./activity.zip")
  data <- read.csv("./activity.csv")
```

### Preprocessory Steps


```r
  str(data)
```

```
## 'data.frame':	17568 obs. of  3 variables:
##  $ steps   : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ date    : chr  "2012-10-01" "2012-10-01" "2012-10-01" "2012-10-01" ...
##  $ interval: int  0 5 10 15 20 25 30 35 40 45 ...
```
The first practical Step after loading the data is to look at the structure of the Dataset. Together with the questions that the Analysis is trying to answer, one can assess adequate preprocessing steps to take. 

In this case, a lot of the questions depend on the dates. The dates in the Dataset are Strings. It would therefore make sense to first transform the values iinto R Date variables. 


```r
  data$date <- as.Date(data$date, "%Y-%m-%d")
  str(data$date)
```

```
##  Date[1:17568], format: "2012-10-01" "2012-10-01" "2012-10-01" "2012-10-01" "2012-10-01" ...
```

## What is mean total number of steps taken per day?

This chapter focuses on summarizing and visualizing the key values of the Steps taken per day. 

### Total Number of Steps per Day

The first step is to calculate the *Total Number of Steps per day* and visualize them through a histogram. For this we use the following Code making use of the group_by and summarize_at functions of the dplyr Package:


```r
  library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
  total_values <- data %>% group_by(date) %>%  summarize_at("steps", sum)
  hist(total_values$steps, col="cyan", xlab="Total Number of Steps", main = "Histogram of the Total Number of Steps per day")
```

![](PA1_files/figure-html/Calculating and visualizing the Total NUmber of steps per Day-1.png)<!-- -->

The Histogram shows a roughly normal distribution of Number of Steps per day. 

### Mean and Median Number of Steps per Day

The next key numbers to calculate will be the *Mean* and the *Median* Number of Steps taken per day. FOr this, the total_values calculated in the last chunk are used. For now, we are going to be removing the days that contain missing values by setting the option na.rm equal to TRUE: 


```r
  mean_steps <- mean(total_values$steps, na.rm=TRUE)
  median_steps <- median(total_values$steps, na.rm = TRUE)
```

From the Calculation we get:

- The *Mean* Number of Steps taken per day is: 10766.19 (Rounded here to two digits for readability)
- The *Median* Number of Steps taken per day is: 10765

### Visualization

To conclude the key numbers, we once again look at the Histogram of the *Total Number of Steps per day*, Adding the Mean/Median values into the graph. As Mean and Median are only off by 1 Step, we can choose one of them for the visualization. In this case we use the mean value: 


```r
  hist(total_values$steps, col="cyan", xlab="Total Number of Steps", main = "Histogram of the Total Number of Steps per day")
  abline(v = mean_steps, col="magenta", lwd=3)
  legend("topright", legend= "Mean No. Steps/Day", col="magenta" ,lty=1, lwd=3)
```

![](PA1_files/figure-html/Visualizing Mean and Median-1.png)<!-- -->

## What is the average daily activity pattern?

After looking at the key numbers in the last Section, the question now becomes what underlying patterns can be found in the data. In this Section, the Average daily Activity Pattern is calculated and visualized through a time series plot

### Meaning 

The Steps in the Dataset are recorded in a 5 Minute Interval for every day. To get the average Steps of a specific time interval (e.g. The Average number of steps Taken between 05:10 and 05:15 pm ) the Average of that interval over all recorded days is taken. All averages combined can bee seen as the **Average daily Activity**.

### Calculation & Visualizing

The Average daily Activity Pattern is gathered by usage of the group_by and summarize_at functions of the dplyr package. To ensure we get a Non-NA Value for every Interval, for now we use the option na.rm equal to true. This removers the NA values from the calculation. The Calculation is done by the following Code chunk: 


```r
  interval_values <- data %>% group_by(interval) %>% summarize_at("steps", mean, na.rm=TRUE)
  head(interval_values)
```

```
## # A tibble: 6 x 2
##   interval  steps
##      <int>  <dbl>
## 1        0 1.72  
## 2        5 0.340 
## 3       10 0.132 
## 4       15 0.151 
## 5       20 0.0755
## 6       25 2.09
```

To see what the resulting Data Frame looks like, the *head()* function displays the first 6 rows of the set. 

The Data Frame can now be used to create the Time Series Plot. The intervals are given in minutes, to make the Plot properly readable, it is therefore recommendable to convert the intervals into a formal Date format. Once the intervals are converted, the plot can be created: 


```r
  # first we convert the interval into seconds
  interval_values$interval <- interval_values$interval * 60 
  
  library(lubridate)
```

```
## 
## Attaching package: 'lubridate'
```

```
## The following objects are masked from 'package:base':
## 
##     date, intersect, setdiff, union
```

```r
  # afterwards we can format the intervals using the lubridate package
  interval_values$interval <- Reduce(c, lapply(interval_values$interval, function(x) as_datetime(seconds_to_period(x) )))
  
  library(ggplot2)
  
  p <- ggplot(interval_values, aes(x=interval, y=steps)) +
  geom_line() + 
  xlab("")
  p
```

![](PA1_files/figure-html/Visualizing the Average Daily Activity Pattern through a time Series Plot-1.png)<!-- -->

Looking at the graph, the highest spike seems to occur on average at around 3pm. To confirm this, we calculate the maximum of the Interval averages: 


```r
  max_interval <- interval_values$interval[ interval_values$steps == max(interval_values$steps)]
  max_interval
```

```
## [1] "1970-01-01 14:55:00 CET"
```

The calculation confirms the observation with the highest average activity interval being at 1970-01-01 14:55:00

## Imputing missing values

To better deal with the missing values than simply ignoring them, this section describes a simple strategy for imputing the missing values. 

### Number of missing Values

To get a bit of insight into how large the effect of missing values on the observations is, we will first calculate how many rows with missing values the dataset contains. We do this with the following expression: 


```r
  sum(apply(data, 1, anyNA))
```

```
## [1] 2304
```

We see that the number of missing values is 2304. Given a total value of 17568 observations in the dataset, this means that over 10% of the observations contain missing values. As this could be make a signiificant difference, in the following we will come up with and apply a strategy to impute the missing value with an educated guess. 

### Strategy 

There are clear differences in average activity depending on the time (this was visualized in the Activity Pattern section above). Therefore, simply taking the mean/median step value of the day and use it for the missing values might lead to strongly misleading values. 

Instead, the median value of the interval of the respective missing observation will be used. This is more in line with the patterns observed so far. 

This is how the process of replacing the missing values is done: 



## Are there differences in activity patterns between weekdays and weekends?
