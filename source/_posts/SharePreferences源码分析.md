---
title: SharePreferences源码分析
tags: [android,源码]
categories: [android]
date: 2017-10-15 17:13:49
description: SharePreferences源码分析
---
最近在项目中使用了SharePreferences，因此看了一下SharePreferences相关的源码，在此进行一下记录
SharePreferences在Android提供的几种数据存储方式中属于轻量级的键值存储方式，以XML文件方式保存数据，通常用来存储一些用户行为开关状态等，一般的存储一些常见的数据类型。

SharePreferences保存的xml文件存放于 /data/data/应用程序包/shared_prefs 的目录下
SharePreferences是接口，位于android.content包中
SharePreferencesImpl是对于SharePreferences的实现，位于android.app包下
以下是一段SharePreferences的写入以及读取的例子：


```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        SharedPreferences sp = getSharedPreferences("data", MODE_PRIVATE);
        SharedPreferences.Editor editor = sp.edit();
        editor.putString("name", "username");
        editor.commit();

        String s = sp.getString("name", "");
    }
```


第6行调用了ContextWrapper中的getSharedPreferences方法：



```java
    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        return mBase.getSharedPreferences(name, mode);
    }
```


这里调用了mBase的getSharedPreferences，mBase是ContextWrapper持有的Context变量，很明显实际类型即是ContextImpl（Context的实现类）：



```java
    @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        checkMode(mode);
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
```


上面代码首先检查了是否缓存过对应的SharedPreferences，如果没有缓存则创建SharedPreferencesImpl对象（第9行）：



```java
    SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        startLoadFromDisk();
    }
```

简单说一下这里的几个变量：

- mFile：当前的xml文件，即/data/data/应用程序包/shared_prefs/xxx.xml
- mBackupFile：当前xml备份文件，名为xxx.xml.bak
- mMode：int类型，表示打开的模式
- mLoaded：boolean类型，表示是否已经加载xml文件
- mMap：xml的数据再内存中的储存形式

在SharedPreferencesImpl构造函数中，调用了startLoadFromDisk方法加载xml中的数据：



```java
    private void startLoadFromDisk() {
        synchronized (this) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                synchronized (SharedPreferencesImpl.this) {
                    loadFromDiskLocked();
                }
            }
        }.start();
    }
	
    private void loadFromDiskLocked() {
        if (mLoaded) {
            return;
        }
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }

        // Debugging
        if (mFile.exists() && !mFile.canRead()) {
            Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
        }

        Map map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    map = XmlUtils.readMapXml(str);
                } catch (XmlPullParserException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (FileNotFoundException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } catch (IOException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
        }
        mLoaded = true;
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtime;
            mStatSize = stat.st_size;
        } else {
            mMap = new HashMap<String, Object>();
        }
        notifyAll();
    }
```


在startLoadFromDisk方法中，SharedPreferencesImpl开启了新的子线程，并持有对象锁来调用loadFromDiskLocked方法，通过XmlUtils工具类把xml读取为内存中的Map形式，最后通过notifyAll通知所有wait的地方


看完SharedPreferencesImpl的初始化，我们来看一下它的get方法，以例子中的getString为例：




```java
    public String getString(String key, String defValue) {
        synchronized (this) {
            awaitLoadedLocked();
            String v = (String)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
	
    private void awaitLoadedLocked() {
        if (!mLoaded) {
            // Raise an explicit StrictMode onReadFromDisk for this
            // thread, since the real read will be in a different
            // thread and otherwise ignored by StrictMode.
            BlockGuard.getThreadPolicy().onReadFromDisk();
        }
        while (!mLoaded) {
            try {
                wait();
            } catch (InterruptedException unused) {
            }
        }
    }
```


该方法在持有对象锁的情况下，通过awaitLoadedLocked方法检查是否已经从xml加载到内存map中，然后直接读取map中的数据即可

awaitLoadedLocked方法处理了一些与StrictMode相关的问题，在此不做讨论，在它发现SharedPreferencesImpl还没有加载完毕时，会调用wait方法把当前线程挂起，直到加载完毕通过notifyAll唤醒
看完了SharedPreferencesImpl的get方法，我们再来看一下如何向其中插入新的数据：


