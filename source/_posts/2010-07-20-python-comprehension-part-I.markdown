---
layout: post
title:  "List comprehensions in Python - Part I"
categories: python
comments: true
---

List comprehensions allow you to constructs arrays (lists in Python) of elements by describing the expression to build them, very similar to the way you write sets descriptions in mathematics. You will find many tutorials about this on the web, so as usual in this blog, I will focus on practical examples that took me a little bit more that just reading the how-tos.

<!-- more -->

Here is the problem that will drive this post. Given a set of ages, and for each age, a number of people of that age in a considered population, the goal is to write a procedure that draws a random age respecting that distribution. 

Let’s first start with the datastructure. As usual in Python, thanks to builtin dictionaries and tuples, no need to start with a `Age` or `Distribution` class. A dictionary will suffice:

``` python
distribution = [
	[20,1],
	[21,2],
	[23,3],
	[24,4],
	[25,3],
	[26,2],
	[27,1],
]
```

Let’s rewrite it to make it easier to visualize:

``` python
distribution = [
	[20, len('*')],
	[21, len('**')],
	[23, len('***')],
	[24, len('****')],
	[25, len('***')],
	[26, len('**')],
	[27, len('*')],
]
```

Now the distribution can litteraly be drawn.

# Warming up

The datastructure is a list of pairs `(age,frequency)`. To get the sum of all frequencies, i.e. the total number of samples,  there are several solutions, all equivalent:

``` python
print sum([pair[1] for pair in distribution])
print sum([freq for (age,freq) in distribution])
print sum(freq for _,freq in distribution)
```

In first line, we traverse every pair of the distribution, take the second element of the pair, and get a list out of the results, that can then be passed to the `sum` function. Very compact. But it can be more elegant, using pattern matching to make it more obvious that we are taking the frequencies out of a list of age/frequency pairs. That is the second line. Let’s push it a step further in the third line. First, the parenthesis are not necessary as the `sum` function accepts _generators_. Second, we'll use the face that `_` is a valid variable name to visually indicate that we are only interested in the second element. That idiom saves us from forging a name for a variable (`age` in that case) whose value is not used.


# Back to the problem

List comprehensions become very productive once you couple them with the various functions available on lists, such as `max`, `sum` etc. For the problem at hand, we need to take the list of age/frequency pairs in the distribution and expand it to a list of ages where each age appears a number of time equal to the given frequency. That is we want the following list:

``` python
[20, 21, 21, 23, 23, 23, 24, 24, 24, 24, 25, 25, 25, 26, 26, 27]
```

Then we just need to make an uniform choice out of that list. To do that will use several features of Python.

Lists can be summed with +, which will simply concatenates them.
Lists can be multiplied by an integer, which simply concatenates several copies of the list.
The `sum` function takes a sequence, an initial element (default to 0) and sums it with all elements in the sequence.
The function `random.choice` returns an element of a sequence with equiprobability.
Try this in your python shell:

``` python
print ['a','b']+['c','d'] # outputs ['a','b','c','d']
print ['a','b']*3         # outputs ['a', 'b', 'a', 'b', 'a', 'b']
print sum([1,2,3],10)     # outputs 16
import random
print random.choice(["good","bad","neutral"])
```

With this, we can write the solution to our problem in one clean and compact line (where the documentation is actually longer than the code itself)!

``` python
import random

def get_value_according_to(distribution):
  """Take a distribution as a sequence of (value,frequency) and return
     a random value with probabilities matching the distribution"""
  return random.choice(sum([[value]*freq for value,freq in distribution],[]))
```
	
# Better algorithm

The problem with the approach above is that it expands the distribution in memory, just to pick one and throw the others! If your distribution is given using figures from a real country, it will not fit in memory. A more efficient approach is to slightly modify our distribution. We will make it a list of pairs whose first element is still an age and second element is the number of samples below or equal to that age in the initial distribution. This can be done in place:

``` python
# Compute the summed frequencies below or equal each value
for i in range(1,len(distribution)):
	distribution[i][1] += distribution[i-1][1]
```
	
Now, we just need to pick an equiprobable integer between 1 and the number of elements in the initial distribution. Since we did the in-place summation, this number of elements is simply obtained from the last element in the (modified) distribution! We then need to advance in the distribution until the summed frequencies becomes larger than this random integer. Let’s reformulate this into _“We need to take the first element amongst all those such that the summed frequencies is bigger than my random number”_. Lists comprehensions can express this immediately, thanks to the filtering operator `if`. The code will be:

``` python
# To get a random value with the initial probability distribution, 
# we pick up a random number between 1 and the initial number of 
# samples, given by the last entry of (modified) distribution.
n = random.randint(1,distribution[-1][1])
# Then we advance through distribution until the summed frequency 
# is higher than n and take the last value considered. 
# This can nicely be written with list comprehensions
print [value for (value,summed_frequency) in distribution if summed_frequency>=n][0]
```

Note that [0] line 6 in the code is safe because the filtered list will necessarily contain one element at least, by choice of n. The code above is still very compact, and I think easy to read. This is why I love list comprehensions, because you can write working code very quickly.

# To go further

