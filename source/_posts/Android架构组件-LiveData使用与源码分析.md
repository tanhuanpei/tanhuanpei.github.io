---
title: Android架构组件-LiveData使用与源码分析
date: 2018-12-18 15:39:37
tags:
---

## 前言
`LiveData`是一个可以被观察的数据装载类。跟普通的被观察者不一样的，LiveData是具有生命周期感知功能，这意味它可以感知并遵循`Activity`,`Fragment`或者`Service`等组件的生命周期。这样的设计保证了LiveData仅当组件处于激活状态时才更新数据。

只有当 Observer 的 Lifecycle 对象处于 `STARTED` 或者 `RESUMED` 状态的时候， LiveData 才处于活动状态，只有在活动状态数据变化事件才会通知到 Observer。当处于`DESTROYED`状态的时候，Observer会被相应的移除。

###  LiveData的优点
使用LiveData具有以下的有点：
- **UI和实时数据保持一致** 因为LiveData采用的是观察者模式，这样一来就可以在数据发生改变时获得通知，更新UI。
**避免内存泄漏**-  观察者被绑定到组件的生命周期上，当被绑定的组件销毁（destroy）时，观察者会立刻自动清理自身的数据。
**不会再产生由于Activity处于stop状态而引起的崩溃**- 例如：当Activity处于后台状态时，是不会收到LiveData的任何事件的。
**不需要再解决生命周期带来的问题**-  LiveData可以感知被绑定的组件的生命周期，只有在活跃状态才会通知数据变化。
**实时数据刷新**-  当组件处于活跃状态或者从不活跃状态到活跃状态时总是能收到最新的数据。
**解决Configuration Change问题**-  在屏幕发生旋转或者被回收再次启动，立刻就能收到最新的数据。
**资源共享**- 通过使用单例模式和继承LiveData，可以实现一次发送，多个观察者更新数据。


## 添加LiveData

### 添加google maven仓库
在项目`build.gradle`中添加：
```
allprojects {
    repositories {
        google()
        jcenter()
    }
}
```
**AndroidX**
```
dependencies {
    def lifecycle_version = "2.0.0"

    // ViewModel and LiveData
    implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"
    // alternatively - just LiveData
    implementation "androidx.lifecycle:lifecycle-livedata:$lifecycle_version"
	
    // optional - ReactiveStreams support for LiveData
    implementation "androidx.lifecycle:lifecycle-reactivestreams:$lifecycle_version" // use -ktx for Kotlin

    // optional - Test helpers for LiveData
    testImplementation "androidx.arch.core:core-testing:$lifecycle_version"
}
```

**AndroidX之前**
```
dependencies {
    def lifecycle_version = "1.1.1"

    // ViewModel and LiveData
    implementation "android.arch.lifecycle:extensions:$lifecycle_version"
    // alternatively - just LiveData
    implementation "android.arch.lifecycle:livedata:$lifecycle_version"
    // optional - ReactiveStreams support for LiveData
    implementation "android.arch.lifecycle:reactivestreams:$lifecycle_version"

    // optional - Test helpers for LiveData
    testImplementation "android.arch.core:core-testing:$lifecycle_version"
}
```
### 创建LiveData对象

**MutableLiveData**
```
public class NameViewModel extends ViewModel {

// Create a LiveData with a String
private MutableLiveData<String> mCurrentName;

    public MutableLiveData<String> getCurrentName() {
        if (mCurrentName == null) {
            mCurrentName = new MutableLiveData<String>();
        }
        return mCurrentName;
    }

// Rest of the ViewModel...
}
```
```
public class NameActivity extends AppCompatActivity {

    private NameViewModel mModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Other code to setup the activity...

        // Get the ViewModel.
        mModel = ViewModelProviders.of(this).get(NameViewModel.class);


        // Create the observer which updates the UI.
        final Observer<String> nameObserver = new Observer<String>() {
            @Override
            public void onChanged(@Nullable final String newName) {
                // Update the UI, in this case, a TextView.
                mNameTextView.setText(newName);
            }
        };

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        mModel.getCurrentName().observe(this, nameObserver);
    }
}
```
**MediatorLiveData**
```
 LiveData liveData1 = ...;
 LiveData liveData2 = ...;

 MediatorLiveData liveDataMerger = new MediatorLiveData<>();
 liveDataMerger.addSource(liveData1, value -> liveDataMerger.setValue(value));
 liveDataMerger.addSource(liveData2, value -> liveDataMerger.setValue(value));
```

