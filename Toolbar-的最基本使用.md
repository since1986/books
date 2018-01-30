---
title: Toolbar 的最基本使用
date: 2017-03-22 14:56:09
tags:
---
>`Toolbar`是`ActionBar`的替代者，对于刚刚入门的Android开发者，也许会对怎样使用`Toolbar`感到困惑(比如那时的我。。。)，所以我把`Toolbar`的最最基础的使用方法简单写一下，希望能帮到刚入门的朋友

</br>
**如何去掉默认的`ActionBar`：**
    在`styles.xml`中创建一个主题`AppTheme.NoActionBar`

 ```
<!-- NoActionBar -->
<style name="AppTheme.NoActionBar" parent="AppTheme">
    <item name="windowActionBar">false</item>
    <item name="windowNoTitle">true</item>
</style>
```
将这个主题应用到`application`上，即可去除所有Activity默认的ActionBar
 ```
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme.NoActionBar"> <!-- 去掉默认的ActionBar -->
```
</br>
**如何修改`Toolbar`默认的文字颜色：**
在`style.xml`中添加主题
 ```
<style name="AppTheme.Toolbar">
    <item name="colorControlNormal">#FFEFEFEF</item>
    <item name="android:textColorPrimary">#FFEFEFEF</item>
</style>

<style name="AppTheme.Toolbar.Popup">
     <item name="android:textColorPrimary">@android:color/black</item>
</style>
```
然后将主题应用到Toolbar上
```
<!-- 改变文字颜色 -->
<android.support.v7.widget.Toolbar
      android:id="@+id/your_toolbar"
      android:layout_width="match_parent"
      android:layout_height="?attr/actionBarSize"
      app:popupTheme="@style/AppTheme.Toolbar.Popup"
      app:theme="@style/AppTheme.Toolbar" />
```
</br> 
**如何加上返回按钮并实现返回：**
```
Toolbar toolbar = (Toolbar) findViewById(R.id.your_toolbar);
setSupportActionBar(toolbar);
getSupportActionBar().setDisplayHomeAsUpEnabled(true); //启用返回按钮
```

```
//实现返回功能
@Override
public boolean onOptionsItemSelected(MenuItem item) {

     switch (item.getItemId()) {
          case android.R.id.home: //android.R.id.home是Android内置home按钮的id
               finish();
               break;
     }
     return super.onOptionsItemSelected(item);
}
```