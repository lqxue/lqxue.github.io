---
title: Activity启动模式
date: 2016-05-16
tags:
  - Activity
categories: Android
---

## Activity的LaunchMode

首先说一下`Activity`为什么需要启动模式。我们知道，在默认的情况下，当我们多次启动同一个`Activity`的时候，系统会创建多个实例并把他们一一放入任务栈中，当我们点击back键的时候会发现这些Activity会一一回退。任务栈是一种`后进先出(LIFO, Last In First Out)`的栈结构，这个好理解，每按一次back键就有一个`Activity`退出栈，直到栈空为止，当这个栈为空的时候，系统就会回收这个任务栈。目前Activity目前有四种启动模式:`standard`、`singleTop`、`singleTask`、`singleInstance`。

<!-- more -->

1. standard:标准模式，这也是系统的默认模式，每次启动一个Activity都会重新创建一个实例，不管这个实例是否已经存在。被创建的实例的生命周期符合典型情况下Activity的生命周期，`onCreate()`、`onStart()`、`onResume()`都会被调用，在这种模式下，谁启动了这个`Activity`，那么这个`Activity`就运行在启动它的`Activity`所在的栈内，比如`Activity A`启动了`Activity B（B是标准模式）`，那么`B`就会进入到`A`所在的栈内，不知道读者有没有注意到在`安卓5.1`版本的手机上，当我们用`ApplicationContext`去启动`standard`模式的`Activity`的时候就会报错：

```java
E/AndroidRuntime(674): android.util.androidruntiomException: 
    Calling startActivity from outside of an Activity context requires the FLAG_ACTIVITY_TASK flag . 
    Is this really what are want?
```

这是因为我们的`standard`模式的`Activity`默认会进入启动它的Activity所属的任务栈中，但是由于非`Activity`类型的`Context（如ApplicationContext）`并没有所谓的任务栈，所以这就有问题了，解决这个问题，就是给待启动`Activity`指定`FLAG_ACTIVITY_TASK`标记位，这样启动的时候就会为它创建一个新的任务栈，这个时候待启动Activity实际上是以`singleTask`模式启动的。

2. singleTop:栈顶复用模式，在这个模式下，如果新的Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的`onNewIntent`方法会被调用，通过此方法的参数我们可以取出当前请求的信息。需要注意的是，这个`Activity`的`onCreate`、`onStart`不会被系统调用，因为他并没有发生改变。如果新Activity已存在但不是在栈顶，那么新Activity则会重新创建，举个例子，假设现在栈内的情况为ABCD，其中ABCD为四个Activity，A位于栈底，D位于栈顶，这个时候假设要再启动D，如果D的启动模式为singleTop，那么站栈内的情况仍然是ABCD，如果D的启动模式是standard，那么由于D会被重新创建，导致情况就是ABCDD。

3. singTask：栈内复用模式，这是一种单实例模式，在这种模式下，只要Activity在一个栈内存在，那么多次启动此Activity都不会重新创建实例，和`singTop`一样，系统也会回调其`onNewIntent`方法。具体一点，当一个具有`singleTask`模式的Activity请求启动后，比如`Activity A`，系统首先会去寻找是否存在A想要的任务栈，如果不存在，就重新创建一个任务栈，然后创建A的实例后把A放进栈中，如果存在A所需要的栈，这个时候就要看A是否在栈中有实例存在，如果实例存在，那么系统就会把A调到栈顶并调用它的`onNewIntent`方法，如果实例不存在，就创建A的实例并且把A压入栈中，举几个例子:

    - 比如目前任务栈S1中的情况为ABC，这个时候`Activity D`以`singleTask`模式请求启动，其所需的任务栈为S2，由于S2和D的实例都不存在，所以系统会先创建任务栈S2，然后再创建D的实例并将其入栈到S2。
    - 另外一种情况，假设D所需的任务栈为S1，其他情况如如上面的一样，那么由于S1已经存在，所以系统会直接创建D的实例并将其入栈到S1中。
    - 如果D所需要的任务栈为S1，并且当前任务栈S1的情况为ADBC，根据栈内复用的原则，此时D不会被重新创建，系统会把D切换到栈顶并且调用其`oNnNewIntent`方法，同时由于`singleTask`默认具有`clearTop`的效果，会导致栈内所有在D上面的Activity全部出栈，于是最终S1中的情况为AD。
    
