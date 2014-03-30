<p>For an iPhone application I developped, I needed to produce images for buttons with the Apple-ish effect of text emboss and color gradient/shadow. There are many tutorials on how to do that using your favorite image editor (Photoshop, Gimp for the courageaous, or the excellent <a href='http://www.pixelmator.com'>Pixelmator</a> for the lucky Mac owners). Here are a few ones for example:</p>
<ul>
	<li><a href='http://cocoawithlove.com/2008/09/drawing-gloss-gradients-in-coregraphics.html'>Drawing gloss gradients in CoreGraphics</a></li>
	<li><a href='http://www.keepthewebweird.com/iphone-style-icon-tutorial/'>iPhone Style Icon Tutorial</li>
	<li><a href='http://aceinfowayindia.com/blog/2009/09/how-to-create-i-phone-like-button-in-photoshop/'>How to Create I-Phone Like Button in Photoshop</a>
</ul>
<p>Problem is: I needed a programmatical way. Here is why, and how.</p>

<!--more-->
<p>In my case, I could not rely on graphical tools, because of several reasons. First, I did not own a Photoshop license and, at the time, I did not know I could get something like Pixelmator for a ridiculous 49$. There are solutions for that particular problem, but I always encourage people to pay for the software they need, so that they can get good software. Since you should always do what you preach, I would not get a cracked version of Photoshop. More seriously, the main problem was automation:</p>
<ol>
	<li>On the iPhone, you cannot install new fonts. My application required to use a special font (the one that matches indexes on Poker cards), to display letters and numbers on buttons. I could not make a button background once and change its text label. The only solution was to &quot;burn&quot; the text label in the image itself. Thus, I had to produce many different buttons.</li>
	<li>This was made worse by the fact that for each button I needed to produce a normal and a highlighted version, doubling the amount of images to produce.</li>
	<li>To make it a bit more complex, I needed to internationalize the application, which means potentially regenerating some of the buttons (the one with letters).</li> 
	<li>Finally, I needed to make the images, then play a bit with the application that used them, and iterate to change the size of the button to get a good balance between button layout (the application looks like a calculator) and usability (the bigger the button, the easier it is to touch them on screen); so it was highly probable that I had to regenerate the image several times with various dimensions.</li>
</ol>
<p>Although graphical tools offer support for automation, I do not find it powerful enough. In the end, it was clear to me that I needed a programmatical way to produce such buttons. Here is the solution I came up with. It is based on the excellent <a href='http://www.imagemagick.org'>ImageMagick</a> tool. ImageMagick is a command line tool to manipulate images (it is also a library to do the same from your C++/Python/Php etc. code). If you have a linux box, chances are that it is already installed. If you have a Mac, you can install it with macports (it is simple to do but the recompilation takes some time). If you are on Windows, well, you do not care about user interface, do you?</p>

<p>So here is the image we are going to produce:</p>

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/jack_button_without_glow.png" alt="Jack Button without Glow" height="60" width="60" class='aligncenter'/>

<p>This image contains many graphical effect. They might not be exactly the effects Apple applies on its buttons, but it is visually close. If you figure out better parameters, please let me know and I will update this post:</p>
<ul>
	<li>the background has a subtle gradient;</li>
	<li>the buttons have rounded corners;</li>
	<li>the buttons have an emboss that gives a lightweight relief feeling;</li>
	<li>the font are slightly engraved;</li>
	<li>and of course, everything should be nicely antialiased.</li>				
</ul>
<p>The image is a 60x60 one with transparency, where the button occupies a 50x50 area in the center. The reason for that is that in my application, I also make a version of this button with a 'selected' border around it. We will detail how each effect is achieved using ImageMagick, and in the end, we will gather all pieces together in a parametrizable script.</p>

<h2>The detailed approach</h2>

<h3>Background with a gradient</h3>

<p>The 50x50 background is made of a gradient with 4 steps, varying from light gray to darker gray slowly, then rapidly to an even darker gray, then slowly again to black. With ImageMagick, we simply create three images of 50x22, 50x6, 50x22 and stack them vertically.The result is saved in <tt>background.png</tt>.

[sourcecode language="bash"]
convert \
	-size 50x22 gradient:\#666666-\#444444 \
	-size 50x6 gradient:\#444444-\#222222 \
	-size 50x22 gradient:\#222222-\#000000 -append \
	background.png
[/sourcecode]

<h3>Rounding corners</h3>

<p>This technique for getting proper rounded corners with antialising is the one explained on the ImageMagick website.</p>

[sourcecode language="bash"]
convert background.png\
    \( +clone  -threshold -1 \
       -draw &quot;fill black polygon 0,0 0,5 5,0 fill white circle 5,5 5,0&quot; \
       \( +clone -flip \) -compose Multiply -composite \
       \( +clone -flop \) -compose Multiply -composite \
   	\) \
	+matte -compose CopyOpacity -composite \
	background-rounded.png
[/sourcecode]

<h3>Adding the transparent area around</h3>

<p>We simply create a transparent 60x60 and composite the precedent image on it, centered.</p>

[sourcecode language="bash"]
convert \
	-size 60x60 xc:transparent \
	background-rounded.png \
	-gravity center -compose src-over -composite \
	background-rounded-enlarged.png
[/sourcecode]

<h3>Adding a small bevel</h3>