```java
    public Editor edit() {
        // TODO: remove the need to call awaitLoadedLocked() when
        // requesting an editor.  will require some work on the
        // Editor, but then we should be able to do:
        //
        //      context.getSharedPreferences(..).edit().putString(..).apply()
        //
        // ... all without blocking.
        synchronized (this) {
            awaitLoadedLocked();
        }

        return new EditorImpl();
    }
```


当我们调用edit方法时，会返回一个新的Edit接口的实现类EditorImpl，EditorImpl使用的是默认构造函数，它包含了两个成员变量：


- mModified：Map类型，用于存储修改的键值对
- mClear：boolean类型，用于标识是否清空SharedPreferences


获取EditorImpl实例后，我们会调用其put或者clear方法修改SharedPreferences：


```java
        public Editor putString(String key, String value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
		
        public Editor clear() {
            synchronized (this) {
                mClear = true;
                return this;
            }
        }
```


put方法会把我们的修改存入mModified变量，而clear方法会把mClear变量置为true
当我们修改SharedPreferences完成后，会调用Editor的apply或者commit方法（apply为异步提交，commit为同步提交）：

```java
        public void apply() {
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }
		
        public boolean commit() {
            MemoryCommitResult mcr = commitToMemory();
            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException e) {
                return false;
            }
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }
```

两个方法都首先调用了commitToMemory方法，把修改同步到内存中，然后都把写入文件中的任务加入到了队列当中等待执行
写入文件的任务使用的同步工具是CounDownLatch，commit在当前线程调用了await方法等待写入完成并返回Boolean表示写入状态，而apply在把写入任务加入队列后则返回了
在看commitToMemory方法之前，我们先来看看他的返回结果：MemoryCommitResult类

```java
    // Return value from EditorImpl#commitToMemory()
    private static class MemoryCommitResult {
        public boolean changesMade;  // any keys different?
        public List<String> keysModified;  // may be null
        public Set<OnSharedPreferenceChangeListener> listeners;  // may be null
        public Map<?, ?> mapToWriteToDisk;
        public final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);
        public volatile boolean writeToDiskResult = false;

        public void setDiskWriteResult(boolean result) {
            writeToDiskResult = result;
            writtenToDiskLatch.countDown();
        }
    }
```


MemoryCommitResult中：writeToDiskResult用于表示写入文件的结果，mapToWriteToDisk表示写入文件的map，即Editor更新后的map
commitToMemory方法如下：

```java
        // Returns true if any changes were made
        private MemoryCommitResult commitToMemory() {
            MemoryCommitResult mcr = new MemoryCommitResult();
            synchronized (SharedPreferencesImpl.this) {
                // We optimistically don't make a deep copy until
                // a memory commit comes in when we're already
                // writing to disk.
                if (mDiskWritesInFlight > 0) {
                    // We can't modify our mMap as a currently
                    // in-flight write owns it.  Clone it before
                    // modifying it.
                    // noinspection unchecked
                    mMap = new HashMap<String, Object>(mMap);
                }
                mcr.mapToWriteToDisk = mMap;
                mDiskWritesInFlight++;

                boolean hasListeners = mListeners.size() > 0;
                if (hasListeners) {
                    mcr.keysModified = new ArrayList<String>();
                    mcr.listeners =
                            new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
                }

                synchronized (this) {
                    if (mClear) {
                        if (!mMap.isEmpty()) {
                            mcr.changesMade = true;
                            mMap.clear();
                        }
                        mClear = false;
                    }

                    for (Map.Entry<String, Object> e : mModified.entrySet()) {
                        String k = e.getKey();
                        Object v = e.getValue();
                        // "this" is the magic value for a removal mutation. In addition,
                        // setting a value to "null" for a given key is specified to be
                        // equivalent to calling remove on that key.
                        if (v == this || v == null) {
                            if (!mMap.containsKey(k)) {
                                continue;
                            }
                            mMap.remove(k);
                        } else {
                            if (mMap.containsKey(k)) {
                                Object existingValue = mMap.get(k);
                                if (existingValue != null && existingValue.equals(v)) {
                                    continue;
                                }
                            }
                            mMap.put(k, v);
                        }

                        mcr.changesMade = true;
                        if (hasListeners) {
                            mcr.keysModified.add(k);
                        }
                    }

                    mModified.clear();
                }
            }
            return mcr;
        }
```


在第26行我们可以看到，Editor首先通过mClear变量决定是否清空mMap，然后通过mModified变量决定如何修改mMap
            