# Activity四种启动模式及onNewIntent()方法、taskAffinity属性分析

#### 一、前期知识储备-Application，Task和Process的区别与联系

**①Application**，翻译成中文时一般称为“应用”或“应用程序”，在android中，总体来说**一个应用就是一组组件的集合**。众所周知，android是在***\**\*应用层\*\**\***组件化程度非常高的系统，android开发的第一课就是学习android的四大组件。当我们写完了多个组件，并且在manifest文件中注册了这些组件之后，把这些组件和组件使用到的资源打包成apk，我们就可以说完成了一个application。application和组件的关系可以在manifest文件中清晰地体现出来。在app安装时，系统会读取manifest的信息，将所有的组件解析出来，以便在运行时对组件进行实例化和调度。

**②task**，是在程序运行时，只针对activity的概念。说白了，**task是一组相互关联的activity的集合**，它是存在于***\*framework层\****的一个概念，控制界面的跳转和返回。这个task存在于一个称为back stack的数据结构中，也就是说，framework是以栈的形式管理用户开启的activity。这个栈的基本行为是，当用户在多个activity之间跳转时，执行压栈操作，当用户按返回键时，执行出栈操作。举例来说，如果应用程序中存在A,B,C三个activity，当用户在Launcher或Home Screen点击应用程序图标时，启动主Activity A，接着A开启B，B开启C，这时栈中有三个Activity，并且这三个Activity默认在同一个任务（task）中，当用户按返回时，弹出C，栈中只剩A和B，再按返回键，弹出B，栈中只剩A，再继续按返回键，弹出A，任务被移除。

task是可以***\*跨应用\****的，这正是task存在的一个重要原因。有的Activity，虽然不在同一个app中，但为了保持用户操作的连贯性，把他们放在同一个任务中。例如，在我们的应用中的一个Activity A中点击发送邮件，会启动邮件程序的一个Activity B来发送邮件，这两个activity是存在于不同app中的，但是***\**\*被系统放在一个任务中\*\**\***，这样当发送完邮件后，用户按back键返回，可以返回到原来的Activity A中，这样就确保了***\**\*用户体验\*\**\***，感兴趣的读者可以试着用手机发送相册中的图片到微信中，然后按back键，看看会发生什么，也可以在音乐app中分享音乐到朋友圈，然后按back键，看看会发生什么。

**③process**，一般翻译成进程，进程是***\**\*操作系统内核\*\**\***中的一个概念，表示直接受内核调度的执行单位。在应用程序的角度看，我们用java编写的应用程序，运行在dalvik虚拟机中，可以认为一个运行中的dalvik虚拟机实例占有一个进程，所以，在默认情况下，一个应用程序的所有组件运行在同一个进程中。但是这种情况也有例外，即，应用程序中的不同组件可以运行在不同的进程中。只需要在manifest中用**process属性**指定组件所运行的**进程的名字**。

#### 二、活动的四种启动模式

（1）Standard，是活动默认的启动模式，在不进行显式指定的情况下，所有活动都会自动使用这种启动模式。Android中是使用返回栈来管理活动的，在standard模式下（即默认情况下），每当启动一个新的活动，它就会在返回栈中入栈，并处于栈顶的位置。对于使用standard模式的活动，系统不会检查这个活动是否已经在返回栈中存在，每次启动都会创建该活动的一个新的实例。

​                 ![img](https://img-blog.csdn.net/201803251927239?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MTEwMTE3Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

（2）singleTop，使用standard时活动明明已经在栈顶了，为什么新启动的时候还要创建一个新的活动实例呢？实际上我们可以在***\*注册文件\****中自己为活动***\*指定\****其他的启动模式，比如使用singleTop模式。当活动的启动模式指定为singleTop时，在启动时如果发现返回栈的***\*栈顶\****已经是该活动，则认为可以直接使用它，不会在创建新的活动实例。

​             ![img](https://img-blog.csdn.net/20180325192742400?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MTEwMTE3Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

（3）singleTask，使用singleTop模式可以很好地解决重复创建栈顶活动的问题，但是如果该活动并没有处在栈顶的位置，还是可能会创建多个活动实例的。那么有没有什么办法可以让某个活动在整个应用程序的上下文中只存在一个实例呢？这就要借助singleTask模式来实现了。当活动的启动模式指定为singleTask，每次启动该活动时系统会首先在***\*返回栈中\****检查是否存在该活动的实例，***\*如果发现已经存在则直接使用该实例，并把在这个活动之上的所有活动统统出栈\****，如果没有发现就会创建一个新的活动实例。

​                 ![img](https://img-blog.csdn.net/20180325192814696?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MTEwMTE3Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

（4）singleInstance，模式应该算是4中启动模式中最特殊也最复杂的一个。不同于以上三种模式，指定为singleInstance模式的活动会***\*启用一个新的返回栈来\****管理这个活动（其实如果**singleTask模式指定了不同的taskAffinity**，也会启动一个新的返回栈）。那么这样做的意义？想象以下场景：假设我们的程序中有一个活动是允许其他程序调用的，如果我们想实现其他程序和我们的程序可以***\*共享这个活动的实例\****，应该如何操作呢？使用前面三种肯定是做不到的，因为每个应用程序都会有自己的返回栈，同一个活动在不同的返回栈中入栈时必然是创建新的实例。而使用singleInstance模式就可以解决这个问题，在这种模式下会有一个单独的返回栈来管理这个活动，不管是哪个应用程序来访问这个活动，都**共用同一个返回栈（此单独返回栈中只有一个实例）**，也就解决了共享活动实例的问题。

​             ![img](https://img-blog.csdn.net/20180325192850561?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MTEwMTE3Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#### 三、onNewIntent()方法解析

launchMode为**singleTask**的时候，通过Intent启到一个Activity,如果系统已经**存在**一个实例，系统就会将请求发送到这个实例上，但这个时候，系统就不会再调用通常情况下我们处理请求数据的onCreate方法，而是调用**onNewIntent()**方法。

protected void ***\*onNewIntent\****(Intent intent) {

 super.onNewIntent(intent);

 setIntent(intent);//must store the new intent unless getIntent() will return the old one

}；

onNewIntent（）非常好用，Activity第一启动的时候执行onCreate()---->onStart()---->onResume()等后续生命周期函数，也就时说第一次启动Activity并不会执行到onNewIntent(). 而后面如果再有想启动Activity的时候，那就是执行***\*onNewIntent()---->onResart()------>onStart()----->onResume()\****.  如果android系统由于内存不足把已存在Activity释放掉了，那么再次调用的时候会重新启动Activity即执行onCreate()---->onStart()---->onResume()等。

#### 四、taskAffinity属性分析



- ltaskAffinity表示当前activity具有亲和力的一个任务（翻译不是很准确，原句为The task that the activity has an affinity for.），大致可以这样理解，这个 taskAffinity表示一个任务，这个任务就是当前activity所在的任务。
- 在概念上，**具有相同的affinity的activity（即设置了相同taskAffinity属性的activity）属于同一个任务**。
- 默认情况下，**一个应用中的所有activity具有相同的taskAffinity**，即应用程序的包名。我们可以通过设置不同的taskAffinity属性给应用中的activity分组，也可以***把不同的应用中的\**activity的taskAffinity设置成相同的值**。（所以启动模式为singleTask时设置不同activity的affinity属性为相同时，可以实现singleInstance启动模式一样的效果，即启用另外一个返回栈）





转载 : [CSDN - Chin_style](https://blog.csdn.net/weixin_41101173/article/details/79689601)