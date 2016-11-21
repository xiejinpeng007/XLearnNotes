# 仿饿了么动画

 最近项目Release完毕，闲暇之余给公司内部的小卖部app升下级(一个人撸完了design+code)，添加了一个商城功能，因为每天都用饿了么点外卖，比较喜欢饿了么点餐落入购物车的动画，所以说自己实现了一个，做一点微小的笔记。
 
 ![](https://github.com/xiejinpeng007/LearnNotes/blob/master/ElemeAnim/elemeanim.gif)  
 
 整个界面相关元素有RecyclerView + FloatActionButton，动画是点击item出现一个图标以抛物线落入购物车。
 
 整体的思路是：  
 1. 点击itemView获取到相关location[]、height、width、position参数   
 2. 在View层中拿到的参数初始化动画View、ViewGroup  
 3. 初始化动画、开始动画
   
### Step1:  RecyclerView.Adapter类
```
   /**
     * 在RecyclerView.Adapter中的itemView添加点击事件，用于获取这个itemView在窗口（Window）中的位置信息(location[])。
     * 这里传递的参数还包括itemView的宽高和position，用于更细致的调整动画的初始位置和更新相关数据。
     * 这里发送点击事件没有使用接口而是用了RxBus调用startAddToCartAnim()，效果和平时使用的接口一致。
     */

holder.itemView.setOnClickListener(v -> {
            int height = holder.itemView.getHeight();
            int width = holder.itemView.getWidth();
            int[] startLocation = new int[2];
            holder.itemView.getLocationInWindow(startLocation);
            String name = list.get(holder.getLayoutPosition()).getName();
            RxBus.getInstance().send(new F01_01_ShopFragment.OnRecyclerItemClickEvent(startLocation, height, width, holder.getLayoutPosition(), name));
        });
```

### Step2: Activity/Fragment类

```
    private void startAddToCartAnim(int[] startLocation, int height, int width) {
        //初始化startLocation、endLocation坐标
        int[] endLocation = new int[2];
        orderFloatingactionbutton.getLocationInWindow(endLocation);

        //初始化动画以及parentLayout的参数
        ImageView animView = new ImageView(getContext());
        animView.setImageResource(R.drawable.ic_card_giftcard_red_400_36dp);
        LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT);
        params.leftMargin = startLocation[0] + width / 2;
        params.topMargin = startLocation[1];
        //获取parentLayout以及添加动画view到parentLayout
        getAnimLayout().addView(animView, params);

        //设定动画类型（使用简单的TranslateAnimation，通过不同的Interpolator来达到抛物线效果）
        TranslateAnimation animationX = new TranslateAnimation(0, endLocation[0] - startLocation[0] - width / 2, 0, 0);
        animationX.setInterpolator(new LinearInterpolator());
        animationX.setFillAfter(true);
        TranslateAnimation animationY = new TranslateAnimation(0, 0, 0, endLocation[1] - startLocation[1]);
        animationY.setInterpolator(new AccelerateInterpolator());
        animationY.setFillAfter(true);
        AlphaAnimation alphaAnimation = new AlphaAnimation(1, 0.5f);
        AnimationSet set = new AnimationSet(false);
        set.addAnimation(animationX);
        set.addAnimation(animationY);
        set.addAnimation(alphaAnimation);
        set.setDuration(500);
        //执行动画
        animView.startAnimation(set);

        set.setAnimationListener(new Animation.AnimationListener() {
            @Override
            public void onAnimationStart(Animation animation) {

            }

            @Override
            public void onAnimationEnd(Animation animation) {
                animView.setVisibility(View.GONE);
            }

            @Override
            public void onAnimationRepeat(Animation animation) {

            }
        });

    }
    
    /**
     * 获取点击动画所在的parentLayout(因为动画的坐标用的getLocationInWindow获取，所以这里也以DecorView的区域为parentLayout)
     * 这里的animLayout和animView的实例不能复用
     */

    private ViewGroup getAnimLayout() {
        ViewGroup rootView = (ViewGroup) getActivity().getWindow().getDecorView();
        LinearLayout animLayout = new LinearLayout(getContext());
        LinearLayout.LayoutParams lp = new LinearLayout.LayoutParams(
                LinearLayout.LayoutParams.MATCH_PARENT,
                LinearLayout.LayoutParams.MATCH_PARENT);
        animLayout.setLayoutParams(lp);
        animLayout.setId(Integer.MAX_VALUE - 1);
        animLayout.setBackgroundResource(android.R.color.transparent);
        rootView.addView(animLayout);
        return animLayout;
    }
```
* 补充：  
这里以item中间为坐标开始动画，以最精简的代码实现了基本动画，开发者可以以此为基础实现更复杂更精细的动画。  
比如：还可以根据location[]、height、width调整动画位置或是重写OnTouchListener根据触摸位置设定开始动画。