## Better implementation and performance analysis
The above implementation performs a linear search in the (modified) distribution. Since the summed frequencies are in increasing order, you can instead perform a binary search. If the distribution contains enough element, you will get a gain. I made several experiments. I post below the final code I came up with and which inspired this post.

``` python
import math, random, sys, time, locale

ages = [     # [age, frequency]
[20,len("*")],
[21,len("**")],
[23,len("***")],
[24,len("****")],
[25,len("***")],
[26,len("**")],
[27,len("*")],
]
# Uncomment this other distribution for performance test (see end of document)
# mu = 50.0
# sigma = 25.0
# ages = [[age,1+int(10000*math.exp(-(age-mu)*(age-mu)/(sigma*sigma)))] 
#         for age in range(1,100)]

#--------------------------------------------
#      Heart of the system
# without comments, only 5 lines of code!
#--------------------------------------------
# We first obtained the summed frequencies below or equal each age
for i in range(1,len(ages)):
	ages[i][1] += ages[i-1][1]

def pick_random_age():
	# To get a random age with the initial probability distribution,
	# we pick up a random number between 1 and the initial number of
	# samples, given by the last entry of ages.
	n = random.randint(1,ages[-1][1])
	# Then we advance through ages until the summed frequency is
	# higher than n and take the last age considered. This can nicely
	# be written with list comprehensions
	return [age for (age,summed_frequency) in ages if summed_frequency>=n][0]

#--------------------------------------------
# The first, naive version uses a linear search
# Let's try to improve with a binary search
def pick_random_age_faster_in_principle():
	n = random.randint(1,ages[-1][1])
	def binary_search(ages):
		if len(ages) == 1: return ages[0][0]
		m = len(ages)/2
		if ages[m][1] < n:
			return binary_search(ages[m:])
		else:
			if ages[m-1][1] < n:
				return ages[m][0]
			else:
				return binary_search(ages[:m])
	return binary_search(ages)

#--------------------------------------------
# The binary_search as implemented above may be slower than the naive approach
# because of list copies that might be hidden behind the splicing operator.
# Let's rewrite the code to actually avoid that by maintaining the slice ourselves
def pick_random_age_faster_in_practice():
	n = random.randint(1,ages[-1][1])
	def binary_search(a,b):
		if a==b: return ages[a][0]
		m = a+(b+1-a)/2
		if ages[m][1] < n:
			return binary_search(m,b)
		else:
			if ages[m-1][1] < n:
				return ages[m][0]
			else:
				return binary_search(a,m-1)
	return binary_search(0,len(ages)-1)

# To test our algoritm we draw a large number of random ages, compute
# the distribution using a simple hash, and prints it nicely to see it
# reflects the inital gaussian distrib.
nb_draws = 10*1000
locale.setlocale(locale.LC_ALL,"en_us")
print "# testing using %s draws" % locale.format("%d",nb_draws,True)
for picking_function in [pick_random_age,
                         pick_random_age_faster_in_principle,
                         pick_random_age_faster_in_practice]:
	print "# -- testing",picking_function.__name__
	distribution = {}
	before = time.time()
	for i in range(1,nb_draws):
		age = picking_function()
		distribution[age] = distribution.get(age,0)+1

	if len(distribution) < 10:
		for age in distribution.keys():
			print "#",age,"*"*((100*distribution[age])/nb_draws)
	print "# ---done in",time.time()-before," seconds"

# Here the output of a typical run
# testing using 10,000 draws
# -- testing pick_random_age
# 20 ******
# 21 ***********
# 23 ******************
# 24 ************************
# 25 *******************
# 26 ************
# 27 ******
# ---done in 0.0495750904083  seconds
# -- testing pick_random_age_faster_in_principle
# 20 *****
# 21 ************
# 23 *******************
# 24 *************************
# 25 ******************
# 26 ************
# 27 ******
# ---done in 0.0656080245972  seconds
# -- testing pick_random_age_faster_in_practice
# 20 *****
# 21 ************
# 23 *******************
# 24 ************************
# 25 ******************
# 26 ************
# 27 ******
# ---done in 0.0588190555573  seconds

# -------------------------------------------------
# Performance Analysis
# -------------------------------------------------
# I made a run of the above examples with 1 million draws
# using two different training distribution (see comments on top).
# - one with 8 ages,
# - one with 100 ages
# Here are the results
#
# testing using 1,000,000 draws
# -- testing pick_random_age
# ---done in 4.91417002678  seconds
# -- testing pick_random_age_faster_in_principle
# ---done in 7.53200507164  seconds
# -- testing pick_random_age_faster_in_practice
# ---done in 7.06282091141  seconds
#
# testing using 1,000,000 draws
# -- testing pick_random_age
# ---done in 14.2727029324  seconds
# -- testing pick_random_age_faster_in_principle
# ---done in 11.9355649948  seconds
# -- testing pick_random_age_faster_in_practice
# ---done in 9.70899605751  seconds
#
#
# This tells us two things:
# - The binary search is worth using (given the more complex code) only if
#   we have a lot of ages in our training distribution
# - The splicing operator is certainly well implemented because the gain
#   by doing it ourselves is not so big.
```
