#### 得到LayoutInflater

```java
public static LayoutInflater from(@UiContext Context context) {
  	//就是个系统服务
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

在SystemServiceRegistry类中可以得知，LayoutInflater的实际类型为：PhoneLayoutInflater

```java
static{}@PhoneLayoutInflater

registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
        new CachedServiceFetcher<LayoutInflater>() {
    @Override
    public LayoutInflater createService(ContextImpl ctx) {
        return new PhoneLayoutInflater(ctx.getOuterContext());
    }});
```

#### inflate@LayoutInflater

```java
----> inflate@LayoutInflater:

public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
  	//同步
    synchronized (mConstructorArgs) {
        ...

        try {
          	//移到START_TAG
            advanceToRootNode(parser);
          	
            final String name = parser.getName();
            ...

            if (TAG_MERGE.equals(name)) {
              	//merge处理，root不能为null，attachToRoot必须为true
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
								//将merge下的view依次添加到root
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                //直接创建View
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ...

                ViewGroup.LayoutParams params = null;
								
                if (root != null) {
                    ...
                  	//得到temp的params
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        //attachToRoot = false时，设置
                        temp.setLayoutParams(params);
                    }
                }

                ...

                // 解析子View
                rInflateChildren(parser, temp, attrs, true);

               	...
                if (root != null && attachToRoot) {
                  	//attachToRoot = true时，将temp添加到root
                    root.addView(temp, params);
                }

                ...
                //没有root，或者有root，attachToRoot = false，时返回temp
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
				...

        return result;
    }
}
```

#### rInflate@LayoutInflater

```java
----> rInflate@LayoutInflater:

void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;

    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {
				
      	//从START_TAG开始
        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();

        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
          	//include不能是根
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
          	//include处理
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
          	//merge必须是根
            throw new InflateException("<merge /> must be the root element");
        } else {
          	//创建view
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
          	//子view解析
            rInflateChildren(parser, view, attrs, true);
          	//添加到viewGroup
            viewGroup.addView(view, params);
        }
    }

    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }
		
    if (finishInflate) {
      	//解析完成
        parent.onFinishInflate();
    }
}
```

#### rInflateChildren@LayoutInflater

```java
----> rInflateChildren@LayoutInflater:

final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
  	//直接调用rInflate
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

#### createViewFromTag@LayoutInflater

```java
----> createViewFromTag@LayoutInflater:

View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    ...

    try {
      	//尝试通过factory创建view
        View view = tryCreateView(parent, name, context, attrs);
				
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
              	//判断view是sdk自带，还是自定义的
                if (-1 == name.indexOf('.')) {
                  	//没有. sdk自带的view，这里会调用到两个参数的onCreateView
                    //在PhoneLayoutInflater中会有全类名补齐
                  	/*
                  	onCreateView(context, parent, name, attrs)@LayoutInflater

                    onCreateView(parent, name, attrs)@LayoutInflater

                    onCreateView(name, attrs)@PhoneLayoutInflater

                    createView(name, prefix, attrs)@LayoutInflater（prefix默认"android.view."）

                    createView(context, name, prefix, attrs)@LayoutInflater
                  	*/
                    view = onCreateView(context, parent, name, attrs);
                } else {
                  	//自定义view
                    view = createView(context, name, null, attrs);
                }
              	//最终都会调用4个参数的createView
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    ...
}
```

#### tryCreateView@LayoutInflater

```java
----> tryCreateView@LayoutInflater:

public final View tryCreateView(@Nullable View parent, @NonNull String name,
    @NonNull Context context,
    @NonNull AttributeSet attrs) {
    ...

    //依次尝试调用mFactory2，mFactory，mPrivateFactory的onCreateView方法创建view
    View view;
    if (mFactory2 != null) {
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }

    if (view == null && mPrivateFactory != null) {
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }
		
    return view;
}
```

#### onCreateView@PhoneLayoutInflater

