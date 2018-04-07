# java File 的路径问题

标签（空格分隔）： java File 路径

---
###问题来源
在“高能耗服务系统”的制作过程中，涉及到上传文件的问题时，遇到一个难点。原系统使用swfUpload组件来支持上传的功能。由于这个系统其实并不重视Struts的使用，所以其实并没有走struts2中的上传办法，而是直接让swfUpload组件吧数据传到后端的一个action中，然后再通过action->service->serviceImpl来最终实现上传的代码。在serviceImpl中，有一句代码
```java
File createPath = new File("/upload");//在指定位置创建了一个文件
```
实际上文件的上传位置就是靠这句代码导向。(当然，源代码file的参数是一个实体类的属性，那个实体类从property文件中读取路径位置)
按照上面代码的写法，文件夹将直接出现在我的D盘的根目录下。这就带来了问题，既然相对路径的“根目录”是我的D盘，我怎么改才能让上传的文件出现在tomcat中，而不是与我的D盘扯上关系？
###解决办法
找到了一个被转载了很多次的文章，献上链接
http://blog.sina.com.cn/s/blog_9386f17b0100w2vv.html
File f = new File("D:/test/mytest.txt");//当执行这句话后在内存的栈空间存在一个f的应用，在堆空间里存在一个mytest.txt对象。注意
这个对象只含有文件的属性(如大小，是否可读，修改时间等)，不包含文件的内容，所以length=0。当我们想执行对文件的操作的时候，这个时
候抽象路径起作用了，比如我们想执行f.createNewFile()命令时，虚拟机会将抽象路径转化为实际的物理路径，到这个转化后的物理路径(此时
是硬盘)下进行文件的创建。这时，如果在你的D盘没有test文件夹，那么不好意思啦，程序就会抛异常，如果有test文件夹，就可以创建一个
mytest.txt文件了。能不能创建mytest.txt就在于这个文件上面有没有test文件夹，这也就是抽象路径在装怪了。
如果想让f引用在硬盘中把文件夹也创建出来怎么办？用f.getParentFile()求出mytest.txt上面的所有文件夹，然后在mkdirs()就搞定了。
-----------------------
-----------------------
File类是用来构造文件或文件夹的类,在其构造函数中要求传入一个String类型的参数,用于指示文件所在的路径.以前一直使用绝对路径作为参
数,其实这里也可以使用相对路径.使用绝对路径不用说,很容易就能定位到文件,那么使用了相对路径jvm如何定位文件的呢?
按照jdk Doc上的说法”绝对路径名是完整的路径名，不需要任何其他信息就可以定位自身表示的文件。相反，相对路径名必须使用来自其他路
径名的信息进行解释。默认情况下，java.io 包中的类总是根据当前用户目录来分析相对路径名。此目录由系统属性 user.dir 指定，通常是
Java 虚拟机的调用目录.”
相对路径顾名思义,相对于某个路径,那么究竟相对于什么路径我们必须弄明白.按照上面jdk文档上讲的这个路径是”当前用户目录”也就是”
java虚拟机的调用目录”.更明白的说这个路径其实是我们在哪里调用jvm的路径.举个例子:
假设有一java源文件Example.java在d盘根目录下,该文件不含package信息.我们进入命令行窗口,然后使用”d:”命令切换到d盘根目录下,然后
用”javac Example.java”来编译此文件,编译无错后,会在d盘根目录下自动生成”Example.class”文件.我们在调用”java Example”来运行
该程序.此时我们已经启动了一个jvm,这个jvm是在d盘根目录下被启动的,所以此jvm所加载的程序中File类的相对路径也就是相对这个路径的,即
d盘根目录:D:\.同时” 当前用户目录”也是D:\.在System.getProperty(“user.dir”);系统变量”user.dir”存放的也是这个值.
我们可以多做几次试验,把”Example.class”移动到不同路径下,同时在那些路径下,执行”java Example”命令启动jvm,我们会发现这个”当前
用户目录”是不断变化的,它的路径始终和我们在哪启动jvm的路径是一致的.
搞清了这些,我们可以使用相对路径来创建文件,例如:
File file = new File(“a.txt”);
file.createNewFile();
假设jvm是在”D:\”下启动的,那么a.txt就会生成在D:\a.txt;
此外,这个参数还可以使用一些常用的路径表示方法,例如”.”或”.\”代表当前目录,这个目录也就是jvm启动路径.所以如下代码能得到当前目
录完整路径:
File f = new File(“.”);
String absolutePath = f.getAbsolutePath();
System.out.println(absolutePath);//D:\
最后要说说在eclipse中的情况:
Eclipse中启动jvm都是在项目根路径上启动的.比如有个项目名为blog,其完整路径为:D:\work\IDE\workspace\blog.那么这个路径就是jvm的启
动路径了.所以以上代码如果在eclipse里运行,则输出结果为” D:\work\IDE\workspace\blog.”
Tomcat中的情况.
如果在tomcat中运行web应用,此时,如果我们在某个类中使用如下代码:
File f = new File(“.”);
String absolutePath = f.getAbsolutePath();
System.out.println(absolutePath);
那么输出的将是tomcat下的bin目录.我的机器就是” D:\work\server\jakarta-tomcat-5.0.28\bin\.”,由此可以看出tomcat服务器是在bin目
录下启动jvm的.其实是在bin目录下的” catalina.bat”文件中启动jvm的.

然后，我把代码改成
```java
File createPath = new File("upload/");//在指定位置创建了一个文件
```
果然，文件夹就在tomcat的bin文件夹内生成了。
然后再改成这样
```java
File createPath = new File("../webapps/gnhzb/FixPhotoUpload/");//在指定位置创建了一个文件
```
也就是相对tomcat的bin，找到项目文件夹，大功告成。
###总的来说，记住几点:
* java File（String）构造方法中的路径，如果使用相对路径的话，是从jvm启动的位置开始的


