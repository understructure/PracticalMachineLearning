# Practical Machine Learning - Course Project
Maashu C  
March 15, 2015  

### Data Loading, Cleaning, and Exploratory Analysis

First, I loaded the data and checked the number of rows to make sure it was correct, and examined the names of the columns.  After doing a little exploratory analysis, it was clear that there were a lot of rows that had very little data in them (about 98% NA or missing data).

I created two simple functions to test for blank or NA values and used sapply() to execute them against every column of the dataset.  I then used the column names from that list to remove those columns from the dataset.  This left only 60 variables in the cleaned data.frame, compared to the 160 in the raw data.

Next, I decided to take out the timestamp values, as they all provided basically the same information, and I didn't feel any of that information would be predictive.


```r
rm(list=ls())
setwd("~/Documents/datsci/Coursera - Practical Machine Learning")
myData <- read.csv("pml-training.csv")
```


```r
nrow(myData)

is.blank <- function(vec) {
  sum(as.character(vec) == "")
}
is.na.sum <- function(vec) {
  sum(is.na(vec))
}

naz <- sapply(myData, FUN=is.na.sum)
badz <- sapply(myData, FUN=is.blank)
naBadCols <- data.frame(colnames(myData), naz, badz)
badColNames <- as.character(naBadCols[naz == 19216 | badz == 19216,"colnames.myData."])
myDataClean <- myData[,!names(myData) %in% badColNames]
tsVars <- c("cvtd_timestamp", "raw_timestamp_part_1","raw_timestamp_part_2")
myDataClean <- myDataClean[,!names(myDataClean) %in% tsVars]
```

As an exploratory technique, I decided to do a mosaic plot.  These are really eye-catching and informative, as they let you know very quickly if there's a serious imbalance in the difference in sample size between two sets of categorical variables.  As you can see, the plot below shows that, while there are slight differences between the users, it appears that the type of data available per "classe" per user is approximately similar.



```r
library(RColorBrewer)

mosaicplot(table(myData$classe, myData$user_name), color=brewer.pal(6, "Set1"), main="Mosaic Plot - Activity by User Name")
```

![](Practical_Machine_Learning_-_Course_Project_files/figure-html/unnamed-chunk-3-1.png) 

Next, I knew if I wanted to use user_name as a variable, I'd probably want to create some dummy variables and remove the original user_name variable.  This would allow these to be used in algorithms that were sensitive to categorical predictors.  I setup dummy variables for user_name, creating one variable per user, and then removed the user_name variable along with the X variable, as X is just a row number.


```r
myDataClean$carlitos <- ifelse(as.character(myDataClean$user_name) == "carlitos", 1, 0)     
myDataClean$pedro <- ifelse(as.character(myDataClean$user_name) == "pedro", 1, 0)     
myDataClean$adelmo <- ifelse(as.character(myDataClean$user_name) == "adelmo", 1, 0)     
myDataClean$charles <- ifelse(as.character(myDataClean$user_name) == "charles", 1, 0)     
myDataClean$eurico <- ifelse(as.character(myDataClean$user_name) == "eurico", 1, 0)     
myDataClean$jeremy <- ifelse(as.character(myDataClean$user_name) == "jeremy", 1, 0) 
myDataClean <- myDataClean[,!names(myDataClean) %in% "user_name"]
myDataClean <- myDataClean[,!names(myDataClean) %in% "X"]
```

Next, I decided it would be good to get rid of any variables that were highly correlated with one or more variables.  I used 0.8 as my cutoff point, and removed the "classe" variable and anything I'd created after it (e.g., the dummy variables for user_name) from my analysis.  I set the diagonal of the correlation matrix I created to 0 so the perfect correlations between variables and themselves wouldn't show up.


```r
#use this to check for correlations b/t vars of a certain threshold

yVarColNum <- which(colnames(myDataClean) == "classe")
#colnames(myDataClean)
lastCols <- seq(from=yVarColNum, to=ncol(myDataClean))
corz <- cor(myDataClean[,-c(lastCols)])
class(corz)
dim(corz)
diag(corz) <- 0
high.cor.cellz <- which(abs(corz) > .8)
length(high.cor.cellz)
```

I went through a series of trial and error at this point, removing variables that had the most correlations with others first, keeping as many variables as I could in the analysis until there were no more correlations higher than my chosen threshold of 0.8.  I then removed these from the cleaned data set.