**使用单例模式**

```
public class SingletonLiveData extends MutableLiveData<String> {

    private static volatile SingletonLiveData instance;

    public SingletonLiveData() {
    }

    public static SingletonLiveData getInstance() {
        if (instance != null) {
            return instance;
        }
        synchronized (SingletonLiveData.class) {
            if (instance == null) {
                instance = new SingletonLiveData();
            }
        }
        return instance;
    }

    @Override
    protected void onActive() {
        super.onActive();
    }

    @Override
    protected void onInactive() {
        super.onInactive();
    }
}
```
```java
SingletonLiveData.getInstance().observe(this, new Observer<String>() {
            @Override
            public void onChanged(@Nullable String s) {

            }
        });
```
```
SingletonLiveData.getInstance().setValue("single");
```
简单说明LiveData中的几个重要方法：
**onActive() **- ：当 LiveData 具有活动状态的 Observer 的时候会调用这个函数。在这个函数中，我们开始监听位置变化。
**onInactive()**- ：当 LiveData 没有活动状态的 Observer 的时候会调用这个函数。所以在这个函数中取消更新位置来避免无所谓的耗电。
**setValue()**- ： 当 LiveData 的数据变化的时候，调用这个函数。然后 LiveData 会通知活动状态的 Observer 数据变化了。
**postValue()**- :当需要在非UI线程更新数据时调用此函数。

需要注意的是，当使用 LiveData 时，由于CREATED 和 RESUMED 状态都被认为是激活状态，所以我们需要在 onCreate 中设置 LiveData并开始监听，如果监听放到 onResume 函数中的话，当 Activity 暂停再次恢复的时候，会调用 onStart 和 onResume 导致 LiveData 的 onActive() 函数分别被调用两次。

**数据转换**
LiveData 还支持简单的数据变换。目前在 `Transformations` 类中有 map 和 switchMap 两个变换函数。

- map 是把一个数据类型变换为另外一个数据类型。
- switchMap 是把一个数据变化为另外一个 LiveData

```java
LiveData<User> userLiveData = ...;
LiveData<String> userName = Transformations.map(userLiveData, user -> {
    user.name + " " + user.lastName
});
```
```java
class MyViewModel extends ViewModel {
    private final PostalCodeRepository repository;
    private final MutableLiveData<String> addressInput = new MutableLiveData();
    public final LiveData<String> postalCode =
            Transformations.switchMap(addressInput, (address) -> {
                return repository.getPostCode(address);
             });

  public MyViewModel(PostalCodeRepository repository) {
      this.repository = repository
  }

  private void setInput(String address) {
      addressInput.setValue(address);
  }
}
```

##  原理分析

下面，我们深入源码，简单分析LiveData如何做到生命周期感知。先从注册观察者入手，源码中LiveData的observe方法如下
```java
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        LifecycleBoundObserver existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && existing.owner != wrapper.owner) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }
```
从这段代码看出，当观察者所在的`LifecycleOwner`状态处于DESTROYED时，注册失败。当重复注册观察者时，`会抛出IllegalArgumentException`异常。通过这些验证后，向`Lifecycle`注册观察者observer，LifecycleBoundObserver只是observer的一个包装类。`LifecycleOwner`是一个接口，只有一个方法getLifecycle()，`在Fragment`、`FragmentActivity`、`SupportActivity`、`LifecycleService`中均实现了这个方法。