4. singleInstance:单实例模式，这是一种加强的`singleTask`的模式，它除了具有singleTask的所有属性之外，还加强了一点，那就是具有此模式下的Activity只能单独的处于一个任务栈中，换句话说，比如`Activity A`是`singleInstance`模式，当A启动的时候，系统会为它创建创建一个新的任务栈，然后A独立在这个任务栈中，由于栈内复用的特性，后续的请求均不会创建新的Activity，除非这个独特的任务栈被系统销毁了。

在singkleTask启动模式中，多次提到了某个Activity所需的任务栈，什么是Activity所需的任务栈呢？这要从一个参数说起：`TaskAffinity`,可以翻译成任务相关性，这个参数标识了一个Activity所需要的任务栈的名字，默认情况下，所有的Activity所需要的任务栈的名字为`应用的包名`，当然，我们可以为每个Activity都单独指定`TaskAffinity`属性，这个属性值必须必须不能和包名相同，否则就相当于没有指定，TaskAffinity属性主要和`singleTask`启动模式或者`allowTaskReparenting`属性配合使用，在其他状况下没有意义，另外，任务栈分为前台任务栈和后台任务栈，后台任务栈中的Activity位于暂停状态，用户可以通过切换将后台任务栈再次调为前台

当TaskAffinity和singleTask启动模式配对使用的时候，他是具有该模式Activity目前任务栈的名字，待启动的Activity会运行在名字和TaskAffinity相同的任务栈中

如何给Activity指定启动模式？有两种方法，第一种是通过清单文件为Activity指定

```java
<activity 
    android:name=".SecondActivity"
    android:launchMode="singleTask"/>
```

另一种启情况就是通过intent的标志位为Activity指定启动模式

