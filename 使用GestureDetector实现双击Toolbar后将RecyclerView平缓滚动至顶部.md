---
title: 使用GestureDetector实现双击Toolbar后将RecyclerView平缓滚动至顶部
date: 2017-03-24 15:04:31
tags:
---
>现在很多的App都会有双击`Toolbar`后将内容列表`RecyclerView`平缓滚动至顶部的功能，这个功能大大提升了用户体验，那么，该如何实现呢

实现这个功能的关键可以分为两点
1. 实现双击手势的监听
2. 实现`RecyclerView`的平滑滚动

第一点，实现双击手势监听，这个我当时走了一点点弯路，我当时想在onTouch里自己手动实现双击的判断，但是写了一半突然想到，google不可能想不到双击手势处理的这种场景，必然会有相关的类来处理的，于是搜了一下，果不其然，sdk里有个`GestureDetector`就是专门干这个的，不光双击、还有甩动、单击等等几个手势的处理，google果然很贴心，都替我们想好了。然后，就是第二点，实现`RecyclerView`平滑滚动到顶部，这里也有一个需要注意的点，那就是平滑滚动的触发条件，可以想象一下，如果现在`RecyclerView`已经显示到了第9999条数据（举个例子，实际可能遇不到这么大的数），这时要触发平滑滚动，从第9999慢慢的平滑滚动回第0，用户会崩溃的，所以，我们需要设置一个阈值，到了某一个值才开始平滑滚动，没到这个值就直接瞬间滚动到这个值，然后再开始平滑滚动，这样在用户看来就像是整个在平滑滚动，而且不会太慢。说了这么多，最终我们还是要用代码来实现它，那么我们还是来上代码吧，看看怎么实现`Toolbar`的双击监听：
```
//代码片段
final GestureDetector gestureDetector = new GestureDetector(SomeActivity.this, new GestureDetector.SimpleOnGestureListener() { //使用SimpleOnGestureListener可以只覆盖实现自己想要的手势
    @Override
    public boolean onDoubleTap(MotionEvent e) { //DoubleTap手势的处理
        LinearLayoutManager linearLayoutManager = (LinearLayoutManager) recyclerView.getLayoutManager();
        if (linearLayoutManager.findFirstCompletelyVisibleItemPosition() != 0) { //判断一下，如果现在RecyclerView不是已经显示了最顶部条目时，才需要做滚动到顶部的处理
            if (linearLayoutManager.getItemCount() > SMOOTH_SCROLL_THRESHOLD) { //SMOOTH_SCROLL_THRESHOLD就是上边我们说的那个触发平滑滚动的阈值，这里设置一个比较小的数就可以了，比如 10 ，当现在RecyclerView显示的数目大于这个阈值时就需要进行前边提到的“先瞬间滚动然后再平滑滚动的处理了”
                recyclerView.scrollToPosition(SMOOTH_SCROLL_THRESHOLD); //先瞬间滚动到THRESHOLD位置
            }
            recyclerView.smoothScrollToPosition(0); //然后再继续平滑滚动，这样既不会太慢又保留了平滑的效果
        }
        return super.onDoubleTap(e);
    }
});

toolbar.setOnTouchListener(new View.OnTouchListener() {
    @Override
    public boolean onTouch(View v, MotionEvent event) { //使用GestureDetector对Toolbar进行手势监听
                return gestureDetector.onTouchEvent(event);
    }
});
```