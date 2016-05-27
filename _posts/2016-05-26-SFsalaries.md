---
layout: post
title: "Gender Pay Gap"
date: 2016-05-26
output:
  html_document
share: true
categories: blog
excerpt: "SF Salaries"
tags: [rstats]
---



## Libraries

{% highlight r %}
# load packages
pacman::p_load(ggplot2, reader, gender, ggthemes, gridExtra, GGally, stringr, 
    dplyr, tm, yarrr, DT, plotly, ggalt, tidyr, scales, webshot)
# check that packages were loaded
pacman::p_loaded(ggplot2, reader, gender, ggthemes, gridExtra, GGally, stringr, 
    dplyr, tm, yarrr, DT, plotly, ggalt, tidyr, scales, webshot)
{% endhighlight %}



{% highlight text %}
##   ggplot2    reader    gender  ggthemes gridExtra    GGally   stringr 
##      TRUE      TRUE      TRUE      TRUE      TRUE      TRUE      TRUE 
##     dplyr        tm     yarrr        DT    plotly     ggalt     tidyr 
##      TRUE      TRUE      TRUE      TRUE      TRUE      TRUE      TRUE 
##    scales   webshot 
##      TRUE      TRUE
{% endhighlight %}

##Data loading, feature creation, cleaning, pre-processing
Let's load up the file first


{% highlight r %}
salaries <- read.csv("Salaries.csv", na.strings = c("Not Provided", " ", ""), 
    strip.white = TRUE)
{% endhighlight %}

Let's convert all the names to lower character and remove all punctuation. 


{% highlight r %}
salaries$EmployeeName <- tolower(salaries$EmployeeName)
salaries$EmployeeName <- removePunctuation(salaries$EmployeeName)
# strip the left whitespace
salaries$EmployeeName <- trimws(salaries$EmployeeName, which = "left")
{% endhighlight %}

Now, let's remove one letter strings. We're removing the one or two letter words because the naming convention for an employee is not always consistent from year to year. For example, lets look at a transit operator named Aaran Luo:


{% highlight r %}
filter(salaries, grepl("aaran", EmployeeName)) %>% select(EmployeeName, JobTitle, 
    Year, TotalPay)
{% endhighlight %}



{% highlight text %}
##   EmployeeName         JobTitle Year TotalPay
## 1  aaran y luo Transit Operator 2014 75252.84
## 2    aaran luo Transit Operator 2012 78118.96
## 3  aaran y luo Transit Operator 2013 82026.78
{% endhighlight %}

There are two conventions for how the employee name was recorded : `aaran y luo` or `aaran luo`. Nontheless, it would be highly unusual for these naming conventions to actually represent two different employees. Hence, I remove all the middle initial characters from the EmployeeName variable.


{% highlight r %}
salaries$EmployeeName <- gsub(" *\\b[[:alpha:]]{1}\\b *", " ", salaries$EmployeeName)
{% endhighlight %}

Now, let us create a new variable with the first names.


{% highlight r %}
salaries$name <- word(salaries$EmployeeName, 1)
{% endhighlight %}

Let's use the Social Security Administration(`ssa`) gender data that represents the names prevelent in United States from 1930 to 2012. **If you're using the `ssa` for the first time, you'll need to wait a little bit for the genderData dependency to install.


{% highlight r %}
genderDF <- salaries %>% distinct(name) %>% do(results = gender(.$name, method = "ssa")) %>% 
    do(bind_rows(.$results))

mergedDF <- salaries %>% left_join(genderDF, by = c(name = "name"))

rm(genderDF)
rm(salaries)
{% endhighlight %}

Now, let's examine if there are any outliers present in the data. I'll examine if there are any unusual values in total pay. First, I'll examine the top 10 obervations for total pay and the bottom 10 observations in total pay. 


{% highlight r %}
mergedDF %>% select(EmployeeName, JobTitle, TotalPayBenefits) %>% arrange(desc(TotalPayBenefits)) %>% 
    head(10)
{% endhighlight %}



