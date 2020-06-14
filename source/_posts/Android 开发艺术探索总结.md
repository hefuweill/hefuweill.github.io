---
title: Android 开发艺术探索总结
date: 2018-03-11 18:24:39
type: "Android"
---

### 前言

前几天读完了 Android 开发艺术探索，这篇文章主要是记录下其中每个章节的重点，便于以后复习。<!-- more --> 

### Activity 的生命周期与启动模式

* Activity A 启动 Activity B，生命周期调用顺序为 A.onPause -> B.onCreate -> B.onStart -> B.onResume -> A.onStop 因此不能在 onPause 里面做耗时任务，这会导致新 Activity 延迟显示。

* onStart 调用完后 Activity 就可见了，onResume 调用完后 Activity 就可以与用户交互了，但是注意由于 View 的三大流程是在 onResume 回调完毕后才进行的，因此 onResume 调用完了，View 也是不会显示的，此时获取宽高都是 0。

* Activity 被意外杀死或者当配置发生改变后，系统会调用 onSaveInstanceState（API 28 后在 onStop 后调用，API 28以下在 onStop 前调用），这里可以保存下信息，然后在 onRestoreInstance 中恢复。

* Activity 的 onCreate 方法参数 bundle 可能为 null ，但是 onRestoreInstance 方法参数 bundle 一定不为空。

* Activity A 启动 Activity B (standard 模式)，会将 Activity B 放入 Activity A 所在的栈中，因此使用 Application.startActivity 需要添加 flag ，因为系统不知道将其放入哪个栈中。

* Luncher 应用程序在开启一个应用时，会对 Intent 设置 Intent.ACTIVITY\_NEW\_TASK ，所以其它应用程序不会和 Luncher 运行在一个任务栈中。

* SingleTask 启动模式，表示整个系统只能有该 Activity 的一个实例，并且启动它的时候附带清除 Top 的效果，比如当前栈中存在 ABCD 四个 Activity，这时候启动 B，栈中就变成了 AB，CD被清除了，并且还会调用 B.onNewIntent ，该启动模式的 Activity 始终运行在 taskAffinity 所指定的栈中。

* SingleInstance 启动模式，表示整个系统只能有该 Activity 的一个实例，启动它的时候会新开一个任务栈，该栈中只会存在它这么一个 Activity，再次启动它如果该栈没有销毁会调用其 onNewIntent 方法。

* 隐式启动的 Activity 必须加上默认的 category，因为当调用 startActivity、startActivityForResult  时系统会验证是否含有默认的 category，如果不加 Activity 就无法启动了(也就是说至少要有一个默认的 category, 入口 Activity 是特例不需要默认 category )

* 可以通过以下 adb 指令查看当前所有的任务栈：

  ```shell
  adb shell dumpsys activity activities
  ```

  

### IPC 机制

* 每个 Android 进程都含有一个虚拟机实例，而每个虚拟机加载同一个类时都会新建一个 Class 实例，因此进程一修改了 A 类中静态值，但是进程二获取的还是原来的值。
* 开启多个进程会导致 Application 被创建多个，由于每次启动组件时，系统都会判断 Application 对象是否存在，而显然非主进程的 Application 对象不存在，因此会创建。
* Android 进程间通信的方式主要有以下几种：
  1. 使用 Intent 传递数据。
  2. 使用共享文件和 SharedPreference。
  3. 使用 AIDL 、Messenger、ContentProvider，基于 Binder。
  4. 使用 Socket 。
* 将对象序列化保存到文件中，要求该类必须实现 serializable 接口，然后使用 ObjectInputSteam 写入，使用 ObjectOutputSteam 读取。

* 实现了 serializable 接口的类最好指定 serialVersionUID，若不指定编译器就会根据当前类自动加上该字段,当类字段发生变化时就会反序列化失败，因为序列化时的 UID 与当前类的UID不同，手动指定就不会导致该问题.当然类结构比如变量类型发生变化反序列化也会失败。