```java
----> onCreateView@PhoneLayoutInflater:

private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
    };

@Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
    for (String prefix : sClassPrefixList) {
        try {
          	//系统自带的view每个创建的时候都可能在这里执行三次？？？
            View view = createView(name, prefix, attrs);
            if (view != null) {
                return view;
            }
        } catch (ClassNotFoundException e) {
            // In this case we want to let the base class take a crack
            // at it.
        }
    }
		//全部创建失败，调用LayoutInflater中的onCreateView
    return super.onCreateView(name, attrs);
}
```

#### createView@LayoutInflater

```java
----> createView@LayoutInflater

static final Class<?>[] mConstructorSignature = new Class[] {
      Context.class, AttributeSet.class};

@Nullable
public final View createView(@NonNull Context viewContext, @NonNull String name,
        @Nullable String prefix, @Nullable AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
  	...
    //classLoad必须是LayoutInflater的classLoad或者context的classLoad？？？？？？？？？
 		Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;

    try {
        ...

        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
            clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                    mContext.getClassLoader()).asSubclass(View.class);
            ...
          	//得到双参构造器
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
          	//缓存构造器
            sConstructorMap.put(name, constructor);
        } else {
            ...
        }

        Object lastContext = mConstructorArgs[0];
        mConstructorArgs[0] = viewContext;
        Object[] args = mConstructorArgs;
        args[1] = attrs;

        try {
          	//通过构造器，反射创建view
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            return view;
        } finally {
            mConstructorArgs[0] = lastContext;
        }
    }
}
```

#### parseInclude@LayoutInflater

```java
----> parseInclude@LayoutInflater:

private void parseInclude(XmlPullParser parser, Context context, View parent,
        AttributeSet attrs) throws XmlPullParserException, IOException {
    ...
    if (precompiled == null) {
        final XmlResourceParser childParser = context.getResources().getLayout(layout);

        try {
            ...

            if (TAG_MERGE.equals(childName)) {
                // The <merge> tag doesn't support android:theme, so
                // nothing special to do here.
                rInflate(childParser, parent, context, childAttrs, false);
            } else {
                final View view = createViewFromTag(parent, childName,
                    context, childAttrs, hasThemeOverride);
                final ViewGroup group = (ViewGroup) parent;

                final TypedArray a = context.obtainStyledAttributes(
                    attrs, R.styleable.Include);
              	//拿到include的id，没有就是NO_ID
                final int id = a.getResourceId(R.styleable.Include_id, View.NO_ID);
                final int visibility = a.getInt(R.styleable.Include_visibility, -1);
                a.recycle();

                ViewGroup.LayoutParams params = null;
                try {
                    params = group.generateLayoutParams(attrs);
                } catch (RuntimeException e) {
                    // Ignore, just fail over to child attrs.
                }
                if (params == null) {
                    params = group.generateLayoutParams(childAttrs);
                }
                view.setLayoutParams(params);

                rInflateChildren(childParser, view, childAttrs, true);
              
								//如果id不是NO_ID，将id设置给include布局的根View
                if (id != View.NO_ID) {
                    view.setId(id);
                }
              	//所以，在include设置了id时，include的layout中的根view的id会被覆盖

                switch (visibility) {
                    case 0:
                        view.setVisibility(View.VISIBLE);
                        break;
                    case 1:
                        view.setVisibility(View.INVISIBLE);
                        break;
                    case 2:
                        view.setVisibility(View.GONE);
                        break;
                }

                group.addView(view);
            }
        } finally {
            childParser.close();
        }
    }
    LayoutInflater.consumeChildElements(parser);
}
```

#### 使用情况

```java
private fun layoutInflater(root: ViewGroup){
    val inflater = LayoutInflater.from(this)
    val mergeLayoutId = "merge_layout.xml".toInt()
    //merge布局必须要有root
    inflater.inflate(mergeLayoutId, root)

    val normalLayoutId = "normal_layout.xml".toInt()
    //root==null时，xml布局中根控件的属性无效
    inflater.inflate(mergeLayoutId, null)

    //root != null，attachToRoot=false时，宽高有效，且不会添加到root
    inflater.inflate(normalLayoutId, root, false)

    //root != null，attachToRoot=true时，宽高有效，且会添加到root
    inflater.inflate(normalLayoutId, root, true)
}
```

