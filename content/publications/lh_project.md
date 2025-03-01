---
title: "TestB"
type: page
description: "DCF Model for Airbnb #ABNB"
topic: model
<!--eofm-->
---
Do left-handed people really die young?

Where are the old left-handed people?

Barack Obama is left-handed. So are Bill Gates and Oprah Winfrey; so were Babe Ruth and Marie Curie. A 1991 study reported that left-handed people die on average nine years earlier than right-handed people. Nine years! Could this really be true?

In this notebook, we will explore this phenomenon using age distribution data to see if we can reproduce a difference in average age at death purely from the changing rates of left-handedness over time, refuting the claim of early death for left-handers. This notebook uses pandas and Bayesian statistics to analyze the probability of being a certain age at death given that you are reported as left-handed or right-handed.

A National Geographic survey in 1986 resulted in over a million responses that included age, sex, and hand preference for throwing and writing. Researchers Avery Gilbert and Charles Wysocki analyzed this data and noticed that rates of left-handedness were around 13% for people younger than 40 but decreased with age to about 5% by the age of 80. They concluded based on analysis of a subgroup of people who throw left-handed but write right-handed that this age-dependence was primarily due to changing social acceptability of left-handedness. This means that the rates aren't a factor of age specifically but rather of the year you were born, and if the same study was done today, we should expect a shifted version of the same distribution as a function of age. 

Ultimately, we'll see what effect this changing rate has on the apparent mean age of death of left-handed people, but let's start by plotting the rates of left-handedness as a function of age. This notebook uses two datasets: death distribution data for the United States from the year 1999 and rates of left-handedness digitized from a figure in this 1992 paper by Gilbert and Wysocki.

Source website of (lh_data.csv & ): 
https://gist.githubusercontent.com/mbonsma/8da0990b71ba9a09f7de395574e54df1/raw/aec88b30af87fad8d45da7e774223f91dad09e88/lh_data.csv
(usdeathdata_cdc.tsv): https://gist.githubusercontent.com/mbonsma/2f4076aab6820ca1807f4e29f75f18ec/raw/62f3ec07514c7e31f5979beeca86f19991540796/cdc_vs00199_table310.tsv


```python
import pandas as pd 
import matplotlib.pyplot as plt

lefthanded_data = pd.read_csv('lh_data.csv')

print(lefthanded_data.head())
print(lefthanded_data.shape)

fig, ax = plt.subplots()
ax.plot("Age", "Female", data=lefthanded_data, marker = "o", color='#ec6ba6', label = "Female")
ax.plot("Age", "Male", data=lefthanded_data, marker = "x", color='#62b2e0', label = "Male")

ax.legend()

ax.set_xlabel('Sex')
ax.set_ylabel('Age')

plt.show()
```

Rates of left-handedness over time

Let's convert this data into a plot of the rates of left-handedness as a function of the year of birth, and average over males and females to get a single rate for both sexes. 

Since the study was done in 1986, the data after this conversion will be the percentage of people alive in 1986 who are left-handed as a function of the year they were born. 


```python
lefthanded_data['Birth_year'] = 1986 - lefthanded_data['Age']
print(lefthanded_data.head())

lefthanded_data['Mean_lh'] = lefthanded_data[['Female', 'Male']].mean(axis=1)
print(lefthanded_data.head())

fig, ax = plt.subplots()
ax.plot("Birth_year", "Mean_lh", data=lefthanded_data, color='#bdb2ff')
ax.set_xlabel('Birth_year')
ax.set_ylabel('Mean_lh')

plt.show()
```

Applying Bayes' rule

The probability of dying at a certain age given that you're left-handed is not equal to the probability of being left-handed given that you died at a certain age. This inequality is why we need Bayes' theorem, a statement about conditional probability which allows us to update our beliefs after seeing the evidence.

We want to calculate the probability of dying at age A given that you're left-handed. Let's write this in shorthand as P(A | LH). We also want the same quantity for right-handers: P(A | RH).

Here's Bayes' theorem for the two events we care about left-handedness (LH) and dying at age A.

equation = P(A|LH)=P(LH|A)P(A)/P(LH)