* 序列化的时候不会把静态变量和 transient (作用就是不参与序列化)标记的成员变量序列化，因为静态变量属于类而不属于对象。

* 实现了 parcelable 接口的类 writeToParcel 和 createFromParcel 的读写顺序必须要一致，不然会出错，parcelable 原理是使用共享内存，当使用内存序列化时使用 Parcelable，其他情况使用 Serialable，毕竟 Serialable 使用简单，但是开销大。

* Binder 对于 framework 来说是用于连接各种 Manger(ActivityManager、WindowManager) 和相应的 ServiceManager(AMS、WMS) 的桥梁,其跨进程传递的数据只能是 Parcelable 以及基本数据类型。

* Intent 能传输的数据类型包括 Serialable、Parcelable、CharSequence 以及基本数据类型。

* API 21 及以上要求 bindService、startService 需要调用 intent.setPackage（目标 Service 包名），因为 ContextImpl 中做了判断：

  ```java
  private void validateServiceIntent(Intent service) {
  	 if (service.getComponent() == null && service.getPackage() == null) {
  	    if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
  	        IllegalArgumentException ex = new IllegalArgumentException(
  	                "Service Intent must be explicit: " + service);
  	        throw ex;
  	    } else {
  	        Log.w(TAG, "Implicit intents with startService are not safe: " + service
  	                + " " + Debug.getCallers(2, 3));
  	    }
  	 }
  }
  ```

* AS 使用 aidl 遇到的坑 https://www.cnblogs.com/rookiechen/p/5352053.html，要使用自定义的类必须让该类实现 Parcelable 接口并且创建同名 .aidl 文件声明自己，最后在主 aidl 中引用。除此之外除基本数据类型，其他类型类型都得标上方向: in、out、或者 inout，并且 aidl 里面不能声明静态变量。

* AIDL 服务端程序验证客户端程序，可以通过自定义权限或者调用 getCallingUid 获取包名。

* AIDL 跨进程注册监听，需要了查看 https://blog.csdn.net/HJXASLZYY/article/details/52204379。

