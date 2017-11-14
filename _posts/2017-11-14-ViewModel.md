
# ViewModel
ViewModel主要用来存储和管理UI相关的数据，它允许数据保存配置的变动（例如：屏幕旋转）
> **注意**：项目中导入`ViewModel`可以参考[adding components to your project](https://developer.android.com/topic/libraries/architecture/adding-components.html)

Android Framework管理UI controllers的生命周期，例如Activity和Fragment。Framework可能会决定销毁还是重新创建一个UI Controller，这个UI Controller会响应一个确定的用户动作或者那些你不受你控制的设备事件。
如果系统销毁或者重新创建了一个UI Controller，你临时存在这些UI controller中的数据都会丢掉，例如：你的应用中的Activity可能包含了一个用户List，当Activity因为configuration change被重新创建的时候，新的Activity就需要重新获取用户List，如果是简单的数据，我们可以用Activity的`onSaveInstanceState()`方法保存数据，在`onCreate()`的时候恢复数据，但这种方法仅适合存储一些可以序列化的少量的数据，不适合潜在的大一点的数据，例如用户list或者bitmap。

另外一个问题就是UI controller肯能经常需要做一些异步的回调，这些回调可能会消耗一些时间，UI controller需要管理这些回调来确保当UI controller被销毁的时候，如果这些耗时的回调还没有执行完，系统能够清理掉这些回调以避免潜在的内存泄漏。而这些管理工作需要大量的维护成本，在这种情况下，对象因为configuration change被重新创建后，可能会重新执行一些已经执行过的方法，这会造成资源的浪费。

UI controller主要是用来展示数据，响应用户动作或者处理操作系统通信（例如权限请求）。让UI controller也去响应数据加载（从数据库或者网络）会让类更加膨胀，如果分配过多的职责给UI controller而不是把一些工作代理给其他类去做，会导致这个类试图自己去处理App的所有工作，也会导致测试很难开展。

从UI controller的逻辑中分离出View data的所有权会更加简单和高效。

## 实现一个ViewMode
Architecture Components 为UI controller提供了ViewModel的帮助类，这些帮助类负责为UI准备数据。在configuration change的时候，这些`ViewModel`对象会被自动保持住，这样当Activity或者fragment重新创建实例的时候，这些数据可以立刻可用，例如：如果你需要在你得App中展示用户List，分配获取和保存用户list的责任给一个ViewModel，不要给Activity或者Fragment，如下面几行代码：

```
public class MyViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<Users>>();
            loadUsers();
        }
        return users;
    }

    private void loadUsers() {
        // Do an asyncronous operation to fetch users.
    }
}
```
之后你就可以在Activity中访问这个List：

```
public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.

        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
```
如果这个Activity被重新创建了，它会重新获取到之前创建的`MyViewModel`，当宿主Activity finish掉后，框架会调用`ViewModel`对象的`onCleared`方法来清理资源。
> **小心**：ViewModel决不能引用一个View，Lifecycle，或者其他任何可能引用了activity context的类。

ViewModel 对象被设计比指定的View的实例或者LifecycleOwners活的更长，这种设计也意味着你写覆盖ViewModel的测试用例会更简单，因为ViewModel不知道View和Lifecycle对象。ViewModel对象可以包含LifecycleObservers,例如LiveData对象，但是决不能观察lifecycle-aware observables,例如LiveData对象。如果ViewModel需要Application context，例如查找系统服务，可以通过继承AndroidViewModel类，并且在构造函数中获取Application。

## ViewModel的生命周期
当获取ViewModel的时候，ViewModel对象被限制在Lifecycle中，传递到ViewModelProvider，ViewModel会一直在内存中直到Lifecycle永久离开（对于Activity是finish后，对于Fragment是在detached后）。

图1展示了一个Activity在经历了旋转然后finish的许多生命周期状态，这幅图也展示了关联了Activity生命周期的ViewModel的lifetime，这幅图做那事了Activity的状态，作用于Fragment的生命周期的基本状态也是这样。
![viewmodel-lifecycle](http://blog-1251624639.file.myqcloud.com/viewmodel-lifecycle.png)
你通常会在系统首次调用Activity的onCreate()方法的时候请求一个ViewModel，在Activity的生命周期中，系统也许会调用多次onCreate()方法，就像设备屏幕旋转的时候，ViewModel从首次调用的时候存在，一直到Activity finish并destroy。

## Fragment之间共享数据
一个Activity中的各个Fragment之间通信时间很正常的事情，试想一个常规的master-detail fragments场景，在一个Fragment中，用户选择了一个菜单项，另一个Fragment显示该菜单项的详细内容，这个案例并不简单，应为两个Fragment都要定义一些接口，并且宿主Activity需要实现这两个接口，额外地，这两个Fragment必须处理这些其他Fragment没有创建或者不可见的情况。

这些痛点可以通过使用ViewModel来处理，这些Fragment可以通过Activity范围内的共享一个ViewModel来处理通信，如下面几行代码：

```
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<Item> selected = new MutableLiveData<Item>();

    public void select(Item item) {
        selected.setValue(item);
    }

    public LiveData<Item> getSelected() {
        return selected;
    }
}

public class MasterFragment extends Fragment {
    private SharedViewModel model;
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        itemSelector.setOnClickListener(item -> {
            model.select(item);
        });
    }
}

public class DetailFragment extends Fragment {
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        model.getSelected().observe(this, { item ->
           // Update the UI.
        });
    }
}

```
需要注意的是，两个Fragment在获取ViewModelProvider的时候通过getActivity()，所以两个Fragment获取到了相同的SharedViewModel实例，这个实例是被Activity约束的。

这种方法有如下的好处:

- Activity不需要知道任何这两个Fragment之间的通信
- Fragment 之间除了SharedViewModel不需要直到彼此，如果其中一个Fragment消失了，其他的Fragment仍然可以正常工作
- 每个Fragment有自己的生命周期，并且不受其他Fragment的影响，如果一个Fragment替换了另外一个Fragment，UI 仍然可以正常工作。

## 使用ViewModel替换Loader
像CursorLoader这样的Loader类常常被用来保持同步UI和数据库之间的数据，你可以使用ViewModel附带一些其他的类来替换Loader。使用ViewModel来将你的UI controller从数据加载操作中分离出来，这样就意味着他们之间的引用关系会更少。

在一个常规的使用Loader过程中，App可能使用CursorLoader来观察数据库的内容，当数据库内容发生改变的时候，Loader自动触发一个数据的reload动作去更新UI：
![viewmodel-loader](http://blog-1251624639.file.myqcloud.com/viewmodel-loader.png)

图 2. 使用Loader加载数据

ViewModel和Room，LiveData一起工作来替换Loader，ViewModel确保设备configuration change后数据一直存在，当数据库发生改变的时候，Room通知LiveData，LiveData反过来使用修改后的数据更新UI。

![viewmodel-replace-loader](http://blog-1251624639.file.myqcloud.com/viewmodel-replace-loader.png)
图 3. 通过ViewModel加载数据

这篇文章描述了如何使用ViewModel和LiveData来替换AsyncTaskLoader。

当你的数据变得越来越复杂，你可能会选择使用一个仅仅用来加载数据的单独的类，ViewModel的目标就是概括UI Controller中的数据，发生configuration changes时让数据一直存在。更多关于如何加载，持久化和跨configuration change数据管理，可以参考[Saving UI State](https://developer.android.com/topic/libraries/architecture/saving-state.html)


The [Guide to Android App Architecture](https://developer.android.com/topic/libraries/architecture/guide.html#fetching_data) 建议构建一个repository类来处理这些事情。

谷歌官方文档参阅[这里](https://developer.android.com/topic/libraries/architecture/viewmodel.html)

