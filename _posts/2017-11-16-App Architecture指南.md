本指南适用于过去开发过App，并且希望能获得 构建健壮和产品级质量的App的最佳实践和推荐架构的开发人员。
> **注意**：本指南假定读者已经对Android Framework比较熟悉，如果你是新手，请先获取Getting Start的课程系列，这些内容覆盖了本指南的先决内容。

## App开发中面临的常见问题
与传统的桌面同类产品不同，桌面应用多数情况下在启动器的快捷方式有一个单独的入口，并且在一个大的进程中运行，Android的应用的机构更加复杂。一个典型的Android应用的构建常常有很多[app components](https://developer.android.com/guide/components/fundamentals.html#Components)，包含：activitys，fragments，services，content providers 和 broadcast receivers。

多数app组件在manifest中声明，这个文件会被Android系统使用，用来决定如何让你的App融入整体的用户体验。然而，就像前面提到的，传统的桌面应用运行在一个大的进程中，一个编写规范的Android应用会更加灵活，就像用户迂回于不同的应用之间不停地切换Flows(`笔者根据下文，理解为应用间的互相调用`)和Tasks一样。

例如：考虑下当你在你喜欢的社交App上分享一张图片的时候会发生什么：App触发一个相机intent，Android系统启动相机App来响应这个intent，此时，用户离开社交App但是体验是无缝的，反过来相机应用也可能会触发其他intent，例如启动文件选择器，文件选择器也可能启动其他App，最终用户回到了社交App并分享了这张照片，当然用户也有可能在这个过程中被电话打断，电话结束后再回来分享照片。

在Android中，这种App-hopping的行为很正常，因此你的App也必须正确处理这些Flows。

所有这些的关键就是App组件能够不按顺序被立刻启动并且能够随时被用户或者系统销毁，因为App组件是短暂的并且它们的生命周期（当被创建和销毁时）不受你的控制，**你不应该在组件中存储任何App的数据或者状态**，并且组件之间之间也不应该互相依赖。

## 常规的架构原则
如果无法使用App组件来存储App的数据和状态，那么App如何组织起来？

在你的App中，你最该关注的是[separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns)（关注点分离），经常犯的错误就是把所有的代码写在一个Activity或者Fragment中，任何无关UI以及系统交互的类都不该放在里面，越少的代码在Activity和Fragment中，越能减少因为生命周期而带来的问题。别忘了，这些类不属于你，它们仅仅是象征着操作系统和你的App之间约定的胶水类。Android系统可能在用户交互或者其他内存不够等情况下随时把它们销毁，最好是尽量减少对他们的依赖以便提供连续的用户体验。

第二个重要的原则是你应该**用model来驱动UI**，更可取的是可以持久化的model。持久化是理想方案，有两个原因：1.如果操作系统销毁了你的App来释放资源用户不会丢失数据，并且你的App甚至是在网络链接不稳定或者没有网络的情况下仍然能够工作。Model是负责为App处理数据的组件，它们在你的App中独立于View和其他组件，因此它们被从这些组件的生命周期问题中隔离出来。保持UI的代码简单逻辑清晰会让管理变得简单。把App建立在有良好定义的管理数据职责的Model类上，会让你的代码可测试，让你的App连贯。

## 推荐的App architecture
这部分，我们示范下如何通过一个用例，使用[Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html)构建一个App。
> 注意：可能有一个对于任何情况来说都是最好的编写App的方法，话虽如此，这个推荐的架构应该对于多数用例来说是一个好的出发点，如果你已经有一个不错的编写Android应用的方法，你不需要改变。

设想我们正在构建一个展示用户信息的UI，用户的信息通过REST API从我们的后台获取。

### 构建UI界面

UI包含一个Fragment`UserProfileFragment.java` 和它相关的布局文件 `user_profile_layout.xml`.

为了驱动UI，我们的数据模型需要持有两个数据元素。

- 用户ID：用的ID，传递到Fragment中是最好是使用Fragment的arguments，如果Android系统销毁了你的进程，这些信息会被保护起来，当下次重新启动时候仍然可用。
- 用户对象：一个持有用户信息POJO类
我们创建一个给予ViewModel的UserProfileViewModel类来存储这些信息。

ViewModel为指定的UI组件提供数据，例如Fragment或者Activity，并且处理与业务的数据处理部分的通信，比如调用其他容器去加载数据或者发送用的的修改，ViewModel不知道View，也不受configuration changes的影响，例如因为屏幕旋转而重新创建Activity。

我们有3个文件。

- `user_profile.xml`: UI定义.
- `UserProfileViewModel.java`: 为UI准备数据的类.
- `UserProfileFragment.java`: 展示数据和响应用户交互的UI controller。

下面使我们的最开始的实现（为了简化省掉了layout文件）:

```
public class UserProfileViewModel extends ViewModel {
    private String userId;
    private User user;

    public void init(String userId) {
        this.userId = userId;
    }
    public User getUser() {
        return user;
    }
}
```

```
public class UserProfileFragment extends Fragment {
    private static final String UID_KEY = "uid";
    private UserProfileViewModel viewModel;

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        String userId = getArguments().getString(UID_KEY);
        viewModel = ViewModelProviders.of(this).get(UserProfileViewModel.class);
        viewModel.init(userId);
    }

    @Override
    public View onCreateView(LayoutInflater inflater,
                @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.user_profile, container, false);
    }
}

```
现在，我们有了这3个代码模块，我们怎么把他们联系起来呢？毕竟，当设置了ViewModel的用户字段后，我们需要一个方法去通知UI，这就是LiveData类的用户之地。

LiveData是一个可观察的数据的holder，它让你App上的组件观察LiveData对象的改变，而无需在它们之间创建直接的死板的依赖路径，LiveData也遵守组件（activities，fragments，services）的生命周期状态，并且做正确的事情避免对象的泄漏，这样你的应用就不需要消耗更多的内存。

> 注意：如果你已经用了像[RxJava](https://github.com/ReactiveX/RxJava)或者[Agera](https://github.com/google/agera)这样的类库，你仍可以继续使用它们而不需要LiveData，但是如果你使用它们或者其他方法是，确定你正确地处理了生命周期，例如当LifecycleOwner停止的时候你的数据流暂停，当LifecycleOwner销毁的时候你的数据流被销毁，你也可以添加`android.arch.lifecycle:reactivestreams`构件来使用另外的reactive streams library（例如，RxJava2）。

现在我们用LiveData<User>替换UserProfileViewModel中的User字段，这样当数据更新后，Fragment能够被通知，关于LiveData最了不起的地方是它是生命周期感知的，并且当它们不再被需要的时候，会自动清理引用。

```
public class UserProfileViewModel extends ViewModel {
    ...
    //private User user;
    private LiveData<User> user;
    public LiveData<User> getUser() {
        return user;
    }
}
```
现在我们修改UserProfileFragment来观察数据并更新UI。

```
@Override
public void onActivityCreated(@Nullable Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    viewModel.getUser().observe(this, user -> {
      // update UI
    });
}
```
每次用户数据更新，onChanged回调都会被调用，并且UI也会被更新。

如果你对其他使用了可被观察的回调的类库熟悉，你可能意识到了我们并没有重写Fragment的onStop()方法来停止观察数据，对LiveData来说是不需要的，因为它是生命周期感知的，那也意味着它不会调用callback除非Fragment处在一个活跃的状态（收到onStart()但是没有收到onStrop()），当Fragment收到onDestory()时，LiveData也会自动移除观察者。

我们也没有做什么特别的来处理configuration changes（例如，用户旋转了屏幕）。当configuration changes时，ViewModel会被自动恢复，因此新的Fragment一旦进入生命周期就会立刻收到相同的ViewModel实例，并且callback会使用当前的数据被立刻调用，这就是为什么ViewModel不应该直接引用View的原因；它们能比View的生命周期活的更久，参阅[The lifecycle of a ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html#the_lifecycle_of_a_viewmodel)

### 数据获取

现在我们已经连接ViewModel到Fragment上了，但是ViewModel如何获取用户数据呢？这个例子中，我们假定我们的后台提供了一个REST API，我们用Retrofit库去访问我们的后台，当然你也可以用其他类库，但是我们的目的是一样的。
下面是retrofit与后台通信的Webservice：

```
public interface Webservice {
    /**
     * @GET declares an HTTP GET request
     * @Path("user") annotation on the userId parameter marks it as a
     * replacement for the {user} placeholder in the @GET path
     */
    @GET("/users/{user}")
    Call<User> getUser(@Path("user") String userId);
}

```
一个本地实现的ViewModel可以直接调用Webservice来获取数据并且指定给用户对象，虽然这样也能跑起来，但是这样很难维护，这给ViewModel类太多的职责，这样违背了我们之前提到的关注点分离的原则，另外，ViewModel的范围也和Activity或者Fragment捆在了一起，这样会导致当生命周期结束后，所有的数据都会丢失，这样用户体验会很糟糕，相反地，我们的ViewModel应该把这些工作代理给一个新的**Repository**模块处理。

**Repository** 模块负责数据处理，他们提供了一个干净的API给余下的部分，他们知道从哪获取数据，数据更新的时候应该调用哪个API，你可以叫它们不同数据源（persistent model，web service，cache等）的调节器。

下面的UserRepository类使用WebService来获取用户数据。

```
public class UserRepository {
    private Webservice webservice;
    // ...
    public LiveData<User> getUser(int userId) {
        // This is not an optimal implementation, we'll fix it below
        final MutableLiveData<User> data = new MutableLiveData<>();
        webservice.getUser(userId).enqueue(new Callback<User>() {
            @Override
            public void onResponse(Call<User> call, Response<User> response) {
                // error case is left out for brevity
                data.setValue(response.body());
            }
        });
        return data;
    }
}

```

尽管repository模块看起来不是必须的，它服务了一个重要的目的，它从app其余的部分抽象了数据源，现在我们的ViewModel不知道数据是从webservice获取的，这意味着如果有需要我们可以切换成其他的实现。

> **注释**: 为了简化我们忽略了网络错误的情况，对于一个可选的暴露错误和加载状态的实现，参阅[Addendum: exposing network status](https://developer.android.com/topic/libraries/architecture/guide.html#addendum)

### 管理组件之间的依赖:

上面的UserRepository类需要一个Webservice的实例来完成它的工作，可以简单地创建它，但是它也需要知道创建Webservice的时候需要的依赖，这很显然复杂且代码重复（例如，每个需要一个Webservice实例的类都需要知道如何通过它的依赖来构造它），另外，UserRepository也不是唯一需要Webservice的类，如果每个类都创建一个WebService，会对资源造成很大的浪费。

有两种模式可以解决这个问题：

- [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection)：依赖注入允许类在没有构造的时候定义它们的依赖，运行的时候，其他类负责提供这些依赖，我们推荐使用谷歌的[Dagger 2](https://google.github.io/dagger/)库来实现Android上的依赖注入，Dagger 2通过遍历依赖树自动构造对象，并且对依赖提供编译时保障。
- [Service Loader](https://en.wikipedia.org/wiki/Service_locator_pattern):Service Loader 可以提供一个注册，在这里类可以在没有构造的情况下获取它们的依赖，它比依赖注入（DI）更容易实现，如果你对DI不熟悉，可以使用Service Loader来替代。

这种模式可以让你的代码更具有伸缩性，因为它们提供了干净的模式来管理依赖，而不用复制代码或者更复杂。它们也都允许为测试时交换实现方式，这也是使用它们的一个好处。

这个例子里面，我们将用Dagger 2来管理依赖。

### 连接ViewModel还有repository
现在我们修改ViewModel来使用repository。

```
public class UserProfileViewModel extends ViewModel {
    private LiveData<User> user;
    private UserRepository userRepo;

    @Inject // UserRepository parameter is provided by Dagger 2
    public UserProfileViewModel(UserRepository userRepo) {
        this.userRepo = userRepo;
    }

    public void init(String userId) {
        if (this.user != null) {
            // ViewModel is created per Fragment so
            // we know the userId won't change
            return;
        }
        user = userRepo.getUser(userId);
    }

    public LiveData<User> getUser() {
        return this.user;
    }
}

```
### 缓存数据

上面的repository实现对抽象webservice的调用很有好处，但是因为它依赖仅仅一个数据源，所以并不是很实用。

上面的UserRepository实现的问题是，当它获取到数据后，并没有保存在哪，如果用户离开了UserProfileFragment然后重新回来，App会重新拉去数据，这很不友好，原因有两个：1.浪费宝贵的网络带宽；2.强制用户等待新的请求完成。为了处理这个问题，我们会再添加一个新的数据源到UserRepository，它会把用户对象缓存在内存中。

```
@Singleton  // informs Dagger that this class should be constructed once
public class UserRepository {
    private Webservice webservice;
    // simple in memory cache, details omitted for brevity
    private UserCache userCache;
    public LiveData<User> getUser(String userId) {
        LiveData<User> cached = userCache.get(userId);
        if (cached != null) {
            return cached;
        }

        final MutableLiveData<User> data = new MutableLiveData<>();
        userCache.put(userId, data);
        // this is still suboptimal but better than before.
        // a complete implementation must also handle the error cases.
        webservice.getUser(userId).enqueue(new Callback<User>() {
            @Override
            public void onResponse(Call<User> call, Response<User> response) {
                data.setValue(response.body());
            }
        });
        return data;
    }
}

```
### 数据持久化

在我们当前的实现中，如果用户旋转屏幕或者离开并重新返回App，已经存在的UI会立刻可见，因为repository会从内存里面取回数据。但是如果用户离开App并在几个小时候返回，这时候Android系统杀死了该进程，会发生什么呢？

当前的实现方案，我们需要重新从网络获取数据。这不仅用户体验差，而且是一个很大的浪费，因为它要用手机的网重新拉去同样的数据。你可以通过缓存web请求来简单地修正这个问题，但是又造成了一个新的问题，如果相同的用户数据从另外一个类型的请求（例如获取朋友列表）展示出来会发生什么呢？然后你的App很可能会展示不一致的数据，这样会让用户感到疑惑。举个例子：相同用户数据可能有不同的展示，因为朋友列表接口请求和用户接口请求可能会在不同时间执行，App应该合并它们来避免显示不一致的数据。

处理这个问题的合适做法是使用persistent model，这就是[Room](https://developer.android.com/training/data-storage/room/index.html)做的事情。

Room是一个ORM库，它提供了使用最少的样板代码来实现本地数据的持久化的方法，在编译时，它对着schema为每一个查询做验证，因此错误的SQL查询会在编译时出错而不是在运行时失败，Room抽象了一些粗的SQL表和查询的实现细节，它也允许观察数据库数据（包括集合和连接查询）的改变，这些改变通过LiveData对象暴露出来。另外，它明确定义了处理常规问题（例如在主线程里面访问存储）的线程限制。

>**注释**: 如果你已经使用了像ORM这样的持久化方案，就没什么必要用Room替换现有方案。然而，如果你在编写一个新应用，或者正在重构已有的应用，我们推荐使用Room来持久化数据，那样你可以获得Room的抽象和查询验证能力的好处。使用Room时，我们需要定义本地schema，首先，用[@Entity](https://developer.android.com/reference/android/arch/persistence/room/Entity.html)注解User类来让User作为一个数据库中的表。

```
@Entity
class User {
  @PrimaryKey
  private int id;
  private String name;
  private String lastName;
  // getters and setters for fields
}
```
然后，通过继承RoomDatabase创建一个数据库类：

```
@Database(entities = {User.class}, version = 1)
public abstract class MyDatabase extends RoomDatabase {
}

```
注意，MyDatabase是抽象类，Room自动提供了它的实现，细节可以查看[Room](https://developer.android.com/topic/libraries/architecture/room.html)的文档。

现在我们需要将数据数据插入到数据库，这样，我们需要创建一个数据访问对象（[DAO](https://en.wikipedia.org/wiki/Data_access_object)）。

```
@Dao
public interface UserDao {
    @Insert(onConflict = REPLACE)
    void save(User user);
    @Query("SELECT * FROM user WHERE id = :userId")
    LiveData<User> load(String userId);
}
```
然后，在database类中引用DAO。

```
@Database(entities = {User.class}, version = 1)
public abstract class MyDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}

```
注意，load方法返回了LiveData<User>，Room知道数据库何时被修改，并且数据被修改时它会自动通知所有活跃的观察者，因为它使用了LiveData，这会很高效，因为它仅仅会在有活跃的观察者的时候，才会更新数据。

>**注释**: Room基于表的修改来检查失效，这意味着它可能分发错误的通知。

现在我们来修改UserRepository来引入Room数据源。

```
@Singleton
public class UserRepository {
    private final Webservice webservice;
    private final UserDao userDao;
    private final Executor executor;

    @Inject
    public UserRepository(Webservice webservice, UserDao userDao, Executor executor) {
        this.webservice = webservice;
        this.userDao = userDao;
        this.executor = executor;
    }

    public LiveData<User> getUser(String userId) {
        refreshUser(userId);
        // return a LiveData directly from the database.
        return userDao.load(userId);
    }

    private void refreshUser(final String userId) {
        executor.execute(() -> {
            // running in a background thread
            // check if user was fetched recently
            boolean userExists = userDao.hasUser(FRESH_TIMEOUT);
            if (!userExists) {
                // refresh the data
                Response response = webservice.getUser(userId).execute();
                // TODO check for error etc.
                // Update the database.The LiveData will automatically refresh so
                // we don't need to do anything else here besides updating the database
                userDao.save(response.body());
            }
        });
    }
}

```
注意，虽然我们修改了UserRepository里的数据源，但是不需要修改UserProfileViewModel或者UserProfileFragment，这就是抽象提供的灵活性，这也对测试很友好，因为当你测试UserProfileViewModel的时候，你可以提供一个假的UserRepository。

现在我们的代码写完了，如果用户几天后回到相同的UI页面，也会立刻看到用户信息，因为我们持久化了这些数据。同时，如果数据过期repository会在后台更新数据。当然，这取决于你的用例，你也可能不展示那些过期的数据。

在有些用例中，例如pull-to-refresh,让UI展示当前是否有正在处理的网络操作给用户很重要，从实际的数据中分离出UI是一个很好的实践，因为这些数据可能因为很多原因（例如，如果我们拉取朋友列表数据，相同的用户信息可能被再次拉取触发LiveData<User>更新）被更新。从UI的观点上讲，正在进行的网络请求仅仅是另一个数据点，和其他任何数据（像用户对象）碎片都一样。

这种情况有两种常规的解决方案：

- 修改getUser，返回一个带有网络操作状态的LiveData。这里提供了一个例子：[exposing network status](https://developer.android.com/topic/libraries/architecture/guide.html#addendum)。

- 在repository类中提供一个公共方法，它返回User的刷新状态，如果你想在UI中仅为确切的用户动作（例如pull-to-refresh）展示网络状态，这个选择会更好点。

#### 单一数据源

不同的REST API接口返回相同的数据是很正常的事情，例如：我们的后台有另外一个借口返回朋友列表，相同的用户对象可能来自两个不同的API，可能粒度不同。如果UserRepository想要从webservice返回响应，UI有可能显示不一致的数据，因为在这些请求之间数据在服务端可能改变了。这就是为什么在UserRepository的实现中，webservice回调仅仅保存数据到数据库，然后数据库的改变会触发活跃的LiveData对象的回调。

在这个模型中，数据库作为单一数据源，其他地方通过repository访问它，不管有没有使用磁盘缓存，我们推荐repository指定一个数据源作为单一数据源。

### Testing

我们之前提到分离的其中一个好处就是容易测试，让我们看下怎么测试每个模块。

- UI和交互：这是你唯一需要使用Android UI Instrumentation的地方。测试UI最好的方法是创建Espresso test。你可以创建Fragment并mock一个ViewModel，因为Fragment仅仅和ViewModel沟通，mock一个ViewModel就足够全部的UI测试了。
- ViewModel: ViewModel可以使用JUnit来测试，仅仅需要mock一个UserRepository就够了。
- UserRepository: 你也可以用JUnit来测试UserRepository，需要mock一个webservice和一个DAO，你可以测试正确的webservice调用，存储结果到数据库；并且如果数据被缓存并且是新的，不需要做任何无用的网络请求。既然webservice和UserDao都是接口，那么你也可以通过mock它们或者创建一个假的实现来更复杂的测试用例。
- UserDao: 推荐使用instrumentation tests来测试UserDao。instrumentation tests不需要任何UI仍能快速地运行，对于每个测试，你可以创建一个内存数据库来确保测试不会造成任何的负面影响（例如修改了磁盘上的数据库文件）。Room也允许指定数据库的实现，这样你可以通过让JUnit，实现SupportSQLiteOpenHelper提供数据（这种方式不推荐，因为运行在当前设备上版本的SQLite可能和你实际的机器上的版本不同）。
- Webservice: 测试独立于外界是很重要的，因此webservice的测试应该避免调用你的后台服务，有很多类库可以帮助我们完成这个，例如：[MockWebServer](https://github.com/square/okhttp/tree/master/mockwebserver) 就是一个非常好的库，他可以帮助你为你的测试创建假的本地服务。
- Testing Artifacts, Architecture Components 提供了一个maven 构件 来控制后台线程，在`android.arch.core:core-testing`构件里, 有两个JUnit rules:
    - InstantTaskExecutorRule: 这个rule可以用来强制Architecture Components立刻执行在回调线程中的后台操作。
    - CountingTaskExecutorRule: 这个rule可以用在instrumentation tests中来等待Architecture Components的后台操作或者把它连接到Espresso作为一个idling resource.

### 最终架构
下图展示了我们推荐架构中的所有模块以及它们之间是如何交互的：
![最终架构](http://blog-1251624639.file.myqcloud.com/final-architecture.png)


## 指导原则
编程是一个有创造性的领域，开发Android应用也不例外。解决问题的方法有很多，例如：在多个Activity和Fragment之间传递数据，获取远程数据并为断网的情况持久化数据，或者任何特别的应用遭遇的其他常见场景。


下面的建议并不是强制的，我们的经验是采纳这些建议会让程序长期运行时更加健壮，更可测试，更可维护。

- 定义在manifest中的入口（Activity，Service，Broadcastreceiver等等）并不是数据源，相反，它们应该只协调与入口相关的数据子集。因为每个组件的生存时间都很短，这取决于用户在设备上的交互以及运行的健康状况，因此你你不想这些入口点成为数据源。

- 在应用程序的各个模块之间建立明确的职责界限时要毫不留情。例如，不要将从网络中加载数据的代码分散到代码基中的多个类或包中。类似地，不要将无关的职责（如数据缓存和数据绑定）填充到同一个类中。
- 尽可能少地从每个模块暴露。不要试图创建从一个模块中公开内部实现细节的“就那一个”快捷方式。你可能会获得一点短期内的时间，但是随着代码库的发展你将支付很多次技术债务。
- 在定义模块之间的交互时，考虑如何使每个模块在隔离中测试。例如，对于从网络中获取数据的定义良好的API，可以更容易地测试在本地数据库中保存该数据的模块。相反，如果你把这两个模块的逻辑混合在一个地方，或者在整个代码库中乱插入你的网络代码，那么测试起来会更困难，如果不是不可能的话。
- 是什么让你的应用脱颖而出。不要重新发明轮子或写同样的样板代码。相反，集中你的精力让你的应用程序的独特，让Android的架构组件和其他推荐的类库处理重复的样板。
- 尽可能多地持久化相关和新的数据，使您的应用程序是可用的，当设备处于脱机模式下。虽然您可以享受恒定和高速的连接，但您的用户可能不会。
- repository 应该指定一个数据源作为单一数据源。不管什么时候访问这些数据，都应该从这个单一数据源获取。更多信息参考[Single source of truth](https://developer.android.com/topic/libraries/architecture/guide.html#truth).

## 补遗: 暴露网路状态

在上面推荐的app架构部分中，我们有意省略网络错误和加载状态以保持示例简单。在本节中，我们演示了如何使用`Resource`类来公开网络状态以封装数据及其状态。

下面是一个示例实现：

```
//a generic class that describes a data with a status
public class Resource<T> {
    @NonNull public final Status status;
    @Nullable public final T data;
    @Nullable public final String message;
    private Resource(@NonNull Status status, @Nullable T data, @Nullable String message) {
        this.status = status;
        this.data = data;
        this.message = message;
    }

    public static <T> Resource<T> success(@NonNull T data) {
        return new Resource<>(SUCCESS, data, null);
    }

    public static <T> Resource<T> error(String msg, @Nullable T data) {
        return new Resource<>(ERROR, data, msg);
    }

    public static <T> Resource<T> loading(@Nullable T data) {
        return new Resource<>(LOADING, data, null);
    }
}

```
由于从网络加载数据而从磁盘呈现是一种常见的使用情况，我们将创建一个辅助类networkboundresource可以在多个地方重复使用。下面是networkboundresource决策树：

![network-bound-resource](http://blog-1251624639.file.myqcloud.com/network-bound-resource.png)

首它为了resource观察数据库。当入口是首次从数据库加载时，networkboundresource检查结果是否足够好用来分发和/或应从网络获取。请注意，这两种情况可能同时发生，因为你可能希望在从网络更新数据时显示缓存的数据。

如果网络调用成功完成，它将响应保存到数据库并重新初始化流程。如果网络请求失败，我们直接发送失败。

>注意：在将新数据保存到磁盘之后，我们重新初始化来自数据库的流，但是通常我们不需要这样做，因为数据库将发送更改。另一方面，依靠数据库来发送更改也会带来副作用，因为如果数据没有改变，数据库可能避免分发更改。我们也不希望分发从网络来的的结果，因为这将违背单一数据源原则（可能是数据库中的触发器会改变保存的值）。我们也不想在没有新数据的情况下发送成功状态，因为它会向客户端发送错误信息。

下面是NetworkBoundResource类给子类提供的公共API：

```
// ResultType: Type for the Resource data
// RequestType: Type for the API response
public abstract class NetworkBoundResource<ResultType, RequestType> {
    // Called to save the result of the API response into the database
    @WorkerThread
    protected abstract void saveCallResult(@NonNull RequestType item);

    // Called with the data in the database to decide whether it should be
    // fetched from the network.
    @MainThread
    protected abstract boolean shouldFetch(@Nullable ResultType data);

    // Called to get the cached data from the database
    @NonNull @MainThread
    protected abstract LiveData<ResultType> loadFromDb();

    // Called to create the API call.
    @NonNull @MainThread
    protected abstract LiveData<ApiResponse<RequestType>> createCall();

    // Called when the fetch fails. The child class may want to reset components
    // like rate limiter.
    @MainThread
    protected void onFetchFailed() {
    }

    // returns a LiveData that represents the resource, implemented
    // in the base class.
    public final LiveData<Resource<ResultType>> getAsLiveData();
}

```
注意：上面的类定义了两个类型的参数（ResultType，RequestType），因为从API返回的数据可能与本地的数据类型不匹配。

同样注意：上面的代码中的网络请求使用了ApiResource，ApiResource是Retrofit2.Call类的一个简单封装，用来将response转成LiveData。

下面是NetworkBoundResource类剩下的实现：

```
public abstract class NetworkBoundResource<ResultType, RequestType> {
    private final MediatorLiveData<Resource<ResultType>> result = new MediatorLiveData<>();

    @MainThread
    NetworkBoundResource() {
        result.setValue(Resource.loading(null));
        LiveData<ResultType> dbSource = loadFromDb();
        result.addSource(dbSource, data -> {
            result.removeSource(dbSource);
            if (shouldFetch(data)) {
                fetchFromNetwork(dbSource);
            } else {
                result.addSource(dbSource,
                        newData -> result.setValue(Resource.success(newData)));
            }
        });
    }

    private void fetchFromNetwork(final LiveData<ResultType> dbSource) {
        LiveData<ApiResponse<RequestType>> apiResponse = createCall();
        // we re-attach dbSource as a new source,
        // it will dispatch its latest value quickly
        result.addSource(dbSource,
                newData -> result.setValue(Resource.loading(newData)));
        result.addSource(apiResponse, response -> {
            result.removeSource(apiResponse);
            result.removeSource(dbSource);
            //noinspection ConstantConditions
            if (response.isSuccessful()) {
                saveResultAndReInit(response);
            } else {
                onFetchFailed();
                result.addSource(dbSource,
                        newData -> result.setValue(
                                Resource.error(response.errorMessage, newData)));
            }
        });
    }

    @MainThread
    private void saveResultAndReInit(ApiResponse<RequestType> response) {
        new AsyncTask<Void, Void, Void>() {

            @Override
            protected Void doInBackground(Void... voids) {
                saveCallResult(response.body);
                return null;
            }

            @Override
            protected void onPostExecute(Void aVoid) {
                // we specially request a new live data,
                // otherwise we will get immediately last cached value,
                // which may not be updated with latest results received from network.
                result.addSource(loadFromDb(),
                        newData -> result.setValue(Resource.success(newData)));
            }
        }.execute();
    }

    public final LiveData<Resource<ResultType>> getAsLiveData() {
        return result;
    }
}

```
现在, 我们可以使用NetworkBoundResource来在repository中编写磁盘和网络绑定User类的实现了。

```
class UserRepository {
    Webservice webservice;
    UserDao userDao;

    public LiveData<Resource<User>> loadUser(final String userId) {
        return new NetworkBoundResource<User,User>() {
            @Override
            protected void saveCallResult(@NonNull User item) {
                userDao.insert(item);
            }

            @Override
            protected boolean shouldFetch(@Nullable User data) {
                return rateLimiter.canFetch(userId) && (data == null || !isFresh(data));
            }

            @NonNull @Override
            protected LiveData<User> loadFromDb() {
                return userDao.load(userId);
            }

            @NonNull @Override
            protected LiveData<ApiResponse<User>> createCall() {
                return webservice.getUser(userId);
            }
        }.getAsLiveData();
    }
}
```

谷歌官方文档[Guide to App Architecture](https://developer.android.com/topic/libraries/architecture/guide.html)