* AIDL 编译后生成的类如下：

  ```java
  public interface IBookManager extends android.os.IInterface {
  
  public static abstract class Stub extends android.os.Binder implements com.hfw.androidtest.IBookManager {
      private static final java.lang.String DESCRIPTOR = "com.hfw.androidtest.IBookManager";
    
      // 1
      public Stub() {
          this.attachInterface(this, DESCRIPTOR);
      }
  
      public static com.hfw.androidtest.IBookManager asInterface(android.os.IBinder obj) {
          if ((obj == null)) {
              return null;
          }
          // 2
          android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
          if (((iin != null) && (iin instanceof com.hfw.androidtest.IBookManager))) {
              return ((com.hfw.androidtest.IBookManager) iin);
          }
        	// 3
          return new com.hfw.androidtest.IBookManager.Stub.Proxy(obj);
      }
  
      @Override
      public android.os.IBinder asBinder() {
          return this;
      }
  
      @Override
      public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
          switch (code) {
              case INTERFACE_TRANSACTION: {
                  reply.writeString(DESCRIPTOR);
                  return true;
              }
              case TRANSACTION_getBookList: {
                  data.enforceInterface(DESCRIPTOR);
                  java.util.List<com.hfw.androidtest.Book> _result = this.getBookList();
                  reply.writeNoException();
                  reply.writeTypedList(_result);
                  return true;
              }
              case TRANSACTION_addBook: {
                	// 4
                  data.enforceInterface(DESCRIPTOR);
                  com.hfw.androidtest.Book _arg0;
                  if ((0 != data.readInt())) {
                      _arg0 = com.hfw.androidtest.Book.CREATOR.createFromParcel(data);
                  } else {
                      _arg0 = null;
                  }
                  this.addBook(_arg0);
                  reply.writeNoException();
                  return true;
              }
          }
          return super.onTransact(code, data, reply, flags);
      }
  
      private static class Proxy implements com.hfw.androidtest.IBookManager {
          private android.os.IBinder mRemote;
  
          Proxy(android.os.IBinder remote) {
              mRemote = remote;
          }
  
          @Override
          public android.os.IBinder asBinder() {
              return mRemote;
          }
  
          public java.lang.String getInterfaceDescriptor() {
              return DESCRIPTOR;
          }
          @Override
          public java.util.List<com.hfw.androidtest.Book> getBookList() throws android.os.RemoteException {
              android.os.Parcel _data = android.os.Parcel.obtain();
              android.os.Parcel _reply = android.os.Parcel.obtain();
              java.util.List<com.hfw.androidtest.Book> _result;
              try {
                  _data.writeInterfaceToken(DESCRIPTOR);
                  mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                  _reply.readException();
                  _result = _reply.createTypedArrayList(com.hfw.androidtest.Book.CREATOR);
              } finally {
                  _reply.recycle();
                  _data.recycle();
              }
              return _result;
          }
  
          @Override
          public void addBook(com.hfw.androidtest.Book book) throws android.os.RemoteException {
              android.os.Parcel _data = android.os.Parcel.obtain();
              android.os.Parcel _reply = android.os.Parcel.obtain();
              try {
                  _data.writeInterfaceToken(DESCRIPTOR);
                  if ((book != null)) {
                      _data.writeInt(1);
                      book.writeToParcel(_data, 0);
                  } else {
                      _data.writeInt(0);
                  }
                  mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                  _reply.readException();
              } finally {
                  _reply.recycle();
                  _data.recycle();
              }
          }
      }
  
      static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
      static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
  }
  
  	public java.util.List<com.hfw.androidtest.Book> getBookList() throws android.os.RemoteException;

  	public void addBook(com.hfw.androidtest.Book book) throws android.os.RemoteException;
}
  ```
  
  主要调用流程为：
  
  1. 服务端先调用 Binder binder = new IBookManager.Stub()  实现 getBookList、addBook，接着在 onBind 方法中返回 binder 。
  2. 客户端在 onServiceConnected 回调中拿到 IBinder 对象，调用 IBookManager.Stub.asInterface，拿到 IBookManager 实例。内部原理是，如果是同一个进程那么直接把服务端返回的 binder 返回，这点可以根据代码 1、2 看出，如果不是同一个进程那么根据代码 3 新建一个 Proxy 对象并返回。
  3. 客户端调用 binder.addBook，如果是同一个进程相当于普通的方法调用，如果不是同一个进程则执行到 Proxy.addBook，首先获取两个 Parcel 对象 \_data、\_reply 其中前者用来传递参数，后者用于接收参数，接着写入 1，表示一个参数，再将 book 实例写入 _data，然后调用 mRemote.transact 传入调用的方法索引，以及两个 Parcel 对象，注意这个的 mRemote 为 onServiceConnected 返回的 BinderProxy 对象。
  4. 调用 transact 方法后会挂起当前线程，接着会在服务端调用 binder.onTransact 方法，根据传入的方法索引，执行代码 4，从 Parcel 对象 data 中取出参数 Book 实例，执行服务端实现的 addBook 方法，由于这个方法无返回值所以不需要写入返回值到 reply 对象，接着客户端线程停止挂起继续执行。



### View 的事件体系

* 当两次滑动距离小于该值时系统不认为滑动，可以通过以下代码获取：

  ```java
  ViewConfiguration.get(getContext()).getScaledTouchSlop();
  ```

* VelocityTracker 用于获取水平或垂直的速度典型代码如下所示：

  ```java
  @Override
  public boolean onTouchEvent(MotionEvent event) {
      VelocityTracker velocityTracker = VelocityTracker.obtain();
      velocityTracker.addMovement(event);
      velocityTracker.computeCurrentVelocity(1000);
      Log.d(TAG, "onTouchEvent: x="+velocityTracker.getXVelocity()+";y="+velocityTracker.getYVelocity());
      return super.onTouchEvent(event);
  }
  ```