```java
class LifecycleBoundObserver implements GenericLifecycleObserver {
        public final LifecycleOwner owner;
        public final Observer<T> observer;
        public boolean active;
        public int lastVersion = START_VERSION;

        LifecycleBoundObserver(LifecycleOwner owner, Observer<T> observer) {
            this.owner = owner;
            this.observer = observer;
        }

        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (owner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(observer);
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            activeStateChanged(isActiveState(owner.getLifecycle().getCurrentState()));
        }

        void activeStateChanged(boolean newActive) {
            if (newActive == active) {
                return;
            }
            active = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += active ? 1 : -1;
            if (wasInactive && active) {
                onActive();
            }
            if (LiveData.this.mActiveCount == 0 && !active) {
                onInactive();
            }
            if (active) {
                dispatchingValue(this);
            }
        }
    }
```
```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}

```
```java
public class Fragment implements ComponentCallbacks, OnCreateContextMenuListener, LifecycleOwner {

    LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
	
	void performStart() {
        if (mChildFragmentManager != null) {
            mChildFragmentManager.noteStateNotSaved();
            mChildFragmentManager.execPendingActions();
        }
        mState = STARTED;
        mCalled = false;
        onStart();
        if (!mCalled) {
            throw new SuperNotCalledException("Fragment " + this
                    + " did not call through to super.onStart()");
        }
        if (mChildFragmentManager != null) {
            mChildFragmentManager.dispatchStart();
        }
        if (mLoaderManager != null) {
            mLoaderManager.doReportStart();
        }
        mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
    }
}
```
**LiveData中内部类LifecycleBoundObserver**
```java
class LifecycleBoundObserver implements GenericLifecycleObserver {
        public final LifecycleOwner owner;
        public final Observer<T> observer;
        public boolean active;
        public int lastVersion = START_VERSION;

        LifecycleBoundObserver(LifecycleOwner owner, Observer<T> observer) {
            this.owner = owner;
            this.observer = observer;
        }

        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (owner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(observer);
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            activeStateChanged(isActiveState(owner.getLifecycle().getCurrentState()));
        }

        void activeStateChanged(boolean newActive) {
            if (newActive == active) {
                return;
            }
            active = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += active ? 1 : -1;
            if (wasInactive && active) {
                onActive();
            }
            if (LiveData.this.mActiveCount == 0 && !active) {
                onInactive();
            }
            if (active) {
                dispatchingValue(this);
            }
        }
    }
```
**LifecycleRegistry.java**
```java
public class LifecycleRegistry extends Lifecycle {

    private static final String LOG_TAG = "LifecycleRegistry";

    /**
     * Custom list that keeps observers and can handle removals / additions during traversal.
     *
     * Invariant: at any moment of time for observer1 & observer2:
     * if addition_order(observer1) < addition_order(observer2), then
     * state(observer1) >= state(observer2),
     */
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();
    /**
     * Current state
     */
    private State mState;
    /**
     * The provider that owns this Lifecycle.
     * Only WeakReference on LifecycleOwner is kept, so if somebody leaks Lifecycle, they won't leak
     * the whole Fragment / Activity. However, to leak Lifecycle object isn't great idea neither,
     * because it keeps strong references on all other listeners, so you'll leak all of them as
     * well.
     */
    private final WeakReference<LifecycleOwner> mLifecycleOwner;

    private int mAddingObserverCounter = 0;

    private boolean mHandlingEvent = false;
    private boolean mNewEventOccurred = false;

    // we have to keep it for cases:
    // void onStart() {
    //     mRegistry.removeObserver(this);
    //     mRegistry.add(newObserver);
    // }
    // newObserver should be brought only to CREATED state during the execution of
    // this onStart method. our invariant with mObserverMap doesn't help, because parent observer
    // is no longer in the map.
    private ArrayList<State> mParentStates = new ArrayList<>();

    /**
     * Creates a new LifecycleRegistry for the given provider.
     * <p>
     * You should usually create this inside your LifecycleOwner class's constructor and hold
     * onto the same instance.
     *
     * @param provider The owner LifecycleOwner
     */
    public LifecycleRegistry(@NonNull LifecycleOwner provider) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
    }

    /**
     * Moves the Lifecycle to the given state and dispatches necessary events to the observers.
     *
     * @param state new state
     */
    @SuppressWarnings("WeakerAccess")
    @MainThread
    public void markState(@NonNull State state) {
        moveToState(state);
    }

    /**
     * Sets the current state and notifies the observers.
     * <p>
     * Note that if the {@code currentState} is the same state as the last call to this method,
     * calling this method has no effect.
     *
     * @param event The event that was received
     */
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }

    private boolean isSynced() {
        if (mObserverMap.size() == 0) {
            return true;
        }
        State eldestObserverState = mObserverMap.eldest().getValue().mState;
        State newestObserverState = mObserverMap.newest().getValue().mState;
        return eldestObserverState == newestObserverState && mState == newestObserverState;
    }

    private State calculateTargetState(LifecycleObserver observer) {
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

        State siblingState = previous != null ? previous.getValue().mState : null;
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
                : null;
        return min(min(mState, siblingState), parentState);
    }

    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }

    private void popParentState() {
        mParentStates.remove(mParentStates.size() - 1);
    }

    private void pushParentState(State state) {
        mParentStates.add(state);
    }

    @Override
    public void removeObserver(@NonNull LifecycleObserver observer) {
        // we consciously decided not to send destruction events here in opposition to addObserver.
        // Our reasons for that:
        // 1. These events haven't yet happened at all. In contrast to events in addObservers, that
        // actually occurred but earlier.
        // 2. There are cases when removeObserver happens as a consequence of some kind of fatal
        // event. If removeObserver method sends destruction events, then a clean up routine becomes
        // more cumbersome. More specific example of that is: your LifecycleObserver listens for
        // a web connection, in the usual routine in OnStop method you report to a server that a
        // session has just ended and you close the connection. Now let's assume now that you
        // lost an internet and as a result you removed this observer. If you get destruction
        // events in removeObserver, you should have a special case in your onStop method that
        // checks if your web connection died and you shouldn't try to report anything to a server.
        mObserverMap.remove(observer);
    }

    /**
     * The number of observers.
     *
     * @return The number of observers.
     */
    @SuppressWarnings("WeakerAccess")
    public int getObserverCount() {
        return mObserverMap.size();
    }

    @Override
    public State getCurrentState() {
        return mState;
    }

    static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }

    private static Event downEvent(State state) {
        switch (state) {
            case INITIALIZED:
                throw new IllegalArgumentException();
            case CREATED:
                return ON_DESTROY;
            case STARTED:
                return ON_STOP;
            case RESUMED:
                return ON_PAUSE;
            case DESTROYED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }

    private static Event upEvent(State state) {
        switch (state) {
            case INITIALIZED:
            case DESTROYED:
                return ON_CREATE;
            case CREATED:
                return ON_START;
            case STARTED:
                return ON_RESUME;
            case RESUMED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }

    private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
                mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                Event event = downEvent(observer.mState);
                pushParentState(getStateAfter(event));
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }

    // happens only on the top of stack (never in reentrance),
    // so it doesn't have to take in account parents
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                    + "new events from it.");
            return;
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }

    static State min(@NonNull State state1, @Nullable State state2) {
        return state2 != null && state2.compareTo(state1) < 0 ? state2 : state1;
    }

    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
}
```
取Fragment举例，当Fragment状态切换调用perforStart()时，会执行LifecycleRegistry的handleLifecycleEvent()，然后依次调用moveToState()->sync()->forwardPass()->内部类ObserverWithState.dispatchEvent(LifecycleOwner owner, Event event)，然后调用LiveData中内部类LifecycleBoundObserver的onStateChanged()，当状态为DESTROYED时，自动解除绑定observer，实现了生命周期感知，避免内存泄露。

