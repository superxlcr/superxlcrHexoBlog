---
title: Android Fragment总结
tags: [android]
categories: [android]
date: 2017-06-02 16:27:26
description: Fragment中getActivity()空指针、FragmentTransaction的四种提交方法
---
最近整理了一些关于Android中Fragment相关的知识，在此进行一下总结。

# Fragment中getActivity()空指针

某些时候，如果我们试图在Fragment中执行异步操作，那我们很大可能会遇到getActivity()返回null的问题
出现了这个奇怪的问题，我们先来了解下原因，为啥getActivity会返回null呢？来一起看下源码
Fragment#getActivity()：
```java
    final public FragmentActivity getActivity() {
        return mHost == null ? null : (FragmentActivity) mHost.getActivity();
    }
```

从上我们可以推断出，是mHost成员变量被置为了null，getActivity方法才返回null的
全局查找下对mHost的修改，修改为null的地方有两处：
1. Fragment#initState() （一看就是初始化状态的方法，忽略）
2. Fragment#moveToState() （根据名称，判断为Fragment状态转移方法，看一下它的源码）

FragmentManager#moveToState()：
```java
	...
	if (newState < Fragment.CREATED) {
		if (mDestroyed) {
			if (f.mAnimatingAway != null) {
				// The fragment's containing activity is
				// being destroyed, but this fragment is
				// currently animating away.  Stop the
				// animation right now -- it is not needed,
				// and we can't wait any more on destroying
				// the fragment.
				View v = f.mAnimatingAway;
				f.mAnimatingAway = null;
				v.clearAnimation();
			}
		}
		if (f.mAnimatingAway != null) {
			// We are waiting for the fragment's view to finish
			// animating away.  Just make a note of the state
			// the fragment now should move to once the animation
			// is done.
			f.mStateAfterAnimating = newState;
			newState = Fragment.CREATED;
		} else {
			if (DEBUG) Log.v(TAG, "movefrom CREATED: " + f);
			if (!f.mRetaining) {
				f.performDestroy();
			} else {
				f.mState = Fragment.INITIALIZING;
			}

			f.performDetach();
			if (!keepActive) {
				if (!f.mRetaining) {
					makeInactive(f);
				} else {
					f.mHost = null;
					f.mParentFragment = null;
					f.mFragmentManager = null;
				}
			}
		}
	}
	...
```

省略了部分无关代码，我们看到第36行mHost变量被置空了
那么置空的时机是什么时候呢？我们往回一点，看到第31行的performDetach()：
Fragment#performDetach()：
```java
void performDetach() {
	mCalled = false;
	onDetach();
	if (!mCalled) {
		throw new SuperNotCalledException("Fragment " + this
				+ " did not call through to super.onDetach()");
	}

	// Destroy the child FragmentManager if we still have it here.
	// We won't unless we're retaining our instance and if we do,
	// our child FragmentManager instance state will have already been saved.
	if (mChildFragmentManager != null) {
		if (!mRetaining) {
			throw new IllegalStateException("Child FragmentManager of " + this + " was not "
					+ " destroyed and this fragment is not retaining instance");
		}
		mChildFragmentManager.dispatchDestroy();
		mChildFragmentManager = null;
	}
}
```

第3行可以看到熟悉的onDetach生命周期，因此我们可以得知，当Fragment执行了onDetach之后，就意味着它已经与宿主Activity脱离了，而此时调用getActivity也将返回null

解决办法：

通过Fragment#isAdded()判断Fragment是否已经detach，如果是，则终止与Context相关的操作

网上有很多的办法是通过新建一个新的变量，强制保留对fragment宿主Activity的引用，博主个人认为这种方法是不可取的：
- 首先，强制保留对Activity的引用会导致Fragment与Activity脱离后，系统无法对Activity进行回收，该引用处理不好容易引起内存泄露
- 其次，官方允许getActivity结果为null，这种情况是在提醒开发者，Fragment已经与其宿主Activity脱离，应该中止一切与Context相关的操作（如UI操作、界面跳转等），强行地使用之前保留的Activity引用，是一种不符合官方设计意图的行为


# FragmentTransaction的四种提交方法

使用Fragment时，可以通过用户交互来执行一些动作，比如增加、移除、替换等

所有这些改变构成一个集合，这个集合被叫做一个transaction

我们可以调用FragmentTransaction中的方法来处理这个transaction，并且可以将transaction存进由activity管理的back stack中，这样用户就可以通过点击回退键，进行fragment 变化的回退操作

FragmentTransaction的提交方法有四个，分别为：

- commit：该方法不会使transaction立即生效，而是形成一个计划任务等待主线程有空时再执行。该方法必须在Activity保存自身状态前调用（即onSaveInstanceState之前，因为savedInstance会保存Fragment 的状态），否则会抛出异常。如果调用了addToBackStack方法返回backStack实例的id，否则返回一个负数
- commitAllowingStateLoss：与commit 方法类似，不同之处在于允许在Activity保存自身状态后调用，那么当Activity恢复自身状态时，该Transaction可能会由于没有被记录进savedInstance而被丢失
- commitNow：API 24添加的新方法。commit 方法不能保证提交的transaction马上生效，如果需要使这些transaction生效，通常会调用FragmentManager 的executePendingTransactions方法使之前提交的transaction依次生效。commitNow方法保证当前提交的transaction马上生效，且与之前commit 提交的transaction没有关系，但由于因此会打乱transaction的提交顺序，使用该方法提交的transaction不允许调用addToBackStack方法加入backStack，否则会抛出异常，也因此该方法没有返回值。其余与commit 方法基本类似。
- commitAllowingStateLoss：API 24添加的新方法。与commitNow类似，不同之处在于允许在Activity保存自身状态后调用，那么当Activity恢复自身状态时，该Transaction可能会由于没有被记录进savedInstance而被丢失