* 实现 View 移动可以使用以下几种方法：

  1. scrollTo、scrollBy。
  2. 修改 View 的 layoutParams。
  3. 修改 View 的 transactionX、transactionY。
  4. offsetLeftAndRight、offsetTopAndBottom。

* scrollTo 与scrollBy 这两个方法都是移动 View 内的内容比如调用 Linearlayout.scrollTo 是将 Linearlayout 内部的内容移动，当调用 TextView.scrollTo 时是将其上的文字移动 TextView 本身不移动，同时参数 x 为正时向左移动，y 为正时向上移动。两者的区别时前者是相对于初始位置，连续调用同样参数无效，后者相对于当前位置可以重复执行(内部也是调用前者只是 x, y 每次都改变)。
* 使用 View 动画不会改变 View 的真实位置，只是用户看起来改变了而已，对于系统来说 View 的位置是没有改变的，并且如果不设置 setFillAfter(true) 在动画结束后又会回到原位置，因此 button 在动画结束的位置触发不了单击事件，而在初始位置可以触发。
* 当改变了一个 View 的 layoutParams 时有两种方法可以生效，一种是再设置回去 View.setLayoutParams ，另一种是调用 View.requestLayout。
* 实现 View 弹性滑动的方式有 Scroller、ObjectAnimator、延迟策略（滑动以下睡眠一下）。
* Scroller 原理是 startScroll 以后会调用 invalidate ，然后会再调用 View.onDraw ，onDraw 里面会再调用computeScroll 该方法在 View 内是空方法。
* View 的 clickable 或 longclickable 有一个为 true 就会消耗点击事件，setClickListener 会把 clickable 设置为true，setLongClickListener 同理。



### View 的工作原理

* ViewRoot 的作用主要是与 WMS 通信绘制 GUI 和给 DecorView 分发用户事件。

* invalidate 会调用onDraw，requestLayout 会调用 onMeasure 、onLayout() 不调用 onDraw。
* MeasureSpec 分为以下三种：
  1. UNSPECIFIED 容器不对 View 的大小做任何限制，子 View 要多大给多大，一般用于系统内部，比如ScrollView 对子 View 的高度。
  2. EXACTLY 确定值，父容器已经检测出当前 View 所需的大小，对应于 LayoutParams 里面的 match_parent 以及确定值。
  3. AT_MOST 父容器指定了一个大小，子 View 不能超过这个大小对应于 layoutParams 里面的wrap_content。

* onMeasure 执行完后 getMeasuredWidth、getMeasureHeight 才能取到，其值大部分情况下等于getWidth、getHeight、最好是在 onLayout 后去获取最终高度。

* View 真正进行 measure layout draw 是在 Activity 的 onResume 执行完后,因此 onResume 时布局并不可见。

* View 的宽高在 onCreate、onStart、onResume 里面都无法获取到，可以通过以下几种方式获得：

  1. 在 Activty、View 的 onWindowFocusChanged 里面获取。
  2. 添加 ViewTreeObserver。
  3. View.post。
  4. View.measure。

* 直接继承自 View 的自定义 View 需要注意解决 wrap\_content 情况因为如果不做处理 wrap_content 将于match_parent 效果一致，详见 View.getDefaultSize，此外还必须处理 padding (不需要处理 margin，margin 由父控件处理)处理方法就是在 onDraw 里面考虑内边界。

* View 被从 Window 中移除时会调用 View.onDetachedFromWindow 该方法内部可以做一些资源回收，停止线程的操作。

* RemoteViews 用于 Notification 中的自定义 View 以及 Widget，而其归根到底其实都是在 SystemServer 进程里面加载的，所以是跨进程的。

