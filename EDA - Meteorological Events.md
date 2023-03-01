---
title: "R Notebook"
output: html_notebook
---
This document is about exploring and extracting some insights from the Storm Data which is an official publication of National Oceanic and Atmospheric Administration.
We will download the data as bz2 zip file format.
```{r}
url <- "https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2"
download.file(url, destfile = "./data.zip")
```

Bzfiles can be read without decompositioning. 
```{r}
data <- read.csv(bzfile("./data.zip"))
```
First we look at structure of our dataset.
```{r}
str(data)
```
And let's have a look at first rows of the data:
```{r}
head(data)
```
And dimension of data:
```{r}
dim(data)
```
So we have 37 columns and 902297 rows. As we understand from the first impression from the data we have day and time of starting and ending event, place (county), event type (EVTYPE), length and width, force of event (F), fatalities and injuries from the event, damage types, latitude and longitude knowledge.
But we have NA values in data. So let's have a look how many NAs we have:
```{r}
colSums(is.na(data))
```
We have char columns and there are "" values which can't be detected by is.na function. So let's try to find empty chars:
```{r}
table(data$CROPDMGEXP)
```
We see count of empty chars as 618413. When we use table function we can see empty chars in first item, another example:

```{r}
table(data$PROPDMGEXP)
```
So let's take a look only first item of what table's retrieve in only character type columns:
```{r}
for(i in 1:length(names(data))){
  if (class(data[,i]) =="character"){
    print(names(data)[i])
    print(table(data[,i])[1])
  }
}
```
So we have empty chars in most of char columns. And what is the ratio of empty chars on total rows in same column?

```{r}
for(i in 1:length(names(data))){
  if (class(data[,i]) =="character"){
    print(names(data)[i])
    print(table(data[,i])[1]/902297)
  }
}
```
Some character columns don't have any empty item which are:BGN_DATE,BGN_TIME, TIME_ZONE, STATE, EVTYPE. But in some columns empty chars are more common than others like %80.33 of END_AZI are empty chars. 

```{r}
data$BGN_DATE[1]
```
We already have a time column and in BGN_DATE column time " 0:00:00" is unnecessary:
```{r}
data$BGN_DATE <- gsub(" 0:00:00","",as.character(data$BGN_DATE))
data$BGN_DATE <- as.Date(data$BGN_DATE, format = "%m/%d/%Y")

```

We have 3 columns about date and time, first one is BGN_DATE second one is BGN_TIME and the last one is TIME_ZONE. We can paste all the columns and create a POSIXct column for further analysis.
```{r}
data$date_time <- paste(data$BGN_DATE, data$BGN_TIME, data$TIME_ZONE)
```

Let's see if have what we wanted
```{r}
data$date_time[1]
```
It looks good. Now let's have a look other char columns:
```{r}
str(data)
```
Let's look at END_DATE.
```{r}
unique(sapply(data$END_DATE, nchar))
```
So we can assume we have 4 kind of values in EDN_DATE including empty chars.
```{r}
table(sapply(data$END_DATE, nchar))
```
We have many empty chars on this column. 
Let's have a look at EVTYPE column:
```{r}
sort(unique(data$EVTYPE))
```

There are typos on the EVTYPE column and misplaced empty spaces also. And There are some Summary knowledge which will be investigated afterwards. But let's first we fix the upper lower cases:
```{r}
data$EVTYPE <- toupper(data$EVTYPE)
```

Let's have a look at Wind kind of events:
```{r}
grep("^WIND", unique(data$EVTYPE), value = TRUE)
```
Also there is WND or " WIND":
```{r}
grep("^WND", unique(data$EVTYPE), value = TRUE)
grep("^ WIND", unique(data$EVTYPE), value = TRUE)
```
Let's merge them into one type:
```{r}
data$EVTYPE[data$EVTYPE =="WND" | data$EVTYPE ==" WIND" | data$EVTYPE =="WINDS"] <- "WIND"
```
And there are empty spaces in front of the words.
```{r}
spaced_list <- list(grep("^ ", unique(data$EVTYPE), value = TRUE))
```
Let's fix empty spaces with trimws:
```{r}
for (i in 1:length(spaced_list[[1]])){
  data$EVTYPE[data$EVTYPE== spaced_list[[1]][i]]<-  trimws(spaced_list[[1]][i])
}
```

Now what do we have?
```{r}
unique(data$EVTYPE)
```
There are some abbreviations. TSTM, CSTL, HVY, FLD. Let's replace them with proper words: THUNDERSTROM, COASTAL, HEAVY and FLOOD respectively. 
```{r}
data$EVTYPE <- gsub("TSTM*","THUNDERSTORM", data$EVTYPE)
data$EVTYPE <- gsub("CSTL*","COASTAL", data$EVTYPE)
data$EVTYPE <- gsub("HVY*","HEAVY", data$EVTYPE)
data$EVTYPE <- gsub("FLD*","FLOOD", data$EVTYPE)
```
```{r}
sort(unique(data$EVTYPE))
```

Let's have a look some of the summaries as event type:
```{r}
data[data$EVTYPE=="SUMMARY SEPT. 25-26",]
```
```{r}
data[data$EVTYPE=="SUMMARY OF MARCH 24-25",]
```
Let's group these summaries and see if any injury or damage has recorded.

```{r}
summary(data[grepl("^SUMMARY", data$EVTYPE),])
```
So those rows doesn't contain any injury or damage report. Those rows are kind of summary of some down rows and only contains the locations and it is not important for us. Because at down rows we can see details. SO let's drop these rows from our data sets.
```{r}
data <- data[!grepl("^SUMMARY", data$EVTYPE),]
```

```{r}
dim(data)
```
So we dropped 902297 - 902222 = 75 rows.