```java
Intent intent = new Intent();
intent.setClass(this,SecondActivity.class);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

这两种方式都可以为Activity指定启动模式，但是二者还是有一些区别的，首先，优先级上，第二种比第一种高，当两种同时存在的时候，以第二种为准，其次，上述两种方式在限定范围内有所不同，比如，第一种方式无法直接为Activity设置`FLAG_ACTIVITY_CLEAR_TOP`标识，而第二种方式无法指定`singleInstance`模式

## 验证启动模式

接下来我们一起验证一下四个启动模式。先从最简单的启动模式开始验证。

为了打印方便，定义一个BaseActivity，在其onCreate方法和onNewIntent方法中打印出当前Activity的日志信息，主要包括所属的task，当前类的hashcode，以及taskAffinity的值。之后我们进行测试的Activity都直接继承该Activity

```java
public class BaseActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Log.e("TAG", "===========================================onCreate=========================================================");
        Log.e("TAG", "onCreate " + getClass().getSimpleName() + " TaskId: " + getTaskId() + " hasCode:" + this.hashCode());
        dumpTaskAffinity();
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        Log.e("TAG", "===========================================onNewIntent=========================================================");
        Log.e("TAG", "onNewIntent " + getClass().getSimpleName() + " TaskId: " + getTaskId() + " hasCode:" + this.hashCode());
        dumpTaskAffinity();
    }

    protected void dumpTaskAffinity(){
        try {
            ActivityInfo info = this.getPackageManager()
                    .getActivityInfo(getComponentName(), PackageManager.GET_META_DATA);
            Log.e("TAG", "taskAffinity:"+info.taskAffinity);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

### standard模式

这个模式是默认的启动模式，即标准模式，在不指定启动模式的前提下，系统默认使用该模式启动Activity，每次启动一个Activity都会重新创建一个新的实例，不管这个实例存不存在，这种模式下，谁启动了该模式的Activity，该Activity就属于启动它的Activity的任务栈中。

新建一个Activity，并声明在manifest文件中

```java
<activity
    android:name=".StandardActivity"
    android:launchMode="standard" />
```

对于standard模式，android:launchMode可以不进行声明，因为默认就是standard。

StandardActivity 的代码如下，入口MainActivity中有一个按钮来启动StandardActivity，StandardActivity中也有一个按钮再次启动StandardActivity。

```java
public class StandardActivity extends BaseActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_standard);

        findViewById(R.id.jump).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(StandardActivity.this, StandardActivity.class);
                startActivity(intent);
            }
        });
    }

}
```

我们首先从MainActivity进入StandardActivity，进入后再点击的按钮，再按四次返回键不断返回。

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 257 hasCode:966292410
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate StandardActivity TaskId: 257 hasCode:891473618
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate StandardActivity TaskId: 257 hasCode:455046717
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate StandardActivity TaskId: 257 hasCode:340861940
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate StandardActivity TaskId: 257 hasCode:386514855
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
```

可以看到日志输出了四次`StandardActivity`和一次`MainActivity`，从MainActivity进入StandardActivity一次，后来我们又按了三次按钮，总共四次StandardActivity的日志，并且所属的任务栈的id都是257，这也验证了**谁启动了该模式的Activity，该Activity就属于启动它的Activity的任务栈中**这句话，因为启动StandardActivity的是MainActivity，而MainActivity的taskId是257，因此启动的StandardActivity也应该属于id为257的这个task，后续的3个StandardActivity是被StandardActivity这个对象启动的，因此也应该还是257，所以taskId都是257。并且每一个Activity的hashcode都是不一样的，说明他们是不同的实例，即**每次启动一个Activity都会重写创建一个新的实例**

### singleTop，栈顶复用模式

这个模式下，如果新的activity已经位于栈顶，那么这个Activity不会被重新创建，同时它的onNewIntent方法会被调用。如果栈顶不存在该Activity的实例，则情况与standard模式相同。

SingleTopActivity代码和StandardActivity类似，只不过记得在manifest文本后中修改启动模式。

```java
<activity
    android:name=".SingleTopActivity"
    android:launchMode="singleTop" />
```

操作和standard模式类似，直接贴输出日志

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 259 hasCode:966292410
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleTopActivity TaskId: 259 hasCode:891473618
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onNewIntent=========================================================
07/com.activitydemo E/TAG: onNewIntent SingleTopActivity TaskId: 259 hasCode:891473618
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onNewIntent=========================================================
07/com.activitydemo E/TAG: onNewIntent SingleTopActivity TaskId: 259 hasCode:891473618
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onNewIntent=========================================================
07/com.activitydemo E/TAG: onNewIntent SingleTopActivity TaskId: 259 hasCode:891473618
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
```

我们看到，除了第一次进入SingleTopActivity这个Activity时，输出的是onCreate方法中的日志，后续的都是调用了onNewIntent方法，并没有调用onCreate方法，并且四个日志的hashcode都是一样的，说明栈中只有一个实例。这是因为第一次进入的时候，栈中没有该实例，则创建，后续的三次发现栈顶有这个实例，则直接复用，并且调用onNewIntent方法。那么假设栈中有该实例，但是该实例不在栈顶情况又如何呢。

我们先从MainActivity中进入到SingleTopActivity，然后再跳转到OtherActivity中，再从OtherActivity中跳回SingleTopActivity，再从SingleTopActivity跳到SingleTopActivity中，看看整个过程的日志。

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 266 hasCode:966292410
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleTopActivity TaskId: 266 hasCode:891473618
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate OtherActivity TaskId: 266 hasCode:867251760
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleTopActivity TaskId: 266 hasCode:322979577
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onNewIntent=========================================================
07/com.activitydemo E/TAG: onNewIntent SingleTopActivity TaskId: 266 hasCode:322979577
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
```

我们看到从MainActivity进入到SingleTopActivity时，新建了一个SingleTopActivity对象，并且task id与MainActivity是一样的，然后从SingleTopActivity跳到OtherActivity时，新建了一个OtherActivity，此时task中存在三个Activity，从栈底到栈顶依次是MainActivity，SingleTopActivity，OtherActivity，此时如果再跳到SingleTopActivity，即使栈中已经有SingleTopActivity实例了，但是依然会创建一个新的SingleTopActivity实例，这一点从上面的日志的hashCode可以看出，此时栈顶是SingleTopActivity，如果再跳到SingleTopActivity，就会复用栈顶的SingleTopActivity，即会调用SingleTopActivity的onNewIntent方法。这就是上述日志的全过程。

对以上内容进行总结

- standard启动模式是默认的启动模式，每次启动一个Activity都会新建一个实例不管栈中是否已有该Activity的实例。
- singleTop模式分3种情况
    - 当当前栈中已有该Activity的实例并且该实例位于栈顶时，不会新建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调onNewIntent方法。
    - 当当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例
    - 当当前栈中不存在该Activity的实例时，其行为同standard启动模式。
- standard和singleTop启动模式都是在原任务栈中新建Activity实例，不会启动新的Task，即时你指定了taskAffinity属性。

那么什么是taskAffinity属性呢，可以简单的理解为任务相关性。

- 这个参数标识了一个Activity所需任务栈的名字，默认情况下，所有Activity所需的任务栈的名字为应用的包名。
- 我们可以单独指定每一个Activity的taskAffinity属性覆盖默认值
- 一个任务的taskAffinity决定于这个任务的根activity（root activity）的taskAffinity。
- 在概念上，具有相同的taskAffinity的activity（即设置了相同taskAffinity属性的activity）属于同一个任务。
- 为一个activity的taskAffinity设置一个空字符串，表明这个activity不属于任何task。

很重要的一点taskAffinity属性不对standard和singleTop模式有任何影响，即时你指定了该属性为其他不同的值，这两种启动模式下不会创建新的task

我们现在尝试下指定之前的例子的`taskAffinity`分别为其他不同的值，入口MainActivity不指定（不指定即默认值，即包名）

```java
<activity
    android:name=".StandardActivity"
    android:label="StandardActivity"
    android:launchMode="standard"
    android:taskAffinity="com.activitydemo.standard" />
    
<activity
    android:name=".SingleTopActivity"
    android:label="SingleTopActivity"
    android:launchMode="singleTop"
    android:taskAffinity="com.activitydemo.singleTop" />
```

分别启动这两个Activity看日志输出

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 269 hasCode:966292410
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate StandardActivity TaskId: 269 hasCode:891473618
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo.standard
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleTopActivity TaskId: 269 hasCode:867251760
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo.singleTop
```

我们看到入口Activity的taskAffinity值就是包名，就是默认情况下不指定的。然后StandardActivity和SingleTopActivity的taskAffinity值被我们覆盖了，分别为不同的值，但是这两个Activity启动的时候任务栈并没有新建，而是直接在原来的Task中启动，这说明这个taskAffinity对这两种启动模式没有什么影响。其实该属性主要是配合SingleTask启动模式使用的。接下来我们看该启动模式，可以说这个启动模式是最复杂的。

### singleTask，即栈内复用模式

这个模式十分复杂，有各式各样的组合。在这个模式下，如果栈中存在这个Activity的实例就会复用这个Activity，不管它是否位于栈顶，复用时，会将它上面的Activity全部出栈，并且会回调该实例的onNewIntent方法。其实这个过程还存在一个任务栈的匹配，因为这个模式启动时，会在自己需要的任务栈中寻找实例，这个任务栈就是通过taskAffinity属性指定。如果这个任务栈不存在，则会创建这个任务栈。

指定Activity为singleTask模式

```java
<activity
    android:name=".SingleTaskActivity"
    android:label="SingleTaskActivity"
    android:launchMode="singleTask"/>
```

现在我们不指定任何taskAffinity属性，对它做类似singleTop的操作，即从入口MainActivity进入SingleTaskActivity，然后跳到OtherActivity，再跳回到SingleTaskActivity。看看整个过程的日志。

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 273 hasCode:966292410
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleTaskActivity TaskId: 273 hasCode:149127957
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate OtherActivity TaskId: 273 hasCode:574150429
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onNewIntent=========================================================
07/com.activitydemo E/TAG: onNewIntent SingleTaskActivity TaskId: 273 hasCode:149127957
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
```

当我们从MainActiviyty进入到SingleTaskActivity，再进入到OtherActivity后，此时栈中有3个Activity实例，并且SingleTaskActivity不在栈顶，而在OtherActivity跳到SingleTaskActivity时，并没有创建一个新的SingleTaskActivity，而是复用了该实例，并且回调了onNewIntent方法。并且原来的OtherActivity出栈了，具体见下面的信息，使用命令`adb shell dumpsys activity activities | grep com.activitydemo`可进行查看

```java
TaskRecord{e179f0c #273 A=com.activitydemo U=0 sz=2}
Run #3: ActivityRecord{3d025344 u0 com.activitydemo/.SingleTaskActivity t273}
Run #2: ActivityRecord{33e43c90 u0 com.activitydemo/.MainActivity t273}
```

可以看到当前栈中只有两个Activity，即原来栈中位于SingleTaskActivity 之上的Activity都出栈了。

这时候，如果我们指定SingleTaskActivity 的taskAffinity值。

```java
<activity
    android:name=".SingleTaskActivity"
    android:label="SingleTaskActivity"
    android:launchMode="singleTask"
    android:taskAffinity="com.activitydemo.singleTask" />
```
还是之前的操作。但是日志就会变得不一样。

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 275 hasCode:373647994
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleTaskActivity TaskId: 276 hasCode:441581669
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo.singleTask
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate OtherActivity TaskId: 276 hasCode:300157549
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onNewIntent=========================================================
07/com.activitydemo E/TAG: onNewIntent SingleTaskActivity TaskId: 276 hasCode:441581669
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo.singleTask
```

我们看到SingleTaskActivity所属的任务栈的TaskId发生了变换，也就是说开启了一个新的Task，并且之后的OtherActivity也运行在了该Task上

打印出信息也证明了存在两个不同的Task

```java
TaskRecord{1c2218e4 #276 A=com.activitydemo.singleTask U=0 sz=1}
Run #3: ActivityRecord{1904086d u0 com.activitydemo/.SingleTaskActivity t276}
TaskRecord{29197402 #275 A=com.activitydemo U=0 sz=1}
Run #2: ActivityRecord{3bac5d43 u0 com.activitydemo/.MainActivity t275}
```

如果我们指定MainActivity的taskAffinity属性和SingleTaskActivity一样，又会出现什么情况呢。

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 277 hasCode:966292410
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo.singleTask
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleTaskActivity TaskId: 277 hasCode:149127957
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo.singleTask
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate OtherActivity TaskId: 277 hasCode:574150429
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onNewIntent=========================================================
07/com.activitydemo E/TAG: onNewIntent SingleTaskActivity TaskId: 277 hasCode:149127957
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo.singleTask
```

没错，就是和什么都不指定是一样的。

这时候，就有了下面的结论

- singleTask启动模式启动Activity时，首先会根据taskAffinity去寻找当前是否存在一个对应名字的任务栈
    - 如果不存在，则会创建一个新的Task，并创建新的Activity实例入栈到新创建的Task中去
    - 如果存在，则得到该任务栈，查找该任务栈中是否存在该Activity实例
        - 如果存在实例，则将它上面的Activity实例都出栈，然后回调启动的Activity实例的onNewIntent方法
        - 如果不存在该实例，则新建Activity，并入栈
        
此外，我们可以将两个不同App中的Activity设置为相同的taskAffinity，这样虽然在不同的应用中，但是Activity会被分配到同一个Task中去

我们再创建另外一个应用，指定它的taskAffinity和之前的一样，都是`com.activitydemo.singleTask`

```java
<activity 
    android:name=".OtherActivity"
    android:launchMode="singleTask"
    android:taskAffinity="com.activitydemo.singleTask"/>
```

然后启动一个应用，让他跳转到该Activity后，再按home键后台，启动另一个应用再进入该Activity，看日志

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 280 hasCode:966292410
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleTaskActivity TaskId: 281 hasCode:398173388
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo.singleTask
07/com.activitydemo.other E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo.other E/TAG: onCreate MainActivity TaskId: 284 hasCode:966292410
07/com.activitydemo.other E/TAG: taskAffinity:com.activitydemo.other
07/com.activitydemo.other E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo.other E/TAG: onCreate OtherActivity TaskId: 281 hasCode:398173388
07/com.activitydemo.other E/TAG: taskAffinity:com.activitydemo.singleTask
```

我们看到，指定了相同的taskAffinity的SingleTaskActivity和OtherActivity被启动到了同一个task中，taskId都为281。

### singleInstance模式

该模式具备singleTask模式的所有特性外，与它的区别就是，这种模式下的Activity会单独占用一个Task栈，具有全局唯一性，即整个系统中就这么一个实例。以singleInstance模式启动的Activity在整个系统中是单例的，如果在启动这样的Activiyt时，已经存在了一个实例，那么会把它所在的任务调度到前台，重用这个实例。

增加一个Activity，声明如下

```java
<activity 
    android:name=".singleinstance.SingleInstanceActivity"
    android:launchMode="singleInstance">
    <intent-filter>
        <action android:name="cn.edu.zafu.lifecycle"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</activity>
```

使用下面的方式分别在另一个应用中启动它

```java
Intent intent = new Intent();
intent.setAction("com.activitydemo");
startActivity(intent);
```

```java
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate MainActivity TaskId: 295 hasCode:966292410
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo E/TAG: onCreate SingleInstanceActivity TaskId: 296 hasCode:398173388
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
07/com.activitydemo.other E/TAG: ===========================================onCreate=========================================================
07/com.activitydemo.other E/TAG: onCreate MainActivity TaskId: 297 hasCode:966292410
07/com.activitydemo.other E/TAG: taskAffinity:com.activitydemo.other
07/com.activitydemo E/TAG: ===========================================onNewIntent=========================================================
07/com.activitydemo E/TAG: onNewIntent SingleInstanceActivity TaskId: 296 hasCode:398173388
07/com.activitydemo E/TAG: taskAffinity:com.activitydemo
```

我们看到，第一个应用显示打印了入口MainActivity的日志,然后启动SingleInstanceActivity时，由于系统中不存在该实例，所以新建了一个Task，按home键后，在启动另一个应用,先打印两一个应用的MainActivity,然后从另一个App启动SingleInstanceActivity，由于系统中已经存在了一个实例，不会再创建新的Task，直接复用该实例，并且回调onNewIntent方法。可以从他们的hashcode中可以看出这是同一个实例。

对以上内容的总结就是

SingleInstance模式启动的Activity在系统中具有全局唯一性。

## intentFilter的匹配规则

我们知道，启动Activity分为两种，显示调用和隐式调用，二者的区别这里就不多讲了，显示调用需要明确的指定被启动对象的组件信息，包括包名和类名，而隐式意图则不需要明确指定组件信息，原则上一个intent不应该即是显式调用又是隐式调用，如果二者共存的话以显式调用为主，显式调用很简单，这里主要介绍隐式调用，隐式调用需要intent能够匹配目标组件的`IntentFilter`中所设置的过滤信息，如果不匹配将无法启动目标Activity，`IntentFilter`中的过滤信息有`action`,`category`,`data`下面是一个过滤规则的示例：

```java
<activity
    android:name=".CodeActivity">
    <intent-filter>
        <action android:name="com.activitydemo.c" />
        <action android:name="com.activitydemo.d" />

        <category android:name="com.activitydemo.category.c" />
        <category android:name="com.activitydemo.category.d" />
        <category android:name="android.intent.category.DEFAULT" />
        
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```

为了匹配过滤列表，需要同时匹配过滤列表中的`action,category,data`信息，否则匹配失败，一个过滤列表中的action,category,data可以有多个，所有的action,category,data分别构成不同类别，同一类型的信息共同约束当前类别的匹配过程，只有一个intent同时匹配action类别,category类别,data类别才算是匹配完成，只有完全匹配才能成功启动目标Activity，另外一点，一个Activity中可以有多个`intent-filter`,一个Intent只要能匹配任何一组`intent-filter`即可成功启动对应的Activity,如下所示:

```java
<activity android:name=".ShareActivity">
    <!-- This activity handlers "SEND" actions with text data-->
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
    <!--This activity also handlers "SEND" and "SEND_MULTIPLE" with media data-->
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <action android:name="android.intent.action.SEND_MULTIPLE" />

        <category android:name="android.intent.category.DEFAULT" />

        <data android:mimeType="application/vnd.google.panorama360+jpg"/>
        <data android:mimeType="image/*" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```

下面详细分析各种属性的匹配规则

**1.action的匹配规则**

action是一个字符串，系统预定了一些action,同时我们也可以在应用中定义自己的action,action的匹配规则是intent中的action必须能够和过滤规则中的action匹配，这里说的匹配是指action的字符串值完全一样，一个过滤规则中的可以有多个action,那么只要intent中的action能够和过滤规则中的任何一个action相同即可匹配成功，针对上面的过滤规则，只要intent中的action的值为`com.activitydemo.c`或者`com.activitydemo.d`都能成功匹配,需要注意的是,Intent中如果没有指定action，那么匹配失败，总结一下，action的匹配要求就是intent中的action存在且必须和过滤规则中的其中一个action相同，这里需要注意它和category匹配规则的不同，另外，action区分大小写，大小写不同的字符串匹配也会失败

**2.category的匹配规则**

category是一个字符串，系统预定义了一些category，同时我们也可以在应用中定义自己的category。category的匹配规则和action不同，它要求Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。换句话说，Intent如果出现了category，不管有几个category，对于每个category来说，它必须是过滤规则中已经定义的category。当然，Intent中可以没有category，如果没有category的话，按照上面的描述，这个Intent仍然可以匹配成功。这里要注意下它和action匹配过程的不同，action是要求Intent中必须有一个action且必须能够和过滤规则中的某个action相同，而category要求Intent可以没有category，但是如果你一旦有category，不管有几个，每个都要能和过滤规则中的任何一个category相同。为了匹配前面的过滤规则中的category，我们可出下面的Intent,`intent.addcategory (“com.activitydemo.category.c”)`或者`intent.addcategory (“com.activitydemo.category.d)`亦或者不设category。为什么不设置category也可以匹配呢？原因是系统在调用startActivity或者startActivityForResult的时候会默认为Intent加上`“android.intent.category.DEFAULT”`这个category，所以这个category就可以匹配前面的过滤规则中的`第三个category`。同时，为了我们的activity能够接收隐式调用，就必须在`intent-filter`中指定`“android intent categor.DEFAULT”`这个category，原因刚才已经说明了。

**3.data匹配规则**

data的匹配规则和action有点类似，如果过滤规则中定义了data,那么intent中必须也要定义可匹配的data,在介绍data的匹配规则之前，我们需要来了解一下data的结构，因为data稍微有点复杂

```java
<data
   android:host="string"
   android:mimeType="string"
   android:path="string"
   android:pathPattern="string"
   android:pathPrefix="string"
   android:port="string"
   android:scheme="sstring" />
```

data由两部分组成，mimeType和URI，前者是媒体类型，比如image/jpeg,video/*等，可以表示图片,视频等不同的媒体格式，而URI包含的数据可就多了，下面的URI的结构：

```java
<scheme>://<host>"<port>/[<path>|<pathPrefix>|<pathPattern>]
```

这里再给几个实际的例子就好理解了

```java
content://com.liuguilin.project:200/folder/subfolder/etc
http://www.baidu.com:80/search/info
```

看了上面的两个例子你是不是瞬间就明白了，没错，就是这么简单，不过下面还是要一一介绍含义的：

- Scheme:URI的模式，比如http，file，content等，如果URI中没有指定的scheme,那么整个URI的其他参数无效，这也意味着URI无效
- Host:URI的主机，比如`www.google.com`,如果host未指定，那么整个URI中的其他参数无效，这也意味着URI无效
- Port:URI中的端口号，比如80，仅当URI中指定了scheme和host参数的时候port参数才有意义
- Path、pathPattem 和 pathPrefix：这三个参数表述路径信息，其中path表示完整的路径信息;pathPattern也表示完整的路径信息，但是它里面可以包含通配符`*`，`*`表示0个或多个任意字符，需要注意的是，由于正则表达式的规范，如果想表示真实的字符串，那么`*` 要写成`\\*`，`\`要写成`\\\\`;pathPrefix表示路径的前缀信息。

介绍完data的数据格式后，我们要说一下data的匹配规则了。前面说到，data的匹配规则和action类似，它也要求Intent中必须含有data数据，并且data数据能够完全匹配过滤规则中的某一个data.这里的完全匹配是指过滤规则中出现的data部分也出现在了Intent中的data中。下面分情况说明。

- (1) 如下过滤规则

```java
<intent-filter>
  <data android:mimeType="image/*"/>
</intent-filter>
```

这种规则指定了所有类型为图片，那么intent中的mineType属性必须为`image/*`才能匹配，这种情况下虽然过滤规则没有指定URI，但是却有默认值，URI的默认值为content和file,也就是说，虽然没有指定URI，但是Intent中的URI部分的scheme必须为content或者file才能匹配，这点事需要注意的，为了匹配(1)中的规则我们可以这样写：

```java
intent.setDataAndType(Uri.parse("file://abc"),"image/png");
```

另外，如果要为Intent指定完整的data，必须调用setDataAndType方法，不能先调用setData在调用setType,因为这两个方法彼此会清除对方的值，这个看源码就比较好理解了，比如setData:

```java
public Intent setData(Uri data) {
    mData = data;
    mType = null;
    return this;
}
```

可以发现，setData会把类型设置为null，同理setType 也会把URI置为null

- (2)如下规律规则

```java
<intent-filter>
        <data android:mimeType="video/mpeg" android:scheme="http" .../>
        <data android:mimeType="audio/mpeg" android:scheme="http" .../>
</intent-filter>
```

这种规则指定了两组data规则，且每个data都指出了完整的属性值，既有URI又有mimeType类型，为了匹配(2)中规则，我们这样写:

```java
intent.setDataAndType(Uri.parse("http://abc"),"video/png");
或者
intent.setDataAndType(Uri.parse("http://abc"),"audio/png"); 
```

通过上面的实例，我们应该知道了data的匹配规则，关于data还有一些特殊的情况需要说明一下，这也是他和action不同的地方,下面两种写法作用是一样的

```java
<intent-filter >
    <data android:scheme="file" android:host="www.baidu.com"/>
    ...
</intent-filter>

<intent-filter >
    <data android:scheme="file" />
    <data android:host="www.baidu.com"/>
    ...
</intent-filter>
```

到这里我们已经把IntentFilter的过滤规则都讲了一遍了，还记得本书前面给出的一个实例吗？现在我们给出完整的intent匹配规则:

```java
Intent intent = new Intent();
intent.addCategory("com.activitydemo.category.c");
intent.setDataAndType(Uri.parse("file//abc"),"text/plain");
startActivity(intent);
```

还记得URI中的scheme中的默认值吗？如果把上面的`intent.setDataAndType(Uri.parse(“file//abc”),”text/plain”)`;这句改成`intent.setDataAndType(Uri.parse(“http//abc”),”text/plain”)`;打开的actiivty就会报错，提示无法找到Activity，另外一点，intent-filter的匹配规则对于服务和广播也是同样的道理，不过系统对于Service的建议是尽量使用显式意图来启动服务。

最后，当我们通过隐式方式启动一个Activity的时候，可以做一下判断，看是否有Activity能够匹配我们的隐式Intent，如果不做判断就有可能出现上述的错误了。判断方法有两种：采用`PackageManager`的`resolveActivity`方法或者Intent的`resolveActivity`方法，如果它们找不到匹配的Activity就会返回null，我们通过判断返回值就可以规避上述错误了，另外，PackageManager还提供了`queryIntentActivities`方法，这个方法和resolveActivity方法法不同的是：它不是返回最佳匹配的Activity信息而是返回所有成功匹配的Activity信息，我们看一下queryIntentActivities和resolveActivity的方法原型：

```java
public abstract List<ResolveInfo>queryIntentActivities(Intent intent,int fladgs);
public abstract ResolveInfo resolveActivity(Intent intent,int flags);
```

上述两个方法的第一个参数比较好理解，第二个参数需要注意，我们要使用`MATCH_DEFAULT_ONLY`这个标记位，这个标记位的含义是仅仅匹配那些在intentfilter中声明了`<category android-name=”android.intent.category DEFAULT”>`这个category的Activity。使用这个标记位的意义在于，只要上述两个方法不返回null，那么`startActivity`一定可以成功，如果不用这个标记位，就可以把intent-filter中category不含DEFAULT的那些Activity给匹配出来，从而导致startActivity可能失败。因为不含有DEFAULT这个category的Activity是无法接收隐式Intent的。在action和category中，有一类action和category比较重要，他们是：

```java
<action android:name="android.intent.action.MAIN" />
<category android:name="android.intent.category.LAUNCHER" />
```

这二者共同作用是用来标明这是一个入口Activity并且会出现在系统的应用列表中,少了任何一个都没有实际意义，也无法出现在系统的应用列表中，也就是二者缺一不可，另外，针对Service和BroadcastReceiver,PackageManager同样提供了类似的方法去获取成功匹配的组件信息。