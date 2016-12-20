# Android 渲染机制 二

## 前言
> 前两节内容主要引自于 [Android UI性能优化实战 识别绘制中的性能问题](http://blog.csdn.net/lmj623565791/article/details/45556391)    
> 第三节内容主要引自于 [Android中RelativeLayout和LinearLayout性能分析](http://www.jianshu.com/p/8a7d059da746)   
> 该章节的主要目的是用于归纳总结 另：[google 原视频Youtube链接](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)

## 实战篇

> 第一篇介绍完渲染机制的基础信息，第二篇我们来讨论下如何才能优化我们的页面，达到良好的渲染效果
     
下面一张图很好的展示出了CPU和GPU的工作，以及潜在的问题，检测的工具和解决方案。 

![](http://img.blog.csdn.net/20150507094007117)

如果你对上图感到不理解，没关系，你只要知道问题：   

&#160; &#160; &#160; &#160;* 通过Hierarchy Viewer去检测渲染效率，去除不必要的嵌套   
&#160; &#160; &#160; &#160;* 通过Show GPU Overdraw去检测Overdraw，最终可以通过移除不必要的背景以及使用canvas.clipRect解决大多数问题。   

如果你还觉得不能理解，没关系，文本毕竟是枯燥的，那么以下结合实例来说明如何去优化     

### 1. Overdraw

> 对于性能优化，那么首先肯定是去发现问题，那么对么overdraw这个问题，还是比较容易发现的。    

按照以下步骤打开Show GPU Overrdraw的选项：  

&#160; &#160; &#160; &#160;设置 -> 开发者选项 -> 调试GPU过度绘制 -> 显示GPU过度绘制

好了，打开以后呢，你会发现屏幕上有各种颜色，此时你可以切换到需要检测的程序，对于各个色块，对比一张Overdraw的参考图：

![](http://static.oschina.net/uploads/img/201503/04080418_o2JX.png)

那么如果你发现你的app上深红色的色块比较多，那么可能就要注意了，接下来就开始说如果遇到overdraw的情况比较严重我们该则么处理。

#### 1.1 Overdraw 的处理方案一：移除不必要的background

下面看一个简单的展示ListView的例子：

* activity_main

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              android:paddingLeft="@dimen/activity_horizontal_margin"
              android:paddingRight="@dimen/activity_horizontal_margin"
              android:background="@android:color/white"
              android:paddingTop="@dimen/activity_vertical_margin"
              android:paddingBottom="@dimen/activity_vertical_margin"
              android:orientation="vertical">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="@dimen/narrow_space"
        android:textSize="@dimen/large_text_size"
        android:layout_marginBottom="@dimen/wide_space"
        android:text="@string/header_text"/>

    <ListView
        android:id="@+id/id_listview_chats"
        android:layout_width="match_parent"
        android:background="@android:color/white"
        android:layout_height="wrap_content"
        android:divider="@android:color/transparent"
        android:dividerHeight="@dimen/divider_height"/>
</LinearLayout>
```

* item的布局文件

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    android:paddingBottom="@dimen/chat_padding_bottom">

    <ImageView
        android:id="@+id/id_chat_icon"
        android:layout_width="@dimen/avatar_dimen"
        android:layout_height="@dimen/avatar_dimen"
        android:src="@drawable/joanna"
        android:layout_margin="@dimen/avatar_layout_margin" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@android:color/darker_gray"
        android:orientation="vertical">

        <RelativeLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:background="@android:color/white"
            android:textColor="#78A"
            android:orientation="horizontal">

            <TextView xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_alignParentLeft="true"
                android:padding="@dimen/narrow_space"
                android:text="@string/hello_world"
                android:gravity="bottom"
                android:id="@+id/id_chat_name" />

            <TextView xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_alignParentRight="true"
                android:textStyle="italic"
                android:text="@string/hello_world"
                android:padding="@dimen/narrow_space"
                android:id="@+id/id_chat_date" />
        </RelativeLayout>

        <TextView xmlns:android="http://schemas.android.com/apk/res/android"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:padding="@dimen/narrow_space"
            android:background="@android:color/white"
            android:text="@string/hello_world"
            android:id="@+id/id_chat_msg" />
    </LinearLayout>
</LinearLayout>
```

* Activity的代码


```
package com.zhy.performance_01_render;

import android.os.Bundle;
import android.os.PersistableBundle;
import android.support.annotation.Nullable;
import android.support.v7.app.AppCompatActivity;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.ArrayAdapter;
import android.widget.ImageView;
import android.widget.ListView;
import android.widget.TextView;

/**
 * Created by zhy on 15/4/29.
 */
public class OverDrawActivity01 extends AppCompatActivity
{
    private ListView mListView;
    private LayoutInflater mInflater ;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_overdraw_01);

        mInflater = LayoutInflater.from(this);
        mListView = (ListView) findViewById(R.id.id_listview_chats);

        mListView.setAdapter(new ArrayAdapter<Droid>(this, -1, Droid.generateDatas())
        {
            @Override
            public View getView(int position, View convertView, ViewGroup parent)
            {

                ViewHolder holder = null ;
                if(convertView == null)
                {
                    convertView = mInflater.inflate(R.layout.chat_item,parent,false);
                    holder = new ViewHolder();
                    holder.icon = (ImageView) convertView.findViewById(R.id.id_chat_icon);
                    holder.name = (TextView) convertView.findViewById(R.id.id_chat_name);
                    holder.date = (TextView) convertView.findViewById(R.id.id_chat_date);
                    holder.msg = (TextView) convertView.findViewById(R.id.id_chat_msg);
                    convertView.setTag(holder);
                }else
                {
                    holder = (ViewHolder) convertView.getTag();
                }

                Droid droid = getItem(position);
                holder.icon.setBackgroundColor(0x44ff0000);
                holder.icon.setImageResource(droid.imageId);
                holder.date.setText(droid.date);
                holder.msg.setText(droid.msg);
                holder.name.setText(droid.name);

                return convertView;
            }

            class ViewHolder
            {
                ImageView icon;
                TextView name;
                TextView date;
                TextView msg;
            }

        });
    }
}

```

* 实体的代码

```
package com.zhy.performance_01_render;

import java.util.ArrayList;
import java.util.List;

public class Droid
{
    public String name;
    public int imageId;
    public String date;
    public String msg;


    public Droid(String msg, String date, int imageId, String name)
    {
        this.msg = msg;
        this.date = date;
        this.imageId = imageId;
        this.name = name;
    }

    public static List<Droid> generateDatas()
    {
        List<Droid> datas = new ArrayList<Droid>();

        datas.add(new Droid("Lorem ipsum dolor sit amet, orci nullam cra", "3分钟前", -1, "alex"));
        datas.add(new Droid("Omnis aptent magnis suspendisse ipsum, semper egestas", "12分钟前", R.drawable.joanna, "john"));
        datas.add(new Droid("eu nibh, rhoncus wisi posuere lacus, ad erat egestas", "17分钟前", -1, "7heaven"));
        datas.add(new Droid("eu nibh, rhoncus wisi posuere lacus, ad erat egestas", "33分钟前", R.drawable.shailen, "Lseven"));

        return datas;
    }


}
```

现在的效果是：

![](http://img.blog.csdn.net/20150507094116038)

注意，我们的需求是整体是Activity是个白色的背景。 
 
开启Show GPU Overdraw以后：

![](http://img.blog.csdn.net/20150507094208657)

对比上面的参照图，可以发现一个简单的ListView展示Item，竟然很多地方被过度绘制了4X 。 那么，其实主要原因是由于该布局文件中存在很多不必要的背景，仔细看上述的布局文件，那么开始移除吧。
   
&#160; &#160; &#160; &#160;* 不必要的Background 1  

&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;我们主布局的文件已经是background为white了，那么可以移除ListView的白色背景

&#160; &#160; &#160; &#160;* 不必要的Background 2  

&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;Item布局中的LinearLayout的android:background="@android:color/darker_gray"
  
&#160; &#160; &#160; &#160;* 不必要的Background 3    

&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;Item布局中的RelativeLayout的android:background="@android:color/white"

&#160; &#160; &#160; &#160;* 不必要的Background 4   

&#160; &#160; &#160; &#160;&#160; &#160; &#160; &#160;Item布局中id为id_msg的TextView的android:background="@android:color/white"

这四个不必要的背景也比较好找，那么移除后的效果是：

![](http://img.blog.csdn.net/20150507094030982)

对比之前的是不是好多了~~~接下来还存在一些不必要的背景，你还能找到吗？   

&#160; &#160; &#160; &#160;* 不必要的Background 5  

&#160; &#160; &#160; &#160;这个背景比较难发现，主要需要看Adapter的getView的代码，上述代码你会发现，首先为每个icon设置了背景色（主要是当没有icon图的时候去显示），然后又设置了一个头像。那么就造成了overdraw，有头像的完全没必要去绘制背景，所有修改代码：    

```
Droid droid = getItem(position);
if(droid.imageId ==-1){
    holder.icon.setBackgroundColor(0x4400ff00);
    holder.icon.setImageResource(android.R.color.transparent);
 }else{
    holder.icon.setImageResource(droid.imageId);
    holder.icon.setBackgroundResource(android.R.color.transparent);
}
```

ok，还有最后一个，这个也是非常容易被忽略的。   

&#160; &#160; &#160; &#160;* 不必要的Background 6  

&#160; &#160; &#160; &#160;记得我们之前说，我们的这个Activity要求背景色是白色，我们的确在layout中去设置了背景色白色，那么这里注意下，我们的Activity的布局最终会添加在DecorView中，这个View会中的背景是不是就没有必要了，所以我们希望调用mDecor.setWindowBackground(drawable);，那么可以在Activity调用getWindow().setBackgroundDrawable(null);。

```
setContentView(R.layout.activity_overdraw_01);
getWindow().setBackgroundDrawable(null);    
```

ok，一个简单的listview显示item，我们一共找出了6个不必要的背景，现在再看最后的Show GPU Overdraw 与最初的比较。

<img src="http://img.blog.csdn.net/20150507094058251" width="300px">
<img src="http://img.blog.csdn.net/20150507094117548" width="300px">

ok，对比参照图，基本已经达到了最优的状态。

#### 1.2 Overdraw 的处理方案二：clipRect的妙用

我们在自定义View的时候，经常会由于疏忽造成很多不必要的绘制，比如大家看下面这样的图：

![](http://img.blog.csdn.net/20150507094218528)

&#160; &#160; &#160; &#160;多张卡片叠加，那么如果你是一张一张卡片从左到右的绘制，效果肯定没问题，但是叠加的区域肯定是过度绘制了。    
&#160; &#160; &#160; &#160;并且material design对于界面设计的新的风格更容易造成上述的问题。那么有什么好的方法去改善呢？    
&#160; &#160; &#160; &#160;答案是有的，android的Canvas对象给我们提供了很便利的方法clipRect就可以很好的去解决这类问题。

下面通过一个实例来展示，那么首先看一个效果图：

<img src="http://img.blog.csdn.net/20150507094257856" width="300px">
<img src="http://img.blog.csdn.net/20150507094315453" width="300px">


左边显示的时效果图，右边显示的是开启Show Override GPU之后的效果，可以看到，卡片叠加处明显的过度渲染了。
(ps:我对这个View添加了一个背景色~~仔细看下面的代码）

* View代码

```
package com.zhy.performance_01_render;

import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.graphics.Canvas;
import android.view.View;

/**
 * Created by zhy on 15/4/30.
 */
public class CardView extends View
{
    private Bitmap[] mCards = new Bitmap[3];

    private int[] mImgId = new int[]{R.drawable.alex, R.drawable.chris, R.drawable.claire};

    public CardView(Context context)
    {
        super(context);

        Bitmap bm = null;
        for (int i = 0; i < mCards.length; i++)
        {
            bm = BitmapFactory.decodeResource(getResources(), mImgId[i]);
            mCards[i] = Bitmap.createScaledBitmap(bm, 400, 600, false);
        }

        setBackgroundColor(0xff00ff00);
    }

    @Override
    protected void onDraw(Canvas canvas)
    {

        super.onDraw(canvas);

        canvas.save();
        canvas.translate(20, 120);
        for (Bitmap bitmap : mCards)
        {
            canvas.translate(120, 0);
            canvas.drawBitmap(bitmap, 0, 0, null);
        }
        canvas.restore();

    }
}
```

* Activity代码

```
/**
 * Created by zhy on 15/4/30.
 */
public class OverDrawActivity02 extends AppCompatActivity
{

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);

        setContentView(new CardView(this));
    }
}
```

那么大家可以考虑下如何去优化，其实很明显哈，我们上面已经说了使用cliprect方法，那么我们目标直指 
自定义View的onDraw方法。 
修改后的代码：

```
@Override
protected void onDraw(Canvas canvas){
    super.onDraw(canvas);

    canvas.save();
    canvas.translate(20, 120);
    for (int i = 0; i < mCards.length; i++){
        canvas.translate(120, 0);
        canvas.save();
        if (i < mCards.length - 1){
            canvas.clipRect(0, 0, 120, mCards[i].getHeight());
        }
        canvas.drawBitmap(mCards[i], 0, 0, null);
        canvas.restore();
    }
    canvas.restore();
}
```

分析得出，除了最后一张需要完整的绘制，其他的都只需要绘制部分；所以我们在循环的时候，给i到n-1都添加了clipRect的代码。

最后的效果图：

![](http://img.blog.csdn.net/20150507094536262)

可以看到，所有卡片变为了淡紫色，对比参照图，都是1X过度绘制，那么是因为我的View添加了一个 
ff00ff00的背景，可以说明已经是最优了。

如果你按照上面的修改，会发现最终效果图不是淡紫色，而是青色（2X），那是为什么呢？因为你还忽略了 
一个优化的地方，本View已经有了不透明的背景，完全可以移除Window的背景了，即在Activity中，添加getWindow().setBackgroundDrawable(null);代码。

### 2. 布局优化

> 布局方面：减少层级 > 同层级下用LinearLayout&FrameLayout 替代 RelativeLayout

#### 2.1 减少不必要的层次：巧用Hierarchy Viewer

* Hierarchy Viewer 在哪？
  
Android Studio 中：   

![](http://img.blog.csdn.net/20150507094357245)

* 如何在真机上使用Hierarchy Viewer？

> 出于安全考虑，Hierarchy Viewer只能连接Android开发版手机或是模拟器

推荐一种解决方案：romainguy在github上有个项目ViewServer，可以下载下来导入到IDE中，里面有个ViewServer的类，类注释上也标注了用法，在你希望调试的Activity以下该三个方法中，添加几行代码：

```
 * <pre>
 * public class MyActivity extends Activity {
 *     public void onCreate(Bundle savedInstanceState) {
 *         super.onCreate(savedInstanceState);
 *         // Set content view, etc.
 *         ViewServer.get(this).addWindow(this);
 *     }
 *       
 *     public void onDestroy() {
 *         super.onDestroy();
 *         ViewServer.get(this).removeWindow(this);
 *     }
 *   
 *     public void onResume() {
 *         super.onResume();
 *         ViewServer.get(this).setFocusedWindow(this);
 *     }
 * }
 * </pre>
```


* Hierarchy Viewer 怎么使用？

一图胜千言：

![](http://img.blog.csdn.net/20150507094619939)

**优化案例**

* 布局文件

```
<?xml version="1.0" encoding="utf-8"?>

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <!-- Version 1. Uses nested LinearLayouts -->
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="horizontal"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="@dimen/activity_vertical_margin">

        <ImageView
            android:id="@+id/chat_author_avatar1"
            android:layout_width="@dimen/avatar_dimen"
            android:layout_height="@dimen/avatar_dimen"
            android:layout_margin="@dimen/avatar_layout_margin"
            android:src="@drawable/joanna"/>

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="@string/line1_text" />

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="@string/line2_text"/>
        </LinearLayout>
    </LinearLayout>


    <!-- Version 2: uses a single RelativeLayout -->
    <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="horizontal"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="@dimen/activity_vertical_margin">

        <ImageView
            android:id="@+id/chat_author_avatar2"
            android:layout_width="@dimen/avatar_dimen"
            android:layout_height="@dimen/avatar_dimen"
            android:layout_margin="@dimen/avatar_layout_margin"
            android:src="@drawable/joanna"/>


        <TextView
            android:id="@+id/rl_line1"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_toRightOf="@id/chat_author_avatar2"
            android:text="@string/line1_text" />

        <TextView
            android:id="@+id/rl_line2"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_below="@id/rl_line1"
            android:layout_toRightOf="@id/chat_author_avatar2"
            android:text="@string/line2_text" />
    </RelativeLayout>
</LinearLayout>
```

* Activity

```
package com.example.liqiuzuo.demo;

import android.os.Bundle;
import android.support.v7.app.ActionBarActivity;

import com.android.debug.hv.ViewServer;


public class CompareLayoutActivity extends ActionBarActivity
{

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_compare_layouts);

        ViewServer.get(this).addWindow(this);
    }

    @Override
    protected void onResume()
    {
        super.onResume();
        ViewServer.get(this).setFocusedWindow(this);
    }

    @Override
    protected void onDestroy()
    {
        super.onDestroy();
        ViewServer.get(this).removeWindow(this);
    }
}
```

然后手机打开此Activity，打开Android Device Moniter，切换到Hierarchy Viewer视图，可以看到：    

![](http://img.blog.csdn.net/20150507094652246)

&#160; &#160; &#160; &#160;点击LinearLayout，然后点击Profile Node，你会发现所有的子View上面都有了3个圈圈 
（取色范围为红、黄、绿色），这三个圈圈分别代表measure 、layout、draw的速度，并且你也可以看到实际的运行的速度，如果你发现某个View上的圈是红色，那么说明这个View相对其他的View，该操作运行最慢，注意只是相对别的View，并不是说就一定很慢。

&#160; &#160; &#160; &#160;红色的指示能给你一个判断的依据，具体慢不慢还是需要你自己去判断的。

&#160; &#160; &#160; &#160;好了，上面的布局文件展示了ListView的Item的编写的两个版本，一个是Linearlayout嵌套的，一个是RelativeLayout的。上图也可以看出Linearlayout的版本相对RelativeLayout的版本要慢很多（请多次点击Profile Node取样）。即可说明RelativeLayout的版本更优于RelativeLayout的写法，并且Hierarchy Viewer可以帮助我们发现类似的问题。

&#160; &#160; &#160; &#160;恩，对了，第一个例子里面的ListView的Item的写法就是Liearlayout嵌套的，大家有兴趣可以修改为RelativeLayout的写法的。

#### 2.2 使用LinearLayout&FrameLayout 替代 RelativeLayout

**案例**

* 布局文件

```
<?xml version="1.0" encoding="utf-8"?>

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <!-- Version 1. Uses single LinearLayout -->
    <LinearLayout
        android:orientation="horizontal"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="@dimen/activity_vertical_margin">

        <ImageView
            android:id="@+id/chat_author_avatar1"
            android:layout_width="@dimen/avatar_dimen"
            android:layout_height="@dimen/avatar_dimen"
            android:layout_margin="@dimen/avatar_layout_margin"
            android:src="@drawable/img_computer" />

        <TextView
            android:layout_width="@dimen/avatar_dimen"
            android:layout_height="@dimen/avatar_dimen"
            android:layout_margin="@dimen/avatar_layout_margin"
            android:text="@string/line1_text" />

    </LinearLayout>


    <!-- Version 2: uses a single RelativeLayout -->
    <RelativeLayout
        android:orientation="horizontal"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="@dimen/activity_vertical_margin">

        <ImageView
            android:id="@+id/chat_author_avatar2"
            android:layout_width="@dimen/avatar_dimen"
            android:layout_height="@dimen/avatar_dimen"
            android:layout_margin="@dimen/avatar_layout_margin"
            android:src="@drawable/img_computer" />


        <TextView
            android:layout_width="@dimen/avatar_dimen"
            android:layout_height="@dimen/avatar_dimen"
            android:layout_margin="@dimen/avatar_layout_margin"
            android:layout_toRightOf="@id/chat_author_avatar2"
            android:text="@string/line1_text" />

    </RelativeLayout>
</LinearLayout>
```

* Activity

```
package com.example.liqiuzuo.demo;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;

import com.android.debug.hv.ViewServer;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ViewServer.get(this).addWindow(this);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        ViewServer.get(this).removeWindow(this);
    }

    @Override
    public void onResume() {
        super.onResume();
        ViewServer.get(this).setFocusedWindow(this);
    }
}
```

执行效果如下：

<img src="https://raw.githubusercontent.com/liqiuzuo/MarkdownPhotos/master/Res/linearlayout.png" width="300px">
<img src="https://raw.githubusercontent.com/liqiuzuo/MarkdownPhotos/master/Res/relativelayout.png" width="300px">

消耗时间上 relativeLayout > linearLayout 其中差距主要是在 measure 的时间消耗上。这是为什么呢？

* **Measure**

**RelativeLayout的onMeasure()方法：**

```
View[] views = mSortedHorizontalChildren;
int count = views.length;

for (int i = 0; i < count; i++) {
   View child = views[i];
   if (child.getVisibility() != GONE) {
      LayoutParams params = (LayoutParams) child.getLayoutParams();
      int[] rules = params.getRules(layoutDirection);

      applyHorizontalSizeRules(params, myWidth, rules);
      measureChildHorizontal(child, params, myWidth, myHeight);

      if (positionChildHorizontal(child, params, myWidth, isWrapContentWidth)) {
          offsetHorizontalAxis = true;
      }
   }
}

views = mSortedVerticalChildren;
count = views.length;
final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;
    
for (int i = 0; i < count; i++) {
  View child = views[i];
  if (child.getVisibility() != GONE) {
     LayoutParams params = (LayoutParams) child.getLayoutParams();

     applyVerticalSizeRules(params, myHeight);
     measureChild(child, params, myWidth, myHeight);
     if (positionChildVertical(child, params, myHeight, isWrapContentHeight)) {
        offsetVerticalAxis = true;
     }

     if (isWrapContentWidth) {
        if (isLayoutRtl()) {
          if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
             width = Math.max(width, myWidth - params.mLeft);
          } else {
             width = Math.max(width, myWidth - params.mLeft - params.leftMargin);
          }
       } else {
          if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
              width = Math.max(width, params.mRight);
          } else {
              width = Math.max(width, params.mRight + params.rightMargin);
          }
       }
    }

     if (isWrapContentHeight) {
        if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
           height = Math.max(height, params.mBottom);
        } else {
           height = Math.max(height, params.mBottom + params.bottomMargin);
        }
    }

     if (child != ignore || verticalGravity) {
          left = Math.min(left, params.mLeft - params.leftMargin);
          top = Math.min(top, params.mTop - params.topMargin);
      }

      if (child != ignore || horizontalGravity) {
          right = Math.max(right, params.mRight + params.rightMargin);
          bottom = Math.max(bottom, params.mBottom + params.bottomMargin);
      }
   }
}
```

根据源码我们发现RelativeLayout会对子View做两次measure。这是为什么呢？首先RelativeLayout中子View的排列方式是基于彼此的依赖关系，而这个依赖关系可能和布局中View的顺序并不相同，在确定每个子View的位置的时候，就需要先给所有的子View排序一下。又因为RelativeLayout允许A，B 2个子View，横向上B依赖A，纵向上A依赖B。所以需要横向纵向分别进行一次排序测量。

**LinearLayout的onMeasure()方法：**

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  if (mOrientation == VERTICAL) {
    measureVertical(widthMeasureSpec, heightMeasureSpec);
  } else {
    measureHorizontal(widthMeasureSpec, heightMeasureSpec);
  }
}
```

与RelativeLayout相比LinearLayout的measure就简单明了的多了，先判断线性规则，然后执行对应方向上的测量。随便看一个吧。

```
   for (int i = 0; i < count; ++i)

    {
        final View child = getVirtualChildAt(i);

        if (child == null) {
            mTotalLength += measureNullChild(i);
            continue;
        }

        if (child.getVisibility() == View.GONE) {
            i += getChildrenSkipCount(child, i);
            continue;
        }

        if (hasDividerBeforeChildAt(i)) {
            mTotalLength += mDividerHeight;
        }

        LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();

        totalWeight += lp.weight;

        if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
            // Optimization: don't bother measuring children who are going to use
            // leftover space. These views will get measured again down below if
            // there is any leftover space.
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
        } else {
            int oldHeight = Integer.MIN_VALUE;

            if (lp.height == 0 && lp.weight > 0) {
                // heightMode is either UNSPECIFIED or AT_MOST, and this
                // child wanted to stretch to fill available space.
                // Translate that to WRAP_CONTENT so that it does not end up
                // with a height of 0
                oldHeight = 0;
                lp.height = LayoutParams.WRAP_CONTENT;
            }

            // Determine how big this child would like to be. If this or
            // previous children have given a weight, then we allow it to
            // use all available space (and we will shrink things later
            // if needed).
            measureChildBeforeLayout(
                    child, i, widthMeasureSpec, 0, heightMeasureSpec,
                    totalWeight == 0 ? mTotalLength : 0);

            if (oldHeight != Integer.MIN_VALUE) {
                lp.height = oldHeight;
            }

            final int childHeight = child.getMeasuredHeight();
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                    lp.bottomMargin + getNextLocationOffset(child));

            if (useLargestChild) {
                largestChildHeight = Math.max(childHeight, largestChildHeight);
            }
        }
    }
```

&#160; &#160; &#160; &#160;父视图在对子视图进行measure操作的过程中，使用变量mTotalLength保存已经measure过的child所占用的高度，该变量刚开始时是0。在for循环中调用measureChildBeforeLayout（）对每一个child进行测量，该函数实际上仅仅是调用了measureChildWithMargins(),在调用该方法时，使用了两个参数。其中一个是heightMeasureSpec，该参数为LinearLayout本身的measureSpec；另一个参数就是mTotalLength，代表该LinearLayout已经被其子视图所占用的高度。 每次for循环对child测量完毕后，调用child.getMeasuredHeight()获取该子视图最终的高度，并将这个高度添加到mTotalLength中。**在本步骤中，暂时避开了lp.weight>0的子视图，即暂时先不测量这些子视图，因为后面将把父视图剩余的高度按照weight值的大小平均分配给相应的子视图。**源码中使用了一个局部变量totalWeight累计所有子视图的weight值。处理lp.weight>0的情况需要注意，如果变量heightMode是EXACTLY，那么，当其他子视图占满父视图的高度后，weight>0的子视图可能分配不到布局空间，从而不被显示，只有当heightMode是AT_MOST或者UNSPECIFIED时，weight>0的视图才能优先获得布局高度。最后我们的结论是：如果不使用weight属性，LinearLayout会在当前方向上进行一次measure的过程，如果使用weight属性，LinearLayout会避开设置过weight属性的view做第一次measure，完了再对设置过weight属性的view做第二次measure。由此可见，weight属性对性能是有影响的，而且本身有大坑，请注意避让。

&#160; &#160; &#160; &#160;从源码中我们似乎能看出，我们先前的测试结果中RelativeLayout不如LinearLayout快的根本原因是RelativeLayout需要对其子View进行两次measure过程。而LinearLayout则只需一次measure过程，所以显然会快于RelativeLayout，但是如果LinearLayout中有weight属性，则也需要进行两次measure，但即便如此，应该仍然会比RelativeLayout的情况好一点。

* **RelativeLayout另一个性能问题**

对比到这里就结束了嘛？显然没有！我们再看看View的Measure（）方法都干了些什么？

```
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
        widthMeasureSpec != mOldWidthMeasureSpec ||
        heightMeasureSpec != mOldHeightMeasureSpec) {
                     ......
    }
    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;

    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
        (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
}
```

View的measure方法里对绘制过程做了一个优化，如果我们或者我们的子View没有要求强制刷新，而父View给子View的传入值也没有变化（也就是说子View的位置没变化），就不会做无谓的measure。但是上面已经说了RelativeLayout要做两次measure，而在做横向的测量时，纵向的测量结果尚未完成，只好暂时使用myHeight传入子View系统，假如子View的Height不等于（设置了margin）myHeight的高度，那么measure中上面代码所做得优化将不起作用，这一过程将进一步影响RelativeLayout的绘制性能。而LinearLayout则无这方面的担忧。解决这个问题也很好办，如果可以，尽量使用padding代替margin。

* **总结**

1. **RelativeLayout会让子View调用2次onMeasure，LinearLayout 在有weight时，也会调用子View2次onMeasure**
2. **RelativeLayout的子View如果高度和RelativeLayout不同，则会引发效率问题，当子View很复杂时，这个问题会更加严重。如果可以，尽量使用padding代替margin。**
3. **在不影响层级深度的情况下,使用LinearLayout和FrameLayout而不是RelativeLayout**
