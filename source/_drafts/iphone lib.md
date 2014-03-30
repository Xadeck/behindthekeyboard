Being a developer for a long time, the notion of reusability is branded into my flesh. When I recently started developing on the iPhone, I wanted to a apply the same principles than the ones I have been using for the last 25 years with various other programming languages and platforms:
<blockquote>Whenever some code exhibits some generality, try to abstract it into a library so that you can reuse it in other projects.</blockquote>
Making libraries avoids code duplication, eases bug fixing and updates. It has a downside. You must understand/master your build tool. And you must care about some good practices such as namespaces, include pathes, interface vs. implementation, etc. that can more easily be overlooked when doing a standalone program. Let's see how to do that for iPhone development, with XCode.

<!--more-->
<h2>A bit of lecturing</h2>
A library that you distribute is made of two parts:
<ul>
	<li>Some header files that describes the interface of your library;</li>
	<li>A binary file that contains the implementation of the functionalities exposed by the interface.</li>
</ul>
The users of your library includes the header in their code so that they can use your classes and functions, and links with the binary so that their program runs. The binary can be either dynamic (<tt>.so</tt>, <tt>.dylib</tt> or <tt>.dll</tt> depending on the platform) or static (<tt>.a</tt>). Dynamic libraries have many advantages. The system can load the most up-to-date version of a library that is compatible with a given executable if several versions are installed (a typical example is <tt>libJPEG</tt>). Also, the system can load the library in memory only once when several programs needs it simultaneously, reducing the memory footprint. Finally, the executable your distribute is smaller, because the dynamic library is either already installed on the system, or distributed separately. The downside is that there are some installation/configuration issues when using them (the mess of <tt>LD_LIBRARY_PATH</tt> on *nix, or <tt>DYLD_LIBRARY_PATH</tt> on Mac OS, or <tt>PATH</tt> on Windows). Static libraries are on the other hand much easier to use since your program contains all the code it needs and do not need a proper system configuration to find the code it depends on. On most systems, using dynamic or static libraries is a project-based choice, depending on various considerations. On iPhone, you can just use static libraries (apparently for security/robustness reasons). So I can stop lecturing you and enter the meat of this post.
<h2>Requirements</h2>
There are various webpages about building static libraries for iPhone. Here is one for example:
<ul>
	<li><a href="http://blog.stormyprods.com/2008/11/using-static-libraries-with-iphone-sdk.html">Building static libraries with the iPhone SDK</a></li>
</ul>
It is only part of the problem. What I need is reusability and code sharing. There is an interesting post on this topic:
<ul>
	<li><a href="http://blog.costan.us/2009/02/iphone-development-and-code-sharing.html">iPhone Development and Code Sharing</a></li>
</ul>
Unfortunately, the proposed solution does not meet my personal requirements. Here they are:
<ul>
	<li>It is too different from the procedure I used with other development environment;</li>
	<li>It does not enforce a clean separation between the library and the program.</li>
</ul>
What I want is to have somewhere on my disk a directory with my library (headers and binary). It can either be installed system-wide by an installer. Or it can be a single directory from a downloaded tarball. Or more conveniently to me, it can be a working copy of an XCode project under <a href="http://en.wikipedia.org/wiki/Source_Code_Management">source control management</a>.

Then somewhere else, I have other XCode projects for my applications. They refer too the library in a simple manner (quick, understandable, in plain sight not under the hood), and when you build, everything works like a charm without scratching your head. Icing on the cake, the XCode projects nicely decide they need rebuild when the headers of the library or the binary are updated. For a finished, distributed, library, it is not that important, but when you are developing the library, it is of primary importance.
<h2>The project</h2>
The project I will use to describe my approach is made of a library named GoWithTheFlow, which offers <a href="http://en.wikipedia.org/wiki/Cover_Flow">CoverFlow</a> effects for iPhone developers. There is already at least one such library, named <a href=" http://apparentlogic.com/openflow">OpenFlow</a>, but it is lacking some functionalities I need. Oh, and of course, I can not use Apple's implementation because it is not in the public API of the iPhone SDK, and altough you can access private API, it is a reason for app rejection during review.