{% highlight text %}
##         EmployeeName                                       JobTitle
## 1     nathaniel ford GENERAL MANAGER-METROPOLITAN TRANSIT AUTHORITY
## 2       gary jimenez                CAPTAIN III (POLICE DEPARTMENT)
## 3        david shinn                                 Deputy Chief 3
## 4           amy hart                              Asst Med Examiner
## 5  william coaker jr                       Chief Investment Officer
## 6       gregory suhr                                Chief of Police
## 7  joanne hayeswhite                         Chief, Fire Department
## 8       gregory suhr                                Chief of Police
## 9  joanne hayeswhite                         Chief, Fire Department
## 10     ellen moffatt                              Asst Med Examiner
##    TotalPayBenefits
## 1          567595.4
## 2          538909.3
## 3          510732.7
## 4          479652.2
## 5          436224.4
## 6          425815.3
## 7          422353.4
## 8          418019.2
## 9          417435.1
## 10         415767.9
{% endhighlight %}

It seems like we have some individuals who earn more than half a million dollars in pay, which is not that unusual considering the seniority of their positions and that SF is one of the most expensive cities in the US. It is interesting that Assistant Medical Examiners earn 400k. If you check the following link on [indeed.com](http://www.indeed.com/salary/q-Assistant-Medical-Examiner-l-San-Francisco,-CA.html), you'll note that this job title generally gets about $65K a year. With that aside, let's check the lowest salaries.


{% highlight r %}
mergedDF %>% select(EmployeeName, JobTitle, TotalPayBenefits) %>% arrange(desc(TotalPayBenefits)) %>% 
    tail(10)
{% endhighlight %}



{% highlight text %}
##              EmployeeName                   JobTitle TotalPayBenefits
## 148645      kaukab mohsin           TRANSIT OPERATOR             0.00
## 148646 josephine mccreary                 MANAGER IV             0.00
## 148647       not provided               Not provided             0.00
## 148648       not provided               Not provided             0.00
## 148649       not provided               Not provided             0.00
## 148650       not provided               Not provided             0.00
## 148651     timothy gibson           Police Officer 3            -2.73
## 148652       mark laherty           Police Officer 3            -8.20
## 148653        david kucia           Police Officer 3           -33.89
## 148654          joe lopez Counselor, Log Cabin Ranch          -618.13
{% endhighlight %}

We have a bunch of issues. First, we have three persons with negative pay. Also, we have some rows which should've been tagged as NA data (the not provided rows). Third, we have individuals who have 0 pay. Let's remove the `Not Provided` rows and the entry for individuals with negative pay and then we'll tackle individuals who had 0 pay. 


{% highlight r %}
# remove not provided rows
mergedDF <- mergedDF %>% filter(EmployeeName != "not provided")

# remove joe lopez
mergedDF <- mergedDF %>% filter(TotalPayBenefits >= 0)

# check the rows with Total Pay of 0
zeroPay <- mergedDF %>% filter(TotalPayBenefits == 0) %>% select(EmployeeName, 
    JobTitle, TotalPayBenefits)
datatable(zeroPay)
{% endhighlight %}

<img src="/figs/2016-05-26-SFsalaries/unnamed-chunk-10-1.png" title="center" alt="center" style="display: block; margin: auto;" />

We have 26 individuals who had 0 pay. One reason why their pay was 0 is that they just registered as employees and did not recieve a salary yet. Another reason is that they are registered as employees, but had to take a sabatical with no pay. Regardless of the reason, the number of people with zero pay is really small and we can feel safe removing them.


{% highlight r %}
mergedDF <- mergedDF %>% filter(TotalPayBenefits != 0)
{% endhighlight %}

Even though we removed employees with zero pay, we still need to explore if there are individuals who recieved really low pay. Let's examine the distribution of the TotalPayBenefits variable in order to answer that question. I will break the distribution of total pay by Status (FT=Full Time, PT=Part Time, NA)


{% highlight r %}
theme_set(theme_solarized())
ggplot(mergedDF, aes(x = TotalPayBenefits/1000)) + geom_histogram(aes(y = ..density..), 
    binwidth = density(mergedDF$TotalPayBenefits/1000)$bw) + geom_density(fill = "green", 
    alpha = 0.2) + facet_grid(. ~ Status) + scale_x_continuous(breaks = seq(0, 
    300, 50), limits = c(0, 300)) + ggtitle("Density distributions of salary (in thousands) for each status") + 
    labs(y = "Density", x = "Salary (thousands)")
{% endhighlight %}

<img src="/figs/2016-05-26-SFsalaries/unnamed-chunk-12-1.png" title="center" alt="center" style="display: block; margin: auto;" />

It seems like we have two groups of individuals. One group represents the part timers and the other represents the group that had a full time salary. The NA responses have a mixture of the two groups (note the bimodality in the distribution of responses). There are a couple observations to be made:

* The full time employees seem to earn at least 50k salary. 
* The part time employees have a much wider range in salary, with salaries starting at near 0 to 300k. 


Next, we will clean up the JobTitle variable. The Job Title category has quite a few issues
* The job category is not standardized. Some jobs come with abbreviations and some have the fully title (Assist. vs Assistant)
* Some job titles are plainly misspelled. 



{% highlight r %}
# library(qdap) spelling_result<-check_spelling(mergedDF$JobTitle, range =
# 2, assume.first.correct = FALSE, method = 'jw', dictionary =
# qdapDictionaries::GradyAugmented, parallel = TRUE, cores =
# parallel::detectCores()/2, n.suggests = 3)
{% endhighlight %}





Let's look at the gender counts for the Full Time and Part Time employees. We'll leave employees with NA aside for now. 


{% highlight r %}
# remove data for which no gender is available; this removed about 10,000
# observations.
genderDF <- mergedDF %>% filter(gender != "NA")

# remove the jobs with NA status.
genderDF <- genderDF %>% filter(Status != "NA")
{% endhighlight %}



{% highlight r %}
countsDf <- genderDF %>% count(Status, gender)

counts <- ggplot(data = countsDf, aes(x = Status, y = n, fill = gender)) + geom_bar(stat = "identity", 
    position = position_dodge(), colour = "black") + scale_fill_manual(values = c("#999999", 
    "#E69F00")) + ggtitle("Number of employees by gender and status") + labs(y = "# Employees", 
    x = "Status")
{% endhighlight %}

 

{% highlight r %}
ggplotly(counts)
{% endhighlight %}

<img src="/figs/2016-05-26-SFsalaries/unnamed-chunk-16-1.png" title="center" alt="center" style="display: block; margin: auto;" />


It seems like more females are working part relative to males. Moreover, it seems like there are more male full time employees than part time female employees. Let us check the pay distribution for full time and the part time male and female employees


{% highlight r %}
meanGenderDf <- genderDF %>% select(Status, gender, TotalPayBenefits) %>% filter(TotalPayBenefits >= 
    quantile(TotalPayBenefits, 0.01), TotalPayBenefits <= quantile(TotalPayBenefits, 
    0.99)) %>% group_by(Status, gender) %>% summarise(meanSalary = mean(TotalPayBenefits))

meanPlot <- ggplot(data = meanGenderDf, aes(x = Status, y = meanSalary/1000, 
    fill = gender)) + geom_bar(stat = "identity", position = position_dodge(), 
    colour = "black") + scale_fill_manual(values = c("#999999", "#E69F00")) + 
    ggtitle("Mean Salary by gender and status") + labs(y = "Mean Salary", x = "Status") + 
    scale_y_continuous(breaks = seq(0, 160, 20), labels = sprintf("$%sK", comma(seq(0, 
        160, by = 20))))

ggplotly()
{% endhighlight %}

<img src="/figs/2016-05-26-SFsalaries/unnamed-chunk-17-1.png" title="center" alt="center" style="display: block; margin: auto;" />

It seems like fully employed men earn on average higher salary than females. **Fully employed women earn about 89cents for every dollar earned by fully employed men.**


Let's see if the same pattern holds for median salary


{% highlight r %}
medianGenderDF <- genderDF %>% select(Status, gender, TotalPayBenefits) %>% 
    group_by(Status, gender) %>% summarise(medianSalary = median(TotalPayBenefits))


medianPlot <- ggplot(data = medianGenderDF, aes(x = Status, y = medianSalary/1000, 
    fill = gender)) + geom_bar(stat = "identity", position = position_dodge(), 
    colour = "black") + scale_fill_manual(values = c("#999999", "#E69F00")) + 
    ggtitle("Median Salary by gender and status") + labs(y = "Median Salary", 
    x = "Status") + scale_y_continuous(breaks = seq(0, 160, 20), labels = sprintf("$%sK", 
    comma(seq(0, 160, by = 20))))

ggplotly()
{% endhighlight %}

<img src="/figs/2016-05-26-SFsalaries/unnamed-chunk-18-1.png" title="center" alt="center" style="display: block; margin: auto;" />


The picture looks even worse. **On median, fully employed women earn about 84 cents for every dollar earned by fully employed males.**