P(LH | A) is the probability that you are left-handed given that you died at age A. P(A) is the overall probability of dying at age A, and P(LH) is the overall probability of being left-handed. We will now calculate each of these three quantities, beginning with P(LH | A).

To calculate P(LH | A) for ages that might fall outside the original data, we will need to extrapolate the data to earlier and later years. Since the rates flatten out in the early 1900s and late 1900s, we'll use a few points at each end and take the mean to extrapolate the rates on each end. The number of points used for this is arbitrary, but we'll pick 10 since the data looks flat-ish until about 1910.


```python
import numpy as np

# create a function for P(LH | A)
def P_lh_given_A(ages_of_death, study_year = 1990):

    early_1900s_rate = lefthanded_data[['Mean_lh'][-10:]].mean()
    late_1900s_rate = lefthanded_data[['Mean_lh'][:10]].mean()
    middle_rates = lefthanded_data.loc[lefthanded_data['Birth_year'].isin(study_year - ages_of_death)]['Mean_lh']
    youngest_age = study_year - 1986 + 10
    oldest_age = study_year - 1986 + 86 
    
    P_return = np.zeros(ages_of_death.shape)
    P_return[ages_of_death > oldest_age] = early_1900s_rate / 100
    P_return[ages_of_death < youngest_age] = late_1900s_rate / 100
    P_return[np.logical_and((ages_of_death <= oldest_age), (ages_of_death >= youngest_age))] = middle_rates / 100
    
    return P_return
```

When do people normally die?

To estimate the probability of living to an age A, we can use data that gives the number of people who died in a given year and how old they were to create a distribution of ages of death. If we normalize the numbers to the total number of people who died, we can think of this data as a probability distribution that gives the probability of dying at age A. The data we'll use for this is from the entire US for the year 1999 - the closest I could find for the time range we're interested in. 

In this block, we'll load the death distribution data and plot it.
The first column is the age, and the other columns are the number of people who died at that age.


```python
death_distribution_data = pd.read_csv('usdeathdata_cdc.tsv', delimiter='\t', skiprows=[1])

print(death_distribution_data.head())
print(death_distribution_data.shape)

death_distribution_data.dropna(subset = ['Both Sexes'])

fig, ax = plt.subplots()
ax.plot("Age", "Both Sexes", data = death_distribution_data, marker='o', color='#dee524')
ax.set_xlabel("Age") 
ax.set_ylabel("Both Sexes")
 
plt.show()
```

The overall probability of left-handedness

In the previous code block, we loaded data to give us P(A), and now we need P(LH). P(LH) is the probability that a person who died in our particular study year is left-handed, assuming we know nothing else about them. This is the average left-handedness in the population of deceased people, and we can calculate it by summing up all of the left-handedness probabilities for each age, weighted with the number of deceased people at each age, then divided by the total number of deceased people to get a probability. In equation form, this is what we're calculating, where N(A) is the number of people who died at age A (given by the data frame death_distribution_data):

equation: P(LH) = Sum of P(LH|A)N(A)/Sum of N(A)


```python
def P_lh(death_distribution_data, study_year = 1990): 
    p_list = death_distribution_data['Both Sexes'] * P_lh_given_A(death_distribution_data['Age'], study_year)
    p = np.sum(p_list)
    return p / np.sum(death_distribution_data['Both Sexes']) 

print(P_lh(death_distribution_data))
```

Putting it all together: dying while left-handed

Now we have the means of calculating all three quantities we need: P(A), P(LH), and P(LH | A). We can combine all three using Bayes' rule to get P(A | LH), the probability of being age A at death (in the study year) given that you're left-handed. To make this answer meaningful, though, we also want to compare it to P(A | RH), the probability of being age A at death given that you're right-handed.

We're calculating the following quantity twice, once for left-handers and once for right-handers.

First, for left-handers:


```python
def P_A_given_lh(ages_of_death, death_distribution_data, study_year = 1990):
    P_A = death_distribution_data['Both Sexes'][ages_of_death] / np.sum(death_distribution_data['Both Sexes'])
    P_left = P_lh(death_distribution_data, study_year)
    P_lh_A = P_lh_given_A(ages_of_death, study_year)
    return P_lh_A*P_A/P_left
```

Then, for right-handers:


