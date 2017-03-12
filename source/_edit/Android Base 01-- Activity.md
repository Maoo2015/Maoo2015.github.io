# ［推翻过往］重新夯实Android基础－－Activity



## Activity生命周期

Android知识最基础的就是Activity的生命周期，几乎没有人说不出来：

* onCreate：

  Activity在创建中的时候回调，个人觉得也可以叫onCreating，当然这个时候Acvctivity对象已经存在了。这是我们可对Activity生命周期控制的第一个方法，通常在它里面做一些UI初始化、资源生成、数据获取等操作。

* onRestart:

  前提是Activity没有被销毁，只是不可见的情况下（如被用户Home了，或者被切换到其他Activity了），重新回到这个Activity时，会调用onRestart，然后调用onStart。

* onStart:

  每次Activity从不可见到可见，或者重新创建后，都会调用。这个时候Activity已经被创建完成，但是还没显示，正在启动。

* onResume:

  onStart的时候Activity还没有显示，而onResume的时候它就可以显示出来了。Activity启动后调用，每一次Activity从后台回到前台都会调用。

* onPause:

  Activity在被切换到后台时调用，onPause中的操作不宜耗时，因为只有当前Activity的onPause执行过后，新Activity的onResume才执行。

* onStop:

  Activity停止中，这个执行完Activity就真停了，可以做一些轻微的资源释放。若Activity完全被遮挡，即不可见了的话，onPause之后会执行onStop，如果没有，就只是暂停到后台。

* onDestroy:

  Actvity对象和资源即将被销毁，销毁了就只能重建。可以做重量级资源销释放。

一般我们说的Activity生命周期就是这7个，并且还会配上一个著名的Google的生命周期图：

