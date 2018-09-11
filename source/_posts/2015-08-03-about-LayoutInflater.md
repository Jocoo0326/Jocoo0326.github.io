---
layout: post
title: About LayoutInflater
date: 2015-08-03
categories:
- android
tags:
- LayoutInflater
---
LayoutInflater是android中加载xml文件变成View的工具，几乎有view的地方都会有LayoutInflater的身影，所以需要熟悉inflate方法的各参数的意义
<!--more-->

android中LayoutInflater原理总结:

* 1、LayoutInflater用于将xml形式的view加载为真正的view；

* 2、获取LayoutInflater：
``` java
LayoutInflater layoutInflater = LayoutInflater.from(context);

LayoutInflater layoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

第一种是对第二种的封装；

* 3、常用的LayoutInflater.inflate两个重载：
``` java
public View inflate(int resource, ViewGroup root);
public View inflate(int resource, ViewGroup root, boolean attachToRoot);
```

  最终会调用以下方法：
``` java
public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context)mConstructorArgs[0];
        mConstructorArgs[0] = mContext;
        View result = root;

        try {
            // Look for the root node.
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }

            final String name = parser.getName();
            if (DEBUG) {
                System.out.println("**************************");
                System.out.println("Creating root view: "
                        + name);
                System.out.println("**************************");
            }

            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, attrs, false, false);
            } else {
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, attrs, false);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }

                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }
                // Inflate all children under temp
                rInflate(parser, temp, attrs, true, true);
                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        } catch (XmlPullParserException e) {
            InflateException ex = new InflateException(e.getMessage());
            ex.initCause(e);
            throw ex;
        } catch (IOException e) {
            InflateException ex = new InflateException(
                    parser.getPositionDescription()
                    + ": " + e.getMessage());
            ex.initCause(e);
            throw ex;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);

        return result;
    }
}
```

## 使用总结：

+ ① 若root == null，则以layout_开头的属性都将失效

+ ② 若root != null且attachToRoot == true，则会将加载的布局添加到root中并返回root

+ ③ 若root != null且attachToRoot == false，则会将布局的layout_*属性进行设置，此时返回加载的view

+ ④ 两个参数的方法中，attachToRoot = (root != null)