* 创建 Widget 步骤：

  1. 新建一个布局文件比如 a.xml。

  2. 在res里面新建文件夹 xml，再在该文件夹下面新建一个布局比如 b.xml 里面代码如下所示：

     ```xml
     <?xml version="1.0" encoding="utf-8"?>
     <appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
         android:initialLayout="@layout/a" 
         android:minHeight="84dp" android:minWidth="84dp"
         android:updatePeriodMillis="5000000"><!--自动更新时间-->
     </appwidget-provider>
     ```

  3. 新建一个类继承自 AppWidgetProvider，小组件的点击事件在 onUpdate 里面进行设置，该方法会在每个小组件添加时就自动调用一次。

  4. 在Manifast文件中进行注册如下所示：

     ```xml
     <receiver android:name=".MyAppWidget">
         <meta-data android:name="android.appwidget.provider"
                  android:resource="@xml/app"/>
         <intent-filter>
           <action android:name="anniu" /> <!--下行必须加不然小组件将无法显示-->
           <action android:name="android.appwidget.action.APPWIDGET_UPDATE"/>
           <category android:name="android.intent.category.DEFAULT" />
         </intent-filter>
     </receiver>
     ```

* RemoteViews 里面只能使用一些系统内置的控件，其子控件及自定义控件都无法使用。

* RemoteViews 的原理,当我们调用 RemoteViews.setXXX 时系统都会创建一个 Action，并将其放入一个ArrayList 中进行保存,只有调用对应 manager 中的更新方法时才会调用 RemoteViews 里面的 apply 方法,该方法里面再一一执行 action 的 apply 方法，这样就能避免多次 IPC 请求。

* 获取本应用中的资源文件可以通过 getResource().getIdentifer("布局文件名字", "layout", getPackageName()) 获取。

### Android Drawable

* selector 文件系统是根据从上到下的顺序来匹配 drawable 的所以必须把默认图案的 drawable 放到最下面，不然将一直显示默认图片。

### Android 动画深入分析

* Android的动画分为三种：View动画、帧动画、属性动画。View 动画既可以是单种动画,也可以是四种动画组合。组合动画所对应的类是 AnimationSet类，推荐在xml里面定义动画。

* 帧动画在 res/drawable 目录下创建一个 xml 文件根节点为 animation-list 然后在代码中View.setBackgroundResource()，然后再View.getBackground().start()。

* LayoutAnimation 的使用先在 anim 目录下新建一个以 layoutAnimation 为根节点的 xml 文件,其中有个属性animation 需要依赖一个其他的 animation,然后在布局文件中为 ViewGroup 设置 layoutAnimation 就行了(案例是给 ListView 设置，会出现刚刚载入布局时每个 item 有序的从右到左从完全透明到完全不透明的动画)。

* Activity 的切换动画 使用 overridePendingTransition 指定开启动画以及暂停动画，该方法必须紧挨startActivity 或 finish 后面否则将不起作用，其中 exitAnim 动画是给需要消失的 Activity 使用的。

* ObjectAnimator 继承自 ValueAnimator，AnimatorSet 代表动画集合。ObjectAnimator 用于改变一个对象的属性并且赋予动画，ValueAnimator需要自己设置监听自己更改，可以通过 xxxAnimator.ofXXX 获取， AnimatorSet 使用 playTogether(ValueAnimator…) 或者 playSequentially()，如果要对 Color 使用属性动画要设置：

  ```java
  setEvaluator(new ArgbEvaluator());
  ```

* ObjectAnimator 要求对象必须要有 setXXX 的方法，因为原理就是根据时间来生成一个值然后一个个通过反射调用 setXXX 方法设置，并且如果 value 可变参数只指定一个那么该对象就必须要有 getXXX 方法因为当属性动画开始时系统会通过反射调用该方法,如果找不到该方法,程序会直接崩溃。

* ValueAnimator 设置 AnimatorListener 时如果不需要实现所有的回调应该使用 AnimatorListenerAdapter。

* 当属性动画显示出的效果不如预期，得先看看是否该对象的 set 和 get 方法在操作同一个属性，如果不是可以通过包装类进行实现。

### 理解 Window 和 WindowManager

* Activity 内可以直接调用 getWindowManager 获取 WindowManager 实例。