#### merge，include，ViewStub

```java
include:
1. 不能作为根元素，需要放在 ViewGroup中
2. findViewById查找不到目标控件，这个问题出现的前提是在使用include时设置了id，而在findViewById时却用了被include进来的布局的根元素id。
   1. 为什么会报空指针呢？
   如果使用include标签时设置了id，这个id就会覆盖 layout根view中设置的id，从而找不到这个id
   代码：LayoutInflate.parseInclude 
   			--》final int id = a.getResourceId(R.styleable.Include_id, View.NO_ID);
   			--》if (id != View.NO_ID) {
                    view.setId(id);
                } 

merge:
1. merge标签必须使用在根布局
2. 因为merge标签并不是View,所以在通过LayoutInflate.inflate()方法渲染的时候,第二个参数必须指定一个父容器,且第三个参数必须为true,也就是必须为merge下的视图指定一个父亲节点.
3. 由于merge不是View所以对merge标签设置的所有属性都是无效的.
    
ViewStub:就是一个宽高都为0的一个View，它默认是不可见的
1. 类似include，但是一个不可见的View类，用于在运行时按需懒加载资源，只有在代码中调用了viewStub.inflate()
	或者viewStub.setVisible(View.visible)方法时才内容才变得可见。
2. 这里需要注意的一点是，当ViewStub被inflate到parent时，ViewStub就被remove掉了，即当前view hierarchy中不再存在ViewStub，而是使用对应的layout视图代替。
```

#### onMeasure@ViewStub

```java
----> onMeasure@ViewStub:

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  	//宽高直接设置为0
    setMeasuredDimension(0, 0);
}

----> setVisibility@ViewStub:
public void setVisibility(int visibility) {
  	//已经解析过了
    if (mInflatedViewRef != null) {
        View view = mInflatedViewRef.get();
        if (view != null) {
            view.setVisibility(visibility);
        } else {
            throw new IllegalStateException("setVisibility called on un-referenced view");
        }
    } else {
        super.setVisibility(visibility);
        if (visibility == VISIBLE || visibility == INVISIBLE) {
          	//未解析时
            inflate();
        }
    }
}

----> inflate@ViewStub:
public View inflate() {
    final ViewParent viewParent = getParent();

    if (viewParent != null && viewParent instanceof ViewGroup) {
        if (mLayoutResource != 0) {
            final ViewGroup parent = (ViewGroup) viewParent;
          	//布局文件的根view
            final View view = inflateViewNoAdd(parent);
            replaceSelfWithView(view, parent);
						
          	//保存到弱引用中
            mInflatedViewRef = new WeakReference<>(view);
            if (mInflateListener != null) {
              	//监听回调
                mInflateListener.onInflate(this, view);
            }

            return view;
        } else {
            throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
        }
    } else {
        throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
    }
}

----> inflateViewNoAdd@ViewStub:
private View inflateViewNoAdd(ViewGroup parent) {
    final LayoutInflater factory;
    if (mInflater != null) {
        factory = mInflater;
    } else {
        factory = LayoutInflater.from(mContext);
    }
  	//根据布局文件创建view
    final View view = factory.inflate(mLayoutResource, parent, false);
		//如果viewStub设置了id，将id设置给布局文件根view
    if (mInflatedId != NO_ID) {
        view.setId(mInflatedId);
    }
    return view;
}

----> replaceSelfWithView@ViewStub:
private void replaceSelfWithView(View view, ViewGroup parent) {
    final int index = parent.indexOfChild(this);
  	//将自己从父view中移除
    parent.removeViewInLayout(this);
		
  	//将布局文件的根view添加到自己的位置
    final ViewGroup.LayoutParams layoutParams = getLayoutParams();
    if (layoutParams != null) {
        parent.addView(view, index, layoutParams);
    } else {
        parent.addView(view, index);
    }
}
```

