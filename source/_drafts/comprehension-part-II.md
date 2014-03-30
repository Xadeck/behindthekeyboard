Now that you have read about <a href="http://xdecoret.free.fr/speaking_of_code/?p=73">List comprehensions in Python</a> , let's dig into more advanced stuff.

<!--more-->
With list comprehensions, you can get a list of elements from another list by applying a transformation to those elements that match a given condition. In the example below, we draw a list of random integers, and then get the list of squares of the odd ones:

[sourcecode language="python"]
import random
# Construct a list of 50 random numbers in [1,100]
l = [random.randint(1,100) for _ in range(50)]
# Get the square of even numbers in l
s = [x*x for x in l if x%2 == 0]

print(s)
[/sourcecode]

The limitation at that point is that you cannot perform operations involving successive elements in the initial list. For that, you need to combine list comprehensions with two functions <tt>zip</tt> and <tt>reduce</tt> and another cool feature of Python, namely lambda functions.

<h2>The warm-up</h2>

<h3>Lambda functions</h3>

Lambda function is a very important feature of functional programming (such as Caml, Haskell, etc). You have limited support for it in Python. For the moment, let's just consider that it is the way to define a locally, unamed function, and that this is very useful to write one-line expressions. Here is a contrived example, where we sort a list of words according to the alphabetical order <b>but ignoring the first character</b>.

[sourcecode language="python"]
words = [&quot;ham&quot;,&quot;cheese&quot;,&quot;tomato&quot;,&quot;salad&quot;,&quot;oignon&quot;]

words.sort(lambda u,v: cmp(u[1:],v[1:]))
print words
[/sourcecode]
By the way, this example uses another very cool feature, named <em>splicing</em>, which provides a very simple syntax to get parts of a list, starting from the beginning or end. Google to learn more about it.

<h3>Reducing a list</h3>

Mathematically, a reduction constructs a single value from a list by recursively applying a function on the two first elements, replacing them with the result. So if f(a,a') is a function and [a1...an] is a list, what you get is f(f(f(a0,a1),a2)...,an). The <tt>reduce</tt> function in Python takes a function and a list and returns the reduction. To sum up the elements of a list, you can then do

[sourcecode language="python"]
# The following prints 21
print reduce(lambda x,y : x+y,[1,2,3,4,5,6]) 
# Of course, this can achieved directly with the sum builtin function
print sum([1,2,3,4,5,6])
[/sourcecode]

<h3>Zipping two lists</h3>

Nothing describes this process better than the documentation of the function. So I will just talk about two very useful tips when programming in Python: the <tt>dir()</tt> and <tt>help()</tt> functions. The first will list all the functions/modules available in a module or an object. So if you want to find all the methods available for a string, start a python shell in your terminal, and call <tt>dir()</tt> on a string object.
<pre>
>>> dir("")
['__add__', '__class__', '__contains__', '__delattr__',
 '__doc__', '__eq__', '__format__'
.....
.....
 'splitlines', 'startswith', 'strip', 
'swapcase', 'title', 'translate', 'upper', 'zfill']
</pre>
Or if you want to get all the builtins functions:
<pre>
>>> dir(__builtins__)
['ArithmeticError', 'AssertionError', 'AttributeError',
....
...
 'sum', 'super', 'tuple', 'type',
 'unichr', 'unicode', 'vars', 'xrange', 'zip']	
</pre>
From there you spot the function you are looking for and now use the help function to display the string that the programmer (normally) puts in the doc section of the function.
<pre>
Help on built-in function zip in module __builtin__:

zip(...)
    zip(seq1 [, seq2 [...]]) -> [(seq1[0], seq2[0] ...), (...)]

    Return a list of tuples, where each tuple contains the i-th element
    from each of the argument sequences.  The returned list is truncated
    in length to the length of the shortest argument sequence.
</pre>	
I use this a lot when I have not programmed Python for a while and I need to use it for a short task. I do not necessarily remember the <em>exact</em> name of a function, or the parameters it accepts. I always open up a terminal, use dir and help, and try some simple example to get back in shape my python-fu.

<h2>The show</h2>
	
So what can we do with list comprehensions, reductions and zipping? Suppose that you have a list of integers (let's assumed it is sorted)	 and you want to know what is the minimum/maximum discrepancy in it. The discrepancy is the gap between two successive elements. An application of that is to check if you have two consecutive elements: a sufficient and necessary condition is that the minimum discrepancy be 1. Here is the code:
	
[sourcecode language="python"]
# Construct a list of 20 random numbers in [1,100]
l = [random.randint(1,100) for _ in range(20)]
l.sort()

if min([b-a for a,b in zip(l,l[1:])]) == 1:
	print &quot;You have consecutive elements in your random list&quot;
[/sourcecode]
Yes, you can find if you have two successive elements in a sorted list with such a simple code! This is why I am such a fan of <a href="http://www.python.org/dev/peps/pep-0020/">The Zen of Python</a>!

Now for more impressive stuff. I was recently asked for a job interview to solve the following problem. Given two words, check if the second one can be obtained by removing an arbitrary number of letters from the first. Stated otherwise, check if the second one is an abbreviation of the first one. This is tricky because a letter in the second word can appear none, one, or several times in the first word.

So let's try to find a human formulation of the algorithm. I came up with "for each letter in the abbreviation, find its index in the first word, <em>starting at the index found for the previous letter</em>". If all indices can be found, then we indeed have an abbreviation!

It happens that we can write the above formulation using reductions. Wrapping in a try/except block to test success, we end up with the following function:
[sourcecode language="python"]
def is_abbreviation(abbreviation,string):
	&quot;&quot;&quot;This function tests if 'abbreviation' can be obtained from 'string'
by removing an arbitrary number of characters&quot;&quot;&quot;
	try:
		reduce(lambda i,c: string.index(c,i+1),[0]+[c for c in abbreviation])
		return True
	except:
		return False
[/sourcecode]
By the way, the triple quoted (so it can span several lines) string right after the signature of the function is the documentation for the function. It will be printed when you call the <tt>help()</tt> I talked about earlier. Be kind to your fellow programmers, write this string everytime you write a function. Besides, by first stating what your function should do in english, odds are that you will write a better implementation.

The trick I used here is to use reductions to maintain a "state", that is the result of the reduction so far. My state here is simply the index of the previous character found. In other cases, it may be more complex, typically a tuple. The problem is then than the type of this state is not homogenous with the type of the elements in the list. So we need to prepend the list with an initial state. This works because lists in Python can be heterogenous (they are weakly typed compared to declarative languages).