```python
def P_A_given_rh(ages_of_death, death_distribution_data, study_year = 1990):
    """ The overall probability of being a particular `age_of_death` given that you're right-handed """
    P_A = death_distribution_data['Both Sexes'][ages_of_death] / np.sum(death_distribution_data['Both Sexes'])
    P_right = 1 - P_lh(death_distribution_data, study_year)
    P_rh_A = 1 - P_lh_given_A(ages_of_death, study_year)
    return P_rh_A*P_A/P_right
```

Plotting the distributions of conditional probabilities

Now that we have functions to calculate the probability of being age A at death given that you're left-handed or right-handed, let's plot these probabilities for a range of ages of death from 6 to 120.

Notice that the left-handed distribution has a bump below age 70: of the pool of deceased people, left-handed people are more likely to be younger.


```python
ages = np.arange(6, 115, 1) # make a list of ages of death to plot

left_handed_probability = P_A_given_lh(ages, death_distribution_data)
right_handed_probability = P_A_given_rh(ages, death_distribution_data)

fig, ax = plt.subplots() 
ax.plot(ages, left_handed_probability, color='#ff6361', label = "Left-handed")
ax.plot(ages, right_handed_probability, color='#008585', label = "Right-handed")
ax.legend()
ax.set_xlabel("Age at death")
ax.set_ylabel("Probability of being age A at death")

plt.show()
```

Moment of truth: age of left and right-handers at death

Finally, let's compare our results with the original study that found that left-handed people were nine years younger at death on average. We can do this by calculating the mean of these probability distributions, in the same way, we calculated P(LH) earlier, weighting the probability distribution by age and summing over the result.


```python
average_lh_age = np.nansum(ages * np.array(left_handed_probability))
average_rh_age = np.nansum(ages * np.array(right_handed_probability))

print("Average age of left-handed at death is " + str(average_lh_age))
print("Average age of right-handed at death is " + str(average_rh_age))

print("The difference in average ages is " + str(round(average_rh_age - average_lh_age, 1)) + " years.")
```

Final comments:

We got a pretty big age gap between left-handed and right-handed people purely as a result of the changing rates of left-handedness in the population, which is good news for left-handers: you probably won't die young because of your sinisterness.
The reported rates of left-handedness have increased from just 3% in the early 1900s to about 11% today, which means that older people are much more likely to be reported as right-handed than left-handed, and so looking at a sample of recently deceased people will have more old right-handers. 

Our number is still less than the 9-year gap measured in the study. It's possible that some of the approximations we made are the cause: 

1. We used death distribution data from almost ten years after the study (1999 instead of 1991), and we used death data from the entire United States instead of California alone (which was the original study).

2. We extrapolated the left-handedness survey results to older and younger age groups, but it's possible our extrapolation wasn't close enough to the true rates for those ages.
One thing we could do next is figuring out how much variability we would expect to encounter in the age difference purely because of random sampling: if you take a smaller sample of recently deceased people and assign handedness with the probabilities of the survey, what does that distribution look like? How often would we encounter an age gap of nine years using the same data and assumptions? We won't do that here, but it's possible with this data and the tools of random sampling. 

To finish off, let's calculate the age gap we'd expect if we did the study in 2018 instead of in 1990. The gap turns out to be much smaller since rates of left-handedness haven't increased for people born after about 1960. Both the National Geographic study and the 1990 study happened at a unique time - the rates of left-handedness had been changing across the lifetimes of most people alive, and the difference in handedness between old and young was at its most striking.


```python
left_handed_probability_2018 = P_A_given_lh(ages, death_distribution_data, 2018)
right_handed_probability_2018 = P_A_given_rh(ages, death_distribution_data, 2018)

average_lh_age_2018 = np.nansum(ages*np.array(left_handed_probability_2018))
average_rh_age_2018 = np.nansum(ages*np.array(right_handed_probability_2018))

print("Average age of left-handed at death in 2018 is " + str(average_lh_age_2018))
print("Average age of right-handed at death in 2018 is " + str(average_rh_age_2018))

print("The difference of average ages between left-handed and right-handed at death in 2018 is " + 
      str(round(average_rh_age_2018 - average_lh_age_2018, 1)) + " years.")
```

@Kamangnt
