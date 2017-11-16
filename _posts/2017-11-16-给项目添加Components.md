Architecture Components可以在谷歌的Maven仓库中找到，下面是使用步骤：
## 添加谷歌的Maven仓库
默认情况下，Android Studio的项目并没有配置谷歌的仓库，需要你打开项目的根`build.gradle`文件，然后手动添加，代码如下：

```
allprojects {
    repositories {
        jcenter()
        maven { url 'https://maven.google.com' }
    }
}
```
## 添加Architecture Components
打开项目的`build.gradle`文件，添加你需要的依赖

- Lifecycles
    - 
`implementation "android.arch.lifecycle:runtime:1.0.3" // not necessary if you are using lifecycle:extensions or lifecycle:common-java8
`
    - 
`annotationProcessor "android.arch.lifecycle:compiler:1.0.0" // not needed if you are using the DefaultLifecycleObserver from common-java8 artifact.
`
    - 如果需要支持Java8, 添加:
        - `implementation "android.arch.lifecycle:common-java8:1.0.0"`
        

- LiveData和ViewModel
    - `implementation "android.arch.lifecycle:extensions:1.0.0"`
    - 测试, 添加:
        - `testImplementation "android.arch.core:core-testing:1.0.0"`
    - 使用 ReactiveStreams API, 添加:
        - `implementation "android.arch.lifecycle:reactivestreams:1.0.0"`
- Room
    - `implementation "android.arch.persistence.room:runtime:1.0.0"`
    - `annotationProcessor "android.arch.persistence.room:compiler:1.0.0"`
    - 测试 Room migrations, 添加:
        - `testImplementation "android.arch.persistence.room:testing:1.0.0"`
    - RxJava支持, 添加:
        - `implementation "android.arch.persistence.room:rxjava2:1.0.0"`
- Paging
    - `implementation "android.arch.paging:runtime:1.0.0-alpha3"`

更多信息，参考[Add Build Dependencies](https://developer.android.com/studio/build/dependencies.html)
谷歌官方文档[Adding Components to your Project](https://developer.android.com/topic/libraries/architecture/adding-components.html)

