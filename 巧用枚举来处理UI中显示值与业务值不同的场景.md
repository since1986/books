---
title: 巧用枚举来处理UI中显示值与业务值不同的场景
date: 2017-03-22 15:03:16
tags:
---
>在Android中，经常会遇到一些在UI上显示的值与实际业务需要的值不一致的场景，这时就是枚举发挥用武之地的时候了

看下图所示的这个场景：
</br>
{% asset_img 23d4cfa37729e611a728.png %}

这个场景是一个类似于web中`<select>`的场景（图中这个下拉组件是我自己写的一个自定义View，用于替换SDK内置的Spinner），从这个场景中不难看出，我们在UI中需要显示的值和业务逻辑中需要的值是不一样的（后端给的接口定义了一组数字来作为参数），我们不能直接把UI中的“正面”这两个字作为参数传给业务逻辑的方法，而应传递一个对应于“正面”的值，该如何实现这个场景呢，这时候**枚举**就该登场了

`Talk is cheap. Show me the code.`

```
//代码片段1
public enum Relativity {

    //直接使用中文来给枚举命名，从而利用继承自父类的 .toString() 来返回UI需要的值
    全部 {
        @Override
        public String value() {
            return "";
        }

    }, 正面 {
        @Override
        public String value() {
            return "1";
        }

    }, 中性 {
        @Override
        public String value() {
            return "0";
        }

    }, 负面 {
        @Override
        public String value() {
            return "-1";
        }

    };

    public abstract String value(); //定义一个抽象方法让子类来实现，这个方法的返回值是业务逻辑中需要用到的值
}
```

```
//代码片段2 （这个 bindData 是我自定义View里的绑定数据的方法，实际上里面是调用了ArrayAdapter的addAll(T... items)）
spinnerRelativity.bindData(Relativity.values()); //直接使用枚举的 .values() 返回所有此类枚举所组成的数组作为参数传给UI
```

```
//代码片段3 
spinnerRelativity.setOnValueChangeListener(new OnValueChangeListener() {

    @Override
    public void onValueChange(View view, CharSequence originalValue, CharSequence newValue) {
        relativity = Relativity.valueOf((String) newValue) //onValueChange 是我自定义View里的一个回调方法，如图所示，当用户选中“全部”时方法的第二个CharSequence参数“newValue”的值就是一个字符串 "全部"， 这时，利用枚举的 valueOf(String s) 方法就可以从这个字符串得到对应的枚举对象
                .value(); //得到枚举对象后再调用枚举中自定义的 .value() 获得业务逻辑所需要的值
        
                //此处是你的业务逻辑
    }
});
```