![activity](http://android-doc.com/images/activity_lifecycle.png)

上面的流程是在一切顺利的情况下，Activity应该有的表现。但是在实际情况（还有面试中），经常会出现各种异常状况，一切不会那么一帆风顺。而对生命周期有影响的异常情况主要是这种：**Activity被杀死并重新创建**。

我认为Actvity被系统重新创建主要出现在这两种情况下：

**1.旋转屏幕，系统配置发生变化**

**2.内存紧张，Activity进程被杀**

这两种情况，其实都会导致一种结果：就是系统会调用onSaveInstanceState来保存当前Activity的状态，然后在重新创建的Activity中把onSaveInstanceState保存的Bundle同时传给onRestoreInstanceState和onCreate方法。屏幕旋转这种情况很容易出现，因此系统知道在配置发生变化后就重建Activity并调用这两个方法。但是Activity被杀这种事情是不可预料的，系统怎么判断一个Activity什么时候会被杀呢？实际上，只要当前Activity不是主动finish，而是进入后台，那么就会调用onSaveInstanceState。总的来说就是下列情况：

1、当用户按下HOME键到Launcher时
2、长按HOME键，选择运行其他的程序时
3、按下电源按键（关闭屏幕显示）时
4、从activity A中启动一个新的activity但没有finish时

这里需要注意，虽然以上情况下onSaveInstanceState会调用并保存数据，但是onRestoreInstanceState则不一定会被调用。onRestoreInstanceState只有在这个Activity后来真的被销毁了，才会在新Activity中调用。这样的设计也是非常合理的。onSaveInstanceState调用在onStop之前，但是它和onPause没有一定的时序关系。而onRestoreInstanceState在onStart之后，同样与onResume没有一定时序关系。



那么Activity最基本的生命周期就复习结束了。这里就联想到一个常见的面试问题：请问一个Activity中打开另外一个Activity再返回的全部生命周期是啥样的？

那么就是这样的：A.onCreate()---->A.onStart()---->A.onResume()---->A.onPause()---->B.onCreate()---->B.onStart()---->B.onResume()---->A.onStop()---->B.onPause()---->A.onReStart()---->A.onStart()---->A.onResume()---->B.onStop()---->B.onDestory()

## Activity启动模式



## Activity启动过程

既然是要推翻过往，就应该深入一些。

首先，打开手机，启动系统，我们来到Launcher的界面上。

然后，我们点击了App的图标。这个时候（关键的来了），Launcher会调用startActivity方法，然后层层调用，最终调用它里面一个Instrumentation引用的execStartActivity来启动应用。Instrumentation似乎就是说这个是个管理器，起了个名字叫仪表盘，如果是打开状态，那么就可以用它来监视所有的系统和应用的交互状态。最后还可以在manifest里写这个标签！？（什么鬼）

总之，在调用了instrumentation的这个executeStartActivity方法后，还没完，在这个方法里instrumentation还要调用它里面一个ActivityManagerProxy对象的startActivity方法启动Activity。这个ActivityManagerProxy对象是ActivityManagerService在应用进程的一个代理对象（详见代理模式），ActivityManagerProxy最终调用ActivityManagerService（运行在system_server进程）的startActvity方法。执行这个方法的时候，它会先尝试启动这个应用的入口Activity，如果发现这个应用进程还未启动，则会进行下一步：通知Zygote进程孵化应用进程。

zygote进程将ZygoteInit作为启动类，会执行它的main方法，先注册ZygoteSocket，然后调用runSelectLoop方法，runSelectLoop方法会调用方法在ZygoteSocket上监听请求，如果别的进程通过ZygoteSocket请求孵化进程，则孵化进程。这里孵化的进程，就是Dalvik
孵化进程的时候调用了一个方法叫forkAndSpecialize，它会调用nativeForkAndSpecialize方法孵化进程，这是一个本地cpp方法，使用Linux的fork系统调用去创建进程，因为这样效率较高。

那么Zygote进程孵化出新进程后，会执行ActivityThread类（这是个应用管理器，运行在我们Zygote孵化出来的应用进程内，ActivityManagerService通过它来管理应用进程）的main方法，在该方法会为应用进程准备好Looper和MessageQueue，然后调用attach方法将应用进程绑定到ActivityManagerService，然后开始loop循环，不断地获取和分发消息。

应用进程绑定到ActivityManagerService后，ActivityManagerService保存一个应用进程的代理对象，这样它就可以通过这个代理对象来控制应用进程。然后这个ActivityManagerService发消息给应用进程的一个默认Handler（居然有这个东西）启动Activity（终于到这了！！）。

如果应用的组件(包括所有组件Activity，Service，ContentProvider，Receiver) 被启动，肯定会先启动以应用包名为进程名的进程，这些组件都会运行在应用包名为进程名的进程里，并且是在主线程里。应用进程启动时会先创建Application对象，并执行Application对象的生命周期方法，然后才启动应用的组件。
有一种情况比较特殊，那就是为**组件**设置了特殊的进程名，也就是说通过android:process设置进程名的情况，此时组件运行在单独的进程内。

至此，我们的主的Activity伟大的生命周期开始。。。



---

#### 如何将一个Activity设置成窗口的样式？

在AndroidManifest.xml文件当中设置当前activity的一个属性（系统自带的属性）：
android:theme="@android:style/Theme.Dialog"

#### 如何退出Activity?如何安全退出已调用多个Activity的Application?

如果你的应用程序有很多Activity，而你又不在MainActivity上，要退出主界面，你需要连续多次按Back键才行。按Home键只是挂起，并没有退出程序。

解决办法：只需要一个专门的集合类对所有的活动进行管理就可以了。在活动管理器中，使用一个List来暂存活动，然后提供一个addActivity()方法用于向List中添加一个活动，使用removeActivity方法用于从List中移除活动，最后提供了一个finishAll()方法用于将List中所有的活动都全部销毁。

自定义一个BaseActivity让每个Activty都继承它，实现将Activity向活动集合中添加或删除的动作。

再或者，你可以用EventBus，发个消息给想结束的Activity，让它finish。

#### onActivityResult(int requestCode, int resultCode, Intent data)方法的用法?

```java
Intent intent=new Intent();
intent.putExtra("second", "I am second!");
setResult(RESULT_CODE, intent);
finish();
```

### 