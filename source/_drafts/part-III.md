This post continues the series on list comprehensions in Python. Be sure to read the <a href="http://xdecoret.free.fr/speaking_of_code/?p=88">previous post</a> on the subject. We will build on it to pretty print 2D diagrams in text mode.

<!--more-->
Here is a practical problem I have been often confronted with when doing data analyis. How do you print a 2D graph of your data?
Let's start with some input data that we generate, as a list of pair (x,y). We want to plot y as a function of x.

[sourcecode language="python"]
import random,math

mu = 15.0
sigma = 6.0
data = [[x,1+int(20*math.exp(-(x-mu)*(x-mu)/(sigma*sigma)))] for x in range(1,30)]
[/sourcecode]

The problem is that printing on screen is done line by line. For a long period of time, I lazily used the following code:

[sourcecode language="python"]
print &quot;\n&quot;.join([&quot;*&quot;*y for _,y in data])
[/sourcecode]

Note how list comprehensions along with the ability to multiply a string by a number allows for a very concise expression. The results looks like this:
<pre>
*
*
*
*
**
***
****
******
********
**********
*************
****************
******************
********************
*********************
********************
******************
****************
*************
**********
********
******
****
***
**
*
*
*
*
</pre>
Ok, we see the gaussian distribution. But you cannot show that to a non programmer and ask him to turn his head to see the result!
What I did back in the days was to make a little function to print the same graph vertically. The code is not complex but not immediate to write. When I started the posts on list comprehensions, I realized that it can solve the problem very simply. The idea is to make lines that are made of stars <b>and</b> are padded by white spaces, so they all have the same length. Then you zip them using a reduction to concatenate the results. Here is the code:
[sourcecode language="python"]
# We need to get the height to padd lines with whitespace	
h = max([y for _,y in data])
# We get the horizontal padded lines
horizontal = [&quot; &quot;*(h-y)+&quot;x&quot;*frequency for _,y in data]
# We reverse them
vertical   = reduce(lambda lines,line: [a+&quot; &quot;+b for a,b in zip(lines,line)],horizontal)
# We print using a join
print &quot;\n&quot;.join(vertical)
[/sourcecode]
And here is the result!
<pre>
	              *              
	             ***             
	             ***             
	            *****            
	            *****            
	           *******           
	           *******           
	           *******           
	          *********          
	          *********          
	          *********          
	         ***********         
	         ***********         
	        *************        
	        *************        
	       ***************       
	       ***************       
	      *****************      
	     *******************     
	    *********************    
	*****************************
</pre>
Ain't it cool? It is just three lines of Python, and it could even be merged into two by inlining the expression for <tt>horizontal</tt> (we do not need an intermediate variable, it is just to make the code easier to read) and the join. Hope you enjoyed!

BTW, you may have the feeling that the vertical graph is incorrect, as it seems that there are some spikes. It is an optical illusion related to the problem of <a href="http://en.wikipedia.org/wiki/Mach_bands">mach banding</a> where your eyes perceive discontinuities in an exaggerated manner.

Here is the complete code if you want to give it a try.
[sourcecode language="python"]
import random,math

mu = 15.0
sigma = 6.0
data = [[x,1+int(20*math.exp(-(x-mu)*(x-mu)/(sigma*sigma)))] for x in range(1,30)]

print &quot;\n&quot;.join([&quot;*&quot;*y for _,y in data])

# We need to get the height to padd lines with whitespace	
h = max([y for _,y in data])
# We get the horizontal padded lines
horizontal = [" "*(h-y)+"*"*y for _,y in data]
# We reverse them
vertical   = reduce(lambda lines,line: [a+b for a,b in zip(lines,line)],horizontal)
# We print using a join
print "\n".join(vertical)
[/sourcecode]