* Activity 其实也是通过调用 WindowManager.addView 添加的(在 ActivityThread.handleResumeActivity 里面。

* 通过 WindowManager 显示一个全局的悬浮窗代码如下：

  ```java
  if (Build.VERSION.SDK_INT >= 23) {
      if (Settings.canDrawOverlays(this)) {
         showWindow();
      } else {
          //打开设置去授权悬浮框权限
          Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
          startActivityForResult(intent,1);
      }
  } else {
      showWindow();
  }
  private void showWindow() {
      Button bt_window = new Button(this);
      bt_window.setText("我是悬浮");
      WindowManager wm = (WindowManager) getSystemService(Service.WINDOW_SERVICE);
      WindowManager.LayoutParams params = new WindowManager.LayoutParams(-2,-2,0,0, PixelFormat.TRANSPARENT);
      params.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE |//表示自己不需要获取焦点,事件向下传递,同时拥有下面的flag特性
      WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL |//表示范围内的事件接受范围外不接受
      WindowManager.LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING;//已经过时
      params.type = WindowManager.LayoutParams.TYPE_SYSTEM_ERROR;//type必须设置,并且该type需要加上权限 system_alert_window
      if (wm != null) {
          wm.addView(bt_window,params);
      }
  }
  protected void onActivityResult(int requestCode, int resultCode, Intent data) {
      super.onActivityResult(requestCode, resultCode, data);
      if(requestCode == 1){
          if(Build.VERSION.SDK_INT >= 23){
              if(Settings.canDrawOverlays(this)){
                  showWindow();
              }
          }
      }
  }
  ```

  

* WindowManager 主要有三个重要方法 addView、removeView、updateViewLayout 其中 addView 通过IPC调用 WMS.addView ，removeView 分为两个方法 removeView、removeImmediate 前者只是发了一个消息就返回了，不一定立马执行，后者直接执行，建议使用前者。

* PhoneWindow 里面维护了一个 DecorView 对象，可以设置 Callback (比如 Activity.attach 就设置了，回调中包括触摸事件等)。

* Toast 对 NMS发出请求，NMS 在能弹出的时候通过 IPC 回调 Tn.show，方法运行在 binder 线程池中需要通过 Handler 转到原来的线程，因为 Handler 需要 Looper，所以 Toast 无法在没 Looper 的线程中使用。

  


### 四大组件的工作过程

* 当程序中使用 startActivity、startService、bindService 时会通过 IPC 调用 AMS 里面的对应的方法后经过一系列流程通过调用 ApplicationThread 里面的方法,最后调到 H 类里面对应的方法。
* Service 的 start 过程，ComtextImpl.startService -> 验证Intent内是否包含了包名 -> AMS.startService -> 经过一系列操作后发送一个 message 让目标应用程序的 ActivityThread 创建 Service 对象，并且调用 Service.onCreate() -> 目标应用程序告诉 AMS 已经执行完毕 -> AMS 再给 ActivityThread 发送消息调用 Service.onStartCommand()。
* Service 的 bind 过程，ComtextImpl.bindService -> 验证Intent内是否包含了包名 -> 将 ServiceConnection对象进行包装成 Binder对象 -> AMS.bindService -> 经过一系列操作后发送一个 message 让目标应用程序的 ActivityThread 创建 Service 对象 并且调用 Service.onCreate -> 目标应用程序告诉 AMS 已经执行完毕 -> AMS 给 client 发送消息调用 Service.onBind -> client 告诉 AMS 执行完毕并且把 onBind 返回的对象传递给 AMS -> AMS通过IPC调用 ServiceConnection.connected。

### Android 的消息机制

* Handler 作用就是把当前线程切换到 Handler 里面的 Looper 所在的线程,比如在主线程创建 Handler，在子线程获取数据再通过 Handler 切换到主线程执行，要使用 Handler 当前线程必须要先含有 Looper。
* ThreadLocal.get 可以取出当前线程中所 set 的值,在 android 消息机制中用于保存每个线程的 Looper。
* Looper.loop 死循环唯一跳出这个循环的条件是 messageQuene.next 返回null (只有调用了 Looper.quit 方法才能退出循环)，当取出消息后,会进行 dispatch 然后再调用 meassageQuene.next 再阻塞。
* 使用 Handler 时如果不想创建其子类的实例可以使用 handler.post(Runnable)，或者在创建 Handler 对象时传入一个 Callback 对象。

### Android 的线程和线程池

* IntentService 处理逻辑一般在 onHandleIntent 中处理，该方法是在子线程中运行的，当该方法运行完并且没有再次 startService 会自动销毁该 service，多次 startService 会多次调用 onHandleIntent 。

* AsyncTask 的使用方式

    ```java
    class MyUpdateTask extends AsyncTask<Void,Integer,Void>{
    
        @Override
        protected void onPreExecute() {
            super.onPreExecute();//主线程
        }
    
        @Override
        protected Void doInBackground(Void... voids) {
            for(int i = 1; i < 100; i++){
                SystemClock.sleep(100);// 该方法运行于子线程
                publishProgress(i);// 该方法会进行进程切换调用主线程的 onProgressUpdate()
                if(isCanceled()){
                    break; //取消了把线程结束
                }
            }
            return null;//返回值作为onPostExecute()的参数
        }
    
        @Override
        protected void onPostExecute(Void result) {
            super.onPostExecute(result)//主线程
        }
    
        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
            pb.setProgress(values[0]);
        }
    }
    ```

    

* AsyncTask 的注意点
    1. AsuncTask 对象必须在主线程中创建，API 21 以下如果不在主线程创建会崩溃。
    2. AsyncTask.execute 也必须在主线程中调用。
    3. AsyncTask.execute 只能调用一次，再次调用会报异常，因为会在调用 execute 时检查当前的状态。
    4. AsyncTask.execute()是串行执行的当一个任务执行完,才会执行下一个任务,如果需要并行执行可以调用AsyncTask.executeOnExecutor。
    5. 不要在程序中显示的调用上面 4 个方法。

### Bitmap 的加载和缓存

* Bitmap 加载前需要进行缩放，因为一般 ImageView 的大小都比图片的小，可以使用 BitmapFactory.options里面的 inSampleSize，当该参数 <= 1 时不缩放，为2时宽高都为原来的 1/2 ,占用空间变为原来的 1/4。

* Bitmap 压缩加载防止 OOM 的步骤如下：

    1. BitmapFactory.Options.inJustDecodeBounds 设置为 true 去加载图片，该参数作用是不把图片加载进内存，但可以获取其宽高。
    2. BitmapFactory.Options 中取出图片的原始宽高信息使用 outWidth、outHeight.

    3. 根据 ImageView 大小以及原图大小计算出 inSampleSize。
    4. 将 inJustDecodeBounds 设置为 false，然后重新加载图片。

    ```java
    public static Bitmap decodeSampledBitmapFromResource(Resources resources,int resId, int reqWidth, int reqHieght){
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(resources, resId, options); //为了获取原始图片的宽高
        options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHieght);
        options.inJustDecodeBounds = false;
        return BitmapFactory.decodeResource(resources, resId, options);
    }
    
    private static int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHieght) {
        final int height = options.outHeight;
        final int width = options.outWidth;
        int inSampleSize = 1;
        if(height > reqHieght || width > reqWidth){
            final int halfHeight = height / 2;
            final int halfWidth = width / 2;
            while((halfHeight / inSampleSize) >= reqHieght && (halfWidth / inSampleSize) >= reqWidth){
                inSampleSize *= 2;
            }
        }
        return inSampleSize;
    }
    ```

    

### 综合技术

* 捕获全局异常可以通过创建一个实现 UncaughtExceptionHandler 的类，然后调用Thread.setDefaultUncaughtException 来捕获全局崩溃信息。

* 一个 dex 文件的方法数量最多为 65536 个，当超过后需要进行如下设置。
    1. defaultConfig 中加入 multiDexEnabled = true。
    2. dependencies 中加入依赖 compile 'com.android.support:multidex:1.0.0'。
    3. 应用的 Application 使用 MultiDexApplication 或其子类。