I used the below algorithm to get the names of the variables from the matrix, and then iteratively, manually removed one at a time, removing the ones with the highest number of highly-correlated variables first.  For instance, roll_belt had a high correlation with eight other variables, so I removed that variable first and then ran this analysis again to find variables with six high correlations, then variables with four or five high correlations, etc.


```r
k <- arrayInd(high.cor.cellz, dim(corz))

highCorz <- character(0)
if (nrow(k) > 0) {
   for (i in 1:nrow(k)) {
   highCorz <- c(highCorz, mapply(`[[`, dimnames(corz), k[i,]))
   }
   table(highCorz)
} else {
 cat("no highly correlated variables")
}
```

Ultimately, I was able to remove 12 variables in this manner.


```r
highly.correlated <- c("roll_belt", "gyros_forearm_z", "accel_belt_x", "accel_belt_y", "accel_arm_x", "pitch_dumbbell", "accel_dumbbell_z", "accel_belt_z", "gyros_arm_x", "gyros_dumbbell_z", "magnet_arm_z", "pitch_belt")
myDataClean <- myDataClean[,!names(myDataClean) %in% highly.correlated]
```

Next, I created a total of ten folds with which to split the training set into.  

```r
library(lubridate)
set.seed(floor(seconds(now())))

library(caret)
tenfold <- createFolds(myDataClean$classe, k = 10, list = FALSE)
fold1 <- myDataClean[tenfold == 10,]; fold2 <- myDataClean[tenfold == 9,]
fold3 <- myDataClean[tenfold == 8,]; fold4 <- myDataClean[tenfold == 7,]
fold5 <- myDataClean[tenfold == 6,]; fold6 <- myDataClean[tenfold == 5,]
fold7 <- myDataClean[tenfold == 4,]; fold8 <- myDataClean[tenfold == 3,]
fold9 <- myDataClean[tenfold == 2,]; fold0 <- myDataClean[tenfold == 1,]
```

### Model Creation

Based on Dr. Leek's statement that random forest and gradient boosting win Kaggle competitions more than any other algorithm, I thought I'd start with fitting models with those two methods. 

```r
fit.1 <- train(classe ~ ., method="rf", data=fold1)
pred.1 <- predict(fit.1, newdata=fold2)
confusionMatrix(fold2$classe, pred.1)

fit.2 <- train(classe ~ ., method="gbm", data=fold1)
pred.2 <- predict(fit.2, newdata=fold2)
confusionMatrix(fold2$classe, pred.2)
```
Both random forest and gradient boosting did very well, but random forest performed better when tested against an unseen validation-type dataset, so I decided to go with that.


```r
val.1 <- predict(fit.1, fold3)
val.2 <- predict(fit.2, fold3)

confusionMatrix(fold3$classe, val.1)
confusionMatrix(fold3$classe, val.2)
```

I also tried J48 (C4.5-like decision tree), linear discriminant analysis, Naive Bayes, rpart, rocc, blackboost, regularized random forest (RRF), ORFsvm, gamboost, and knn models.  J48 and lda were the best, but weren't as good as random forest or gradient boosting (gbm).  Naive Bayes, rpart and rocc did poorly against the training set.  I have to admit that my patience ran out with Blackboost, RFF and ORFsvm, and I stopped them after 30-60 minutes of running each.  Even on a subsample with 500 cases, the Boruta algorithm ran for 90 minutes before I decided to stop it.  Here are some examples of other models I created.  I tested J48 against a validation set as well just to be sure it didn't perform better on the training set than random forest.


```r
fit.3 <- train(classe ~ ., method="J48", data=fold1)
pred.3 <- predict(fit.3, newdata=fold2)
confusionMatrix(fold2$classe, pred.3)

fit.4 <- train(classe ~ ., method="lda", data=fold1)
pred.4 <- predict(fit.4, newdata=fold2)
confusionMatrix(fold2$classe, pred.4)

val.3 <- predict(fit.3, fold3)
confusionMatrix(fold3$classe, val.3)
```

### Out of Sample Error

In order to calculate this, I performed several cross-validation tests.  These were manual tests where I predicted one fold from another by writing the calls myself other than using a pre-defined function.  To start, I used the first fold to predict the second using rf.  Next, I used the third fold to predict the fourth, and so on, for a total of five fits and five predictions.

The accuracy for the models ranged from ~ 0.954 to ~ 0.972, so I expected to get about 19 out of 20 predictions correct on the first submission (which is exactly what I got). 

To verify my assumption, I averaged all of these together to get my expected out of sample error, which was ~ 0.964.  I wouldn't have been surprised if I'd gotten all the predictions right, or if I'd only gotten 18 out of 20, or 90% correct.


```r
fit.134 <- train(classe ~ ., method="rf", data=fold3)
pred.134 <- predict(fit.134, newdata=fold4)

fit.156 <- train(classe ~ ., method="rf", data=fold5)
pred.156 <- predict(fit.156, newdata=fold6)

fit.178 <- train(classe ~ ., method="rf", data=fold7)
pred.178 <- predict(fit.178, newdata=fold8)

fit.190 <- train(classe ~ ., method="rf", data=fold9)
pred.190 <- predict(fit.190, newdata=fold0)

avg.accuracy <- sum(
confusionMatrix(fold2$classe, pred.1)$overall["Accuracy"],
confusionMatrix(fold4$classe, pred.134)$overall["Accuracy"],
confusionMatrix(fold0$classe, pred.190)$overall["Accuracy"],
confusionMatrix(fold8$classe, pred.178)$overall["Accuracy"],
confusionMatrix(fold6$classe, pred.156)$overall["Accuracy"]
) / 5
```

### Results and Test Set Performance

The test set data was loaded and prepared to make it look the same as the training data.  I created a total of seven predictions for seven different models, one for the original gbm, one for the J48, and five for random forest. 

I used the values from the first set of random forest predictions to submit my first set of answers, and as predicted got 19 out of 20 correct.  I realized the stupid mistake I'd made, and then ran the other four random forest models in addition to the gbm and J48, just to get the perspective of another model.  I created a data.frame so I could look at all of the random forest models together to see which classe variable was chosen by majority vote, and realized that the variable fit.134 had the set of outcomes that had the highest majority vote,  matched all of the ones I'd gotten right, and did not match the one I'd gotten wrong.  It turned out that this was the correct set of answers.

```r
#load testing data
test.raw <- read.csv("pml-testing.csv")
# Make test data columns look like training set
test.raw <- test.raw[,!names(test.raw) %in% tsVars]
test.raw <- test.raw[,!names(test.raw) %in% badColNames]
test.raw <- test.raw[,!names(test.raw) %in% "X"]
test.raw <- test.raw[,!names(test.raw) %in% "problem_id"]
test.raw <- test.raw[,!names(test.raw) %in% highly.correlated]
test.raw$carlitos <- ifelse(as.character(test.raw$user_name) == "carlitos", 1, 0)     
test.raw$pedro <- ifelse(as.character(test.raw$user_name) == "pedro", 1, 0)     
test.raw$adelmo <- ifelse(as.character(test.raw$user_name) == "adelmo", 1, 0)     
test.raw$charles <- ifelse(as.character(test.raw$user_name) == "charles", 1, 0)     
test.raw$eurico <- ifelse(as.character(test.raw$user_name) == "eurico", 1, 0)     
test.raw$jeremy <- ifelse(as.character(test.raw$user_name) == "jeremy", 1, 0) 
test.raw <- test.raw[,!names(test.raw) %in% "user_name"]
#create predictions
pred.test.1 <- predict(fit.1, test.raw)
pred.test.2 <- predict(fit.2, test.raw)
pred.test.3 <- predict(fit.3, test.raw)
pred.test.134 <- predict(fit.134, test.raw)
pred.test.156 <- predict(fit.156, test.raw)
pred.test.178 <- predict(fit.178, test.raw)
pred.test.190 <- predict(fit.190, test.raw)
#create data frame to compare predictions
data.frame(pred.test.1, pred.test.134, pred.test.156, pred.test.178, pred.test.190)
answers <- pred.test.134
```
### For future research

I realize now that creating dummy variables for the different participants probably didn't help the overall prediction, so if I had to do it over again, I would forego that step.  Also, it would be worth looking into removing even more predictors in order to get a more parsimonious model.  Finally, I think it's important to read up on some of the methods that didn't finish in a reasonable amount of time, and also to look into a method I found called Stagewise Additive Modeling [(SAMME)](http://web.stanford.edu/~hastie/Papers/SII-2-3-A8-Zhu.pdf) to apply the technique of ada boost to a multiclass target variable.
