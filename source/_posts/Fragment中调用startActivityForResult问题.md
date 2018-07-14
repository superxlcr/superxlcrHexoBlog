---
title: Fragment中调用startActivityForResult问题
tags: [android]
categories: [android]
date: 2017-10-09 20:49:47
description: Fragment中调用startActivityForResult问题
---
在使用support v4中的Fragment时，如果我们需要调用startActivityForResult方法来与跳转的Activity进行通信时，如果希望Fragment的onActivityResult方法能够被响应，我们就必须调用Fragment的startActivityForResult方法，而不是调用：
```java
getActivity().startActivityForResult()
```


后者调用的是Fragment的宿主Activity，即FragmentActivity的startActivityForResult方法。


两者的区别如下：
Fragment中的startActivityForResult方法如下：

```java
    public void startActivityForResult(Intent intent, int requestCode) {
        if (mActivity == null) {
            throw new IllegalStateException("Fragment " + this + " not attached to Activity");
        }
        mActivity.startActivityFromFragment(this, intent, requestCode);
    }
```


调用了FragmentActivity中的startActivityFromFragment方法：

```java
    public void startActivityFromFragment(Fragment fragment, Intent intent, 
            int requestCode) {
        if (requestCode == -1) {
            super.startActivityForResult(intent, -1);
            return;
        }
        if ((requestCode&0xffff0000) != 0) {
            throw new IllegalArgumentException("Can only use lower 16 bits for requestCode");
        }
        super.startActivityForResult(intent, ((fragment.mIndex+1)<<16) + (requestCode&0xffff));
    }
```


可以看到，该方法把Fragment的index值存在了requestCode的高16位，然后调用了startActivityForResult方法。
接下来我们来看看消息的返回处理，FragmentActivity中的onActivityResult方法：

```java
    /**
     * Dispatch incoming result to the correct fragment.
     */
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        mFragments.noteStateNotSaved();
        int index = requestCode>>16;
        if (index != 0) {
            index--;
            if (mFragments.mActive == null || index < 0 || index >= mFragments.mActive.size()) {
                Log.w(TAG, "Activity result fragment index out of range: 0x"
                        + Integer.toHexString(requestCode));
                return;
            }
            Fragment frag = mFragments.mActive.get(index);
            if (frag == null) {
                Log.w(TAG, "Activity result no fragment exists for index: 0x"
                        + Integer.toHexString(requestCode));
            } else {
                frag.onActivityResult(requestCode&0xffff, resultCode, data);
            }
            return;
        }
        
        super.onActivityResult(requestCode, resultCode, data);
    }
```


这里把requestCode的高16位（即Fragment的index值）取了出来，并通过index找到对应的Fragment，然后调用Fragment的onActivityResult来分发消息。
因此，如果直接调用Fragment的宿主FragmentActivity的startActivityForResult方法，requestCode中就不会存入Fragment的index值，在onActivityResult处理消息时也会找不到相应的Fragment进行进一步的消息分发。