Then there are many applications that use the GoWithTheFlow library. I will describe only one, named Chapelet, which is a program for magicians to revise stacks of cards (what, magicians use deck of cards in memorized order?!?! but I saw him shuffle the cards?!?!).
<h3>The library</h3>
Let's start with the library. Launch XCode and go <tt>File &gt; New project </tt>. Select the Cocoa Touch Static Library as shown below:

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/xcode_new_project_static_library.png" alt="XCode New Project Static Library" />

Name the library, in my case <tt>GoWithTheFlow</tt> and select a location, in my case <tt>/Users/xdecoret/Code/iPhone/GoWithTheFlow</tt>. Then in the project, add the class you need. In my case, I added a new Objective-C class, subclass of <tt>UIVIew</tt>, that I named <tt>CoverFlowView</tt> class, ending up with the project window like the one shown below (you may not have exactly the same view. This is because I configured my XCode to to use the All-in-One layout, see Preferences/General):

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/go_with_the_flow_project.png" alt="Go With the Flow Project" />

Then I added some drawing code to my class. For this post, I added code stolen from the excellent book <a href="http://pragprog.com/titles/amiphd/iphone-sdk-development">iPhone SDK Development</a> at <a href="http://www.pragprog.com">Pragmatic Programmers</a>, which is a publisher I am really fond of. Sorry, I did not put the actual code for CoverFlow ;-). The goal here is just to have a custom view to test in the application.

<a onclick="xcollapse('X6916');return false;" href="#"> Source code to put in <tt>CoverFlowView.m</tt> </a>
<div id="X6916" style="display: none;">
<pre lang="objc">-(CGPathRef) pathInRect:(CGRect)rect {
	CGMutablePathRef path = CGPathCreateMutable();
	CGFloat radius = CGRectGetHeight(rect) / 2.0f;
	CGPathMoveToPoint(path, NULL, CGRectGetMinX(rect), CGRectGetMinY(rect));
	CGPathAddLineToPoint(path, NULL, CGRectGetMaxX(rect) - radius,
                         CGRectGetMinY(rect));
	CGPathAddArc(path, NULL, CGRectGetMaxX(rect) - radius,
                 CGRectGetMinY(rect) + radius,
                 radius, -M_PI / 2.0f, M_PI / 2.0f, NO);
	CGPathAddLineToPoint(path, NULL, CGRectGetMinX(rect), CGRectGetMaxY(rect));
	CGPathCloseSubpath(path);
	CGPathRef imutablePath = CGPathCreateCopy(path);
	CGPathRelease(path);
	return imutablePath;
}

