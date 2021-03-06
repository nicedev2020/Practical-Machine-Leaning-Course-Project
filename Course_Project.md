# Practical-Machine-Leaning-Course-Project
title: "Course Project - Build a machine learning model to predict how well weight lifting exercises is done"
author: "Mayank Kashyap"
date: "03/02/2019"

# Machine Learning Course Project - Build a machine learning model to predict how well weight lifting exercises is done
## Coursera course: Practical Machine Learning

### Load the data

```{r,warning=FALSE}
library(readr)
training <- read_csv("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv")
testing <-read_csv("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv")
```

### Step 1: Clean data

#### In training and testing set, remove columns which are all NA (since we cannot get any prediction value from them)
```{r}
library(dplyr)
# keep the same columns in training & testing set
testing<-testing[,colSums(is.na(testing))<nrow(testing)]
li<-c(colnames(testing)[1:59],"classe")
training<-training %>% select(li)
rm(li)
```

#### Find columns which contain missing values
```{r}
colnames(training)[colSums(is.na(training))>0]
```

#### Find how many NAs are in these columns
```{r}
sum(is.na(training$magnet_dumbbell_z))
sum(is.na(training$magnet_forearm_y))
sum(is.na(training$magnet_forearm_z))
```

#### Find row index for the NA rows
```{r}
which(is.na(training),arr.ind=TRUE)
```
It seems that there is only one row in the training set that contains NA (row 5373). We can handle it in two ways:
1. Replace it with column mean
2. Drop this row

Considering the size of the training dataset, I'll use #2 and drop this row, resulting in 19621 rows in the training set (previously 19622).
```{r}
training<-training[-c(5373),]
```

#### Find out data type for each column, and only select numeric columns to build the model
```{r}
numeric<-lapply(training,is.numeric)
# add a column for classe variable
numeric$classe<-TRUE
# remove non-numeric columns
numeric$X1<-FALSE
numeric$user_name<-FALSE
numeric$raw_timestamp_part_1<-FALSE
numeric$raw_timestamp_part_2<-FALSE
numeric$cvtd_timestamp<-FALSE
numeric$num_window<-FALSE
numericCol<-unlist(numeric)
training<-training[,numericCol]
testing<-testing[,numericCol]
colnames(testing)[53]<-"classe"
rm(numeric)
```

### Step 2: split the training set into a sub-training (70%) and testing(30%) set
```{r}
library(caret)
set.seed(1)
inTrain<-createDataPartition(y=training$classe,p=0.7,list=FALSE)
training_sub<-training[inTrain,]
testing_sub<-training[-inTrain,]
```


### Step 3: build model on the sub-training set
```{r}
set.seed(1)
# model 1: SVM
#modFit1<-train(classe~.,method="svmRadial",data=training4)
# model 2: gbm
#modFit2<-train(classe~.,method="gbm",data=training4)
# model 3: random forest
#modFit3<-train(classe~.,method="rf",data=training4)
# I've commented out these models here because knitting them to html takes too much time (> 5 hours). Conslusion is that among the three models, random forest has the best performance, so we're using random forest to continue with the prediction. 
# Use parallel implementation to increase performance of the random forest model (it was taking too much time before)
library(parallel)
library(doParallel)
cluster <-makeCluster(detectCores()-1)
registerDoParallel(cluster)
# configure trainControl object
fitControl<-trainControl(method="cv",number=20,allowParallel = TRUE)
# develop training model with fitControl
modFit<-train(classe~.,method="rf",data=training_sub,trControl=fitControl)
# de-register parallel processing cluster
stopCluster(cluster)
registerDoSEQ()
# create a confusion matrix for the model on the sub-testing set to see how accurate the prediction is
ConMatrix<-confusionMatrix(table(testing_sub$classe,predict(modFit,testing_sub)))
```


### Step 4: Prediction
```{r}
# predict
pred<-predict(modFit,testing)
```

### Step 5: (Optional) Check importance of features
```{r}
importance<-varImp(modFit,scale=FALSE)
plot(importance)
```