<p>To give a little bit of relief, we use the <tt>-shade</tt> function, which casts a shadow on a heightfield image.  So we clone the image, get only its alpha component to get an image with 255 where the button is and 0 around (transparent). We then shade it with a light source coming from underneath, and we composite this shading with the initial image.</p>

[sourcecode language="bash"]
convert \
	background-rounded.png \
	\( -clone 0 -alpha extract -shade  90x0 -alpha on +level 0%,80% \)\
	-compose  lighten -composite \
	background-rounded-shaded.png
[/sourcecode]

<h3>Drawing some text</h3>

<p>We draw the text "J" in a 25x25 box (ImageMagick automatically chooses the font size so that the text fits in the given rectangle), passing the truetype font we want to use (it is also possible to give a standard font name, see the manual for that). Here I used a font found on the web after a hard day of googling for Poker font. Unfortunately, I cannot find the link again. We use <tt>-shadow</tt> technique described in ImageMagick manual again to give the text an emboss effect.</p>
	
[sourcecode language="bash"]
convert \
	-background none -stroke none -fill white -font CARDC___.TTF -size 25x25 -gravity center label:&quot;J&quot; \
	\( +clone -background none -shadow 100x0+0-2 \) \
	+swap -layers merge +repage \
	text.png
[/sourcecode]

<h3>Combining text and background</h3>

<p>That the easiest part.The only twist is that I adjust a little bit the text because I found the baseline was as I wanted, so I shift it 3 pixels down.</p>

[sourcecode language="bash"]
convert \
	background-rounded-shaded.png\
	text.png -geometry +0-3 \
	-gravity center -composite \
	result.png
[/sourcecode]

<h2>The condensed approach</h2>

<p>All the scripts pieces given above could (and should) be put in a <tt>script.sh</tt> and run from the command line. The problem is that it produces many intermediary files, that are necessary only for teaching or debugging. ImageMagick can actually manipulate a stack of images. So we can combine all the instructions above in one. Of course, it is a long command line, so I actually put it in a script file, using line breaks and indentation to make it easier to maintain.</p>
[sourcecode language="bash"]
convert \
	-size 50x22 gradient:\#666666-\#444444 \
	-size 50x6 gradient:\#444444-\#222222 \
	-size 50x22 gradient:\#222222-\#000000 -append \
    \( +clone  -threshold -1 \
       -draw &quot;fill black polygon 0,0 0,5 5,0 fill white circle 5,5 5,0&quot; \
       \( +clone -flip \) -compose Multiply -composite \
       \( +clone -flop \) -compose Multiply -composite \
   	\) +matte -compose CopyOpacity -composite \
	-size 60x60 xc:transparent \
	+swap\
	-gravity center -compose src-over -composite \
	\( -clone 0 -alpha extract -shade  90x0 -alpha on +level 0%,80% \)\
	-compose  lighten -composite \
	\(\
		-background none -stroke none -fill white -font CARDC___.TTF -size 25x25 -gravity center label:&quot;J&quot; \
		\( +clone -background none -shadow 100x0+0-2 \) \
		+swap -layers merge +repage \
	\)\
	-geometry +0-3 \
	-gravity center -composite \
	result.png
[/sourcecode]

<p>The last step is to add a little bit of shell scripting, with variables, to make the script simply configurable. My actual script is much more complex than that, but this version should be a good start to derive your own.</p>
[sourcecode language="bash"]
#!/bin/sh

# Configure your button below
outer_size=60 # size of the image
inner_size=50 # size occupied by button inside image
middle_size=6 # the height of the gradient in the middle of background
text_size=25  # size of text inside the button
roundness=5   # size of round corners
text=&quot;J&quot;      # text to write in button

gradient0=\#666666 # gradient top color 
gradient1=\#444444 # gradient middle top color 
gradient2=\#222222 # gradient middle bottom color
gradient3=\#000000 # gradient bottom color

#---------------------------------------
# DO NOT EDIT BELOW UNLESS YOU KNOW
#---------------------------------------
topbottom_size=`expr \( ${inner_size} - ${middle_size} \) / 2`

convert \
	-size ${inner_size}x${topbottom_size} gradient:${gradient0}-${gradient1} \
	-size ${inner_size}x${middle_size} gradient:${gradient1}-${gradient2} \
	-size ${inner_size}x${topbottom_size} gradient:${gradient2}-${gradient3} -append \
    \( +clone  -threshold -1 \
       -draw &quot;fill black polygon 0,0 0,$roundness $roundness,0 fill white circle $roundness,$roundness $roundness,0&quot; \
       \( +clone -flip \) -compose Multiply -composite \
       \( +clone -flop \) -compose Multiply -composite \
   	\) +matte -compose CopyOpacity -composite \
	-size ${outer_size}x${outer_size} xc:transparent \
	+swap\
	-gravity center -compose src-over -composite \
	\( -clone 0 -alpha extract -shade  90x0 -alpha on +level 0%,80% \)\
	-compose  lighten -composite \
	\(\
		-background none -stroke none -fill white -font CARDC___.TTF -size ${text_size}x${text_size} -gravity center label:&quot;${text}&quot; \
		\( +clone -background none -shadow 100x0+0-2 \) \
		+swap -layers merge +repage \
	\)\
	-geometry +0-3 \
	-gravity center -composite \
	result.png
[/sourcecode]