- (void)drawRect:(CGRect)rect {
	CGSize size = self.bounds.size;
	CGFloat width1 = size.width * 0.75f;
	CGFloat width2 = size.width * 0.35f;
	CGFloat width3 = size.width * 0.55f;
	CGFloat height = size.height * 0.2f;
	CGRect one = CGRectMake(0.0f, height, width1, height - 10.0f);
	CGRect oneText = CGRectMake(10.0f, height + 25.0f, width1, height - 30.0f);
	CGRect two = CGRectMake(0.0f, 2.0 * (height + 5.0f),width2, height - 10.0f);
	CGRect twoText = CGRectMake(10.0f, 2.0 * height + 30.0f,width2, height - 30.0f);
	CGRect three = CGRectMake(0.0f, 3.0 * (height + 5.0f),width3, height - 10.0f);
	CGRect threeText = CGRectMake(10.0f, 3.0 * height + 35.0f,width3, height - 30.0f);
	CGContextRef ctx = UIGraphicsGetCurrentContext();
	[[UIColor blueColor] setFill];
	CGPathRef pathOne = [self pathInRect:one];
	CGContextAddPath(ctx, pathOne);
	CGPathRelease(pathOne);
	CGContextFillPath(ctx);
	[[UIColor blackColor] setFill];
	[@"One" drawInRect:oneText withFont:[UIFont systemFontOfSize:34]];
	[[UIColor redColor] setFill];
	CGPathRef pathTwo = [self pathInRect:two];
	CGContextAddPath(ctx, pathTwo);
	CGPathRelease(pathTwo);
	CGContextFillPath(ctx);
	[[UIColor blackColor] setFill];
	[@"Two" drawInRect:twoText withFont:[UIFont systemFontOfSize:34]];
	[[UIColor yellowColor] setFill];
	CGPathRef pathThree = [self pathInRect:three];
	CGContextAddPath(ctx, pathThree);
	CGPathRelease(pathThree);
	CGContextFillPath(ctx);
	[[UIColor blackColor] setFill];
	[@"Three" drawInRect:threeText withFont:[UIFont systemFontOfSize:34]];
}</pre>
</div>
Then I build the library, and everything went fine. I built the library both in debug, and in release, for both the iPhone Simulator, and my iPhone device. As a result, I have the following layout on disk (where I ignored the byproducts of build):
<pre>Xavier-Decorets-MacBook-Pro:GoWithTheFlow xdecoret$tree -I GoWithTheFlow.build
.
|-- CoverFlowView.h
|-- CoverFlowView.m
|-- GoWithTheFlow.xcodeproj
|   |-- project.pbxproj
|   |-- xdecoret.pbxuser
|   `-- xdecoret.perspectivev3
|-- GoWithTheFlow_Prefix.pch
`-- build
    |-- Debug-iphonesimulator
    |   `-- libGoWithTheFlow.a
    |-- Release-iphoneos
    |   `-- libGoWithTheFlow.a
    `-- Release-iphonesimulator
        `-- libGoWithTheFlow.a

5 directories, 9 files</pre>
Ok, I have my header and my binaries for my library. Now let's make an application that uses it.
<h3>The application</h3>
Create another project in XCode (you can keep the library one opened). Select an application template. For this post, I use a simple view based application.

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/xcode_new_project_application.png" alt="Xcode new Project Application" />

Compile and run, you should get an empty grey view in the iPhone simulator. We are going to replace that with our own view, from the library.
<h2>Personal approach, promising but...</h2>
In most IDEs and build system, using a library is just a matter of integrating its two components. You add the path to the headers to the include path, and you link with the binary. Let's do that in XCode. Configuring the include path is simple. In  the project view, right-click the application and choose 'Get Info', or press Command-I. You can also double click the project.

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/xcode_application_get_info.png" alt="XCode Application Get Info" />

This opens a property window. Select the 'build' tab. Then look for for the 'Search Paths' section, and in it, for the 'Header Search Paths' entry. You can use the search box to help you. Enter the path to your library project. Be sure to make the change for 'All configurations'. The snapshot below emphasizes the important elements.

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/xcode_application_build_info.png" alt="XCode Application Build Info" />

Now for the library. Right-click again the project, choose 'Add', then 'Existing Frameworks'. In the dialog box, click on 'Add other...'. In the file dialog, browse to the file <tt>Debug-iphonesimulator/libGoWithTheFlow.a</tt> and click 'Add'. You project window should look like this:

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/xcode_library_added.png" alt="XCode library added" />.

Finally, go to the <tt>ChapeletViewController.m</tt> file, and add the following code
<pre lang="objc">#import "ChapeletViewController.h"
#import                 // added by you

@implementation ChapeletViewController
// ...
// ...
- (void)viewDidLoad {
	[super viewDidLoad];
	CoverFlowView *v = [[CoverFlowView alloc] initWithFrame:self.view.frame];
	[self.view addSubview:v];
	[v release];
}
// ...
// ...</pre>
Compile and run in the iPhone simulator. This should work and give you a view like this:

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/i_phone_simulator.png" alt="I Phone Simulator" width="100" />