到这里，我们已经了解到LiveData绑定观察者和解绑观察者的流程。下面我们看下数据是怎么实现更新的。


```java
protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }

protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }


private final Runnable mPostValueRunnable = new Runnable() {
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            //noinspection unchecked
            setValue((T) newValue);
        }
    };
```
```java
private void dispatchingValue(@Nullable LifecycleBoundObserver initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<T>, LifecycleBoundObserver>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }


private void considerNotify(LifecycleBoundObserver observer) {
        if (!observer.active) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!isActiveState(observer.owner.getLifecycle().getCurrentState())) {
            observer.activeStateChanged(false);
            return;
        }
        if (observer.lastVersion >= mVersion) {
            return;
        }
        observer.lastVersion = mVersion;
        //noinspection unchecked
        observer.observer.onChanged((T) mData);
    }
```
在`LiveData`中setVaule和postValue方法最终结果都是一样的，只是postValue提供给在非UI线程中调用。
这里有个逻辑需要注意，在这篇[文章](https://tech.meituan.com/Android_LiveDataBus.html "文章")中已经讲的很清晰，当LifeCircleOwner的状态发生变化的时候，会调用LiveData.ObserverWrapper的activeStateChanged函数，如果这个时候ObserverWrapper的状态是active，就会调用LiveData的dispatchingValue，接着调用considerNotify方法。
在LiveData的considerNotify方法中，如果ObserverWrapper的mLastVersion小于LiveData的mVersion，就会去回调mObserver的onChanged方法。而每个新的订阅者，其version都是-1，LiveData一旦设置过其version是大于-1的（每次LiveData设置值都会使其version加1），这样就会导致LiveDataBus每注册一个新的订阅者，这个订阅者立刻会收到一个回调，即使这个设置的动作发生在订阅之前。

## 总结
上面已经介绍了LiveData的使用方法和通过源码分析一遍实现原理。通过LifecycleRegistry类将Fragment、Activity和Service组件的生命周期绑定到LiveData上，并通过观察者模式实现了数据的更新。