So far so good, did I think at first. But reality is much bitter. This approach has the advantage that it is coherent with what you would do in other build systems or IDEs. Except for one very important thing: you cannot link with libraries depending on the build configuration. What this means is that we added the <tt>Debug-iphonesimulator/libGoWithTheFlow.a</tt> to all configurations. So when we build in Release, we will still link with the debug version of our library. Worst, when we build for the iPhone device and not the simulator, we will still be linking with the debug <em>for the simulator</em>. So it won't work. Well, XCode does not seem quite elegant on this. And we are <a href="http://www.evilexperiments.com/evilog/wp-trackback.php?p=7">at least two</a> to agree on this. There is another problem with this approach: it does not integrate well with Interface Builder. In the example below, I added my <tt>CoverFlowView</tt> programmatically. But the truth is that I wanted to simply open the <tt>ChapeletViewController.xib</tt> and change its view type to <tt>CoverFlowView</tt>, as I normally do with custom views <em>defined in the project</em>. But it does not work. After a few moment of frustration, I found <a href='http://stackoverflow.com/questions/1058766/interface-builder-cant-see-classes-in-a-static-library'>a solution on StackOverflow</a>. But once again, it is not elegant.

So we need another approach. <a href="http://www.evilexperiments.com/">Evilog</a>, whose post I mentionned earlier, did the search-fu for me and pointed me to <a href="http://lists.apple.com/archives/xcode-users/2005/Jul/msg00439.html">the Apple discussion</a> that explains how to do it the 'proper' way.
<h2>Apple approach, working (but...)</h2>
Delete your Chapelet project and create a fresh new one. First, we are going to add the 'GoWithTheFlow' project to the 'Chapelet' project. Right-click on the 'Chapelet' project, select 'Add' and 'Existing Files'. Now browse to the <tt>GoWithTheFlow.xcodeproh</tt> file, and click 'Add'. In the dialog box that pops up, be sure to <strong>uncheck</strong> the 'Copy items...' box.

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/add_files_options.png" alt="Add Files Options" />

Next step is to link with the library, and add a dependency so that if the GoWithTheFlow changes, Chapelet gets recompiled and re-linked. Expand your project view so that it looks like the first image below. Then drag and drop the libGoWithTheFlow.a file to the 'Link Binary with...' folder of the Chapelet target. Then double-click the Chapelet target and in the 'Info' window that opens, go to the 'General' tab and add a dependency to GoWithTheFlow. The second window shows what you should have in the end.

<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/chapelet_project_before.png" alt="Chapelet project before" />
<img src="http://xdecoret.free.fr/speaking_of_code/wp-content/uploads/2010/01/chapelet_project_after.png" alt="Chapelet project after" />

Now edit <tt>ChapeletViewController.m</tt> as before and try to compile. It does not work. To my surprise, adding the project does not add its classes and pathes. So do like in previous approach, add the project path to the search path. Compile and run as before.

Finally, we are almost like in the previous approach, except that now, we have gain the multiple configurations link, and the dependency rebuild. But we also have lost on some points. First, we need the XCode project of GoWithTheFlow, which may not work with the distribution model you have. Second, we had to do many steps manually, and it cannot be factored using something like property sheets in Visual Studio. Finally, the Interface Builder is still not solved. So in the end, it does not look like a valid approach.
<h2>Other experimentations</h2>
I have tried to add the <tt>CoverFlowView.h</tt> to the project. It works weirdly. It compiles without needing to change the 'search paths'. The CoverFlowView is now known from Interface Builder, but it does not link with the correct code?!?!.

Another approach is to add the <tt>.h</tt> and <tt>.m</tt> to the application project. This works fine, and solves the Interface Builder problem, but this violates the fact that you may not want to distribute the source of your implementation.
<h2>Conclusion</h2>
So in the end, I do not have a 'good' solution. I decided to use the Apple approach when developing my library. When the code is clean, I switch to the Personal approach, only using a <tt>Simulator/Release</tt> build of the library, and changing manually to a <tt>Device/Release</tt> once the application is finished and ready to be distributed on a device.

Then, for the Interface Builder, I decided to investigate the plugin approach. I will tell you more about my investigations in the future. For the moment, I can simply conclude that compared to a tool like Qt's designer, I found Interface Builder less usable.

<h2>New conclusion</h2>

I recently found <a href='http://blog.stormyprods.com/2008/11/using-static-libraries-with-iphone-sdk.html'>this post</a> which offers complementary insights and links to tools that you might use to get around.