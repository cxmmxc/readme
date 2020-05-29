### Glide流程

#### Glide的前期配置及build过程

* 一般使用方式

  ```GldieApp.with(context).load("url").into(imageview)```

  使用非常简单，让我们来一步步解析

  1. ```GlideApp.with(context)``` ，首先说`GlideApp`的来源,手动写一个类继承`AppGlideModule`，给类加上注解`@GlideModule`，这里 会自动生成`GlideApp`类，如果给注解加上`value`，比如`@GlideModule(glideName="GlideNewApp")`，那么则会生成`GlideNewApp`的辅助类

  2. 继承`AppGlideModule`可以自定义很多功能

     ```java
     @Override
     public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
       super.applyOptions(context, builder);
       builder.setDefaultRequestOptions(new RequestOptions().format(DecodeFormat.PREFER_ARGB_8888));
     }
     ```

     例如这个方法中，可以对`GlideBuilder`的各个属性进行定制，例如设置自己的`MemoryCache` `DiskCache.Factory`等

     ```java
     @Override
     public void registerComponents(
         @NonNull Context context, @NonNull Glide glide, @NonNull Registry registry) {
       registry.append(Photo.class, InputStream.class, new FlickrModelLoader.Factory());
     }
     ```

     这个方法中，通过registry来注册自己的解析对象和解析器，例如`Glide`提供的`com.github.bumptech.glide:okhttp3-integration:4.11.0`就是提供了基于`OkHttp`的网络请求(默认是`HttpUrlConnection`).

     可以用如下代码替换为`OkHttp`的网络请求

     ```java
     registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory(okHttpClient));
     ```

* `GlideApp.with(context)`分析

  调用的是

  `GlideApp.java`

  ```java
  (GlideRequests) Glide.with(context);
  ```

  `Glide.java`

  ```java
  getRetriever(context).get(context);
  ```

  这里需要调查`getRetriever(context)`的作用，也就是返回`RequestManagerRetriever`的功能和作用。

  ```java
  Glide.get(context).getRequestManagerRetriever();
  ```

  这里通过Glide来获取`RequestManagerRetriever`

  发现`RequestManagerRetriever`是在`Glide`构造函数传进来的，`Glide`构造是通过`GlideBuilder`来进行构造的，此时就需要对`Glide.get(context)`来进行解析了

* `Glide.get(context)`解析

  `Glide`是单例模式，首先检查是否有注解生成类`GeneratedAppGlideModuleImpl`

  然后拿着生成类取实例化`Glide`，这个方法`initializeGlide`传入了一个`new GlideBuilder()`。

  

   ```Java
    new ManifestParser(applicationContext).parse()
   ```

    通过`ManifestParser`来解析`AndroidManifest.xml`中标明的`META_DATA`字段

    **这段是早期的解析方式，现在是通过继承`AppGlideModule`，加`@GlideModule`注解自动parse**

    ```java
    RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory()
            : null;
    ```

    这里是设置`Builder`里的`RequestManagerFactory`

    ```java
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.applyOptions(applicationContext, builder);
    }
    ```

    这里是针对有继承`AppGlideModule`，并对`applyOptions`方法里的`Builder`进行过配置，这里用的很巧妙，可以让用户针对性的对`Builder`进行配置

    `Glide glide = builder.build(applicationContext)`

    这里才是获得到的`Glide`对象，`build`方法里`new RequestManagerRetriever(requestManagerFactory)`对应了`Glide.get(context).getRequestManagerRetriever()`获得的`RequestManagerRetriever`对象

* `RequestManagerRetriever`的作用

  `Glide.with(context)`就是通过`getRetriever(activity).get(activity/fragment/context)`来获取到`RequestManager`

  **`get(activity/fragment/xxxx)`方法解析**

  - 如果不是在UI线程，通过`context.getApplicationContext()`获取					`ApplicationContext`

    调用`get(Context)`方法，参数为`context`

    ```java
  public RequestManager get(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper
          // Only unwrap a ContextWrapper if the baseContext has a non-null application context.
          // Context#createPackageContext may return a Context without an Application instance,
          // in which case a ContextWrapper may be used to attach one.
          && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }
  
    return getApplicationManager(context);
  }
    ```

    经过判断，此处为非`UI`线程，所以最后会走`getApplicationManager(context)`方法

    ```java
  Glide glide = Glide.get(context.getApplicationContext());
  applicationManager =
      factory.build(
          glide,
          new ApplicationLifecycle(),
          new EmptyRequestManagerTreeNode(),
          context.getApplicationContext());
    ```

  ​	此处最终会得到一个`RequestManager`

  - `get(Activity/FragmentActivity/Fragment)`会直接

    `Activity`类型调用`fragmentGet(Activity,FragmentManager,(Fragment)ParentHint, isVisible)`来返回`RequestManager`**`fragmentGet()方法已经过时，建议用supportFagmentGet()`**。

    `FragmentActivity`和`Fragment`类型调用`supportFragmentGet(context, FragmentManager,Fragment,isVisisble)`来返回`Requestmanager`。

    通过`getSupportRequestManagerFragment(fm, parentHint, isParentVisible)`获取`SupportRequestManagerFragment`，在这个类里`set` `get`设置和获取`RequestManager`

    如果获取的`RequestManager`是`null`，则：

    ```java
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    ```

    `getSupportRequestManagerFragment(xx,xx,xx)`方法：首先根据`TAG`，使用`fragmentManager`来获取`SupportRequestManagerFragment`,没有就试图从`pendingSupportRequestManagerFragments`（着是一个`HashMap`）来拿，还是`null`，则`new`一个，并设置`parentHint` ，如果`isVisible`为`true`，则调用回调`SupportRequestManagerFragment`内的`GlideLifecycle.start()`方法，同时以`fragmentManager`为`key`,以`SupportRequestManagerFragment`为value存入`HashMap`，并将`Fragment` 通过`fragmentManager`进行`add并commit` 。通过`Hander`发送`ID_REMOVE_SUPPORT_FRAGMENT_MANAGER`和`fm`，目的是将`fm`从`pendingSupportRequestManagerFragments`这个`hashmap`中移除掉。

    **对于这里的理解：对于一个`Activity`或者一个`Fragment`，都创造一个对应的`Fragment`或者子`Fragment`，并将`Lifecycle`和`RequestManager`赋值进去，对`RequestManager`分发的网络请求对应上`Fragment`的生命周期，所以才说Glide是自动管理生命周期的**

    **以上就告一段落，获取到了`RequestManager`**

* **这里着重看一下`GlideBuilder.build()`方法构造Glide的过程**

  * `build()`方法里

    ```java
    sourceExecutor = GlideExecutor.newSourceExecutor();
    diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    animationExecutor = GlideExecutor.newAnimationExecutor();
    .....
    memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    //这里最重要Engine，在这里实例化了
    engine =
              new Engine(
                  memoryCache,
                  diskCacheFactory,
                  diskCacheExecutor,
                  sourceExecutor,
                  GlideExecutor.newUnlimitedSourceExecutor(),
                  animationExecutor,
                  isActiveResourceRetentionAllowed);
    ```

  * `Glide(xx,xx,xx,...)`的构造方法里

    ```java
    //此构造函数默认已经实例化了很多Registry，例如：ModelLoaderRegistry，EncoderRegistry，ResourceDecoderRegistry，ResourceEncoderRegistry，DataRewinderRegistry，TranscoderRegistry，ImageHeaderParserRegistry
    registry = new Registry();
    //注册了默认的解析Image头的解析类
    registry.register(new DefaultImageHeaderParser());
    //下面registry注册了N多不同类型的加载类和对应的处理器
    registry
            .append(
                Registry.BUCKET_BITMAP,
                ParcelFileDescriptor.class,
                Bitmap.class,
                parcelFileDescriptorVideoDecoder)
    //实例化图片解码和旋转的类
    new Downsampler()
    //这里为了适配Android Q(29)，就是拿到的图片都是Uri的方式来展示
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
          registry.append(
              Uri.class, InputStream.class, new 									      QMediaStoreUriLoader.InputStreamFactory(context));
          registry.append(
              Uri.class,
              ParcelFileDescriptor.class,
              new QMediaStoreUriLoader.FileDescriptorFactory(context));
    }
    //实例化ImageViewTargetFactory
    new ImageViewTargetFactory()
    //实例化GlideContext，相当于Glide的上下文
    new GlideContext(
                context,
                arrayPool,
                registry,
                imageViewTargetFactory,
                defaultRequestOptionsFactory,
                defaultTransitionOptions,
                defaultRequestListeners,
                engine,
                isLoggingRequestOriginsEnabled,
                logLevel)
    ```
    
    **`Glide`的构造方法里，实例化了N多对象。例如：`ImageHeader`默认解析器，`ExifImage`的解析器，`Downsampler`图片解码旋转，各种类型的`Decoder`，以及通过register来注册各种各种格式的`url`及对应的解析类。并将这么多类集中写入到`GlideContext`上下文中**

* `RequestManager.load(url/string/xxx)`

  默认情况下，会调用`asDrawable()`

  而`asDrawable()`方法会调用`RequestBuilder`的构造函数`new RequestBuilder<>(glide, this, resourceClass, context)`。构造方法里会调用`super.apply(RequestOption)`将设置的`Options`设置到`RequestBuilder`中。

  进而再次调用`RequestBuilder`的l`load(String/Uri/xxxx)`方法

* `RequestBuilder.load(ImageView/Target)`

  `load（xxxx）`这里会调用`loadGeneric（Object）`方法，只是设置了`model`，并且设置`isModelSet = true`。

  最后走最简单的`into(ImageView/Target)`方法

  以最简单的`into(ImageView)`为例：

  ```java
  into(
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions,
      Executors.mainThreadExecutor());
  ```

  第一次参数是`Target`，这里`ImageView`是没有`Target`的，所以在此处创建了了`Target`,我们进入到`buildImageViewTarget（）`方法中查看，最终会走到`ImageViewTargetFactory.buildTarget()`，这里只会创建两种`Target`，根据传入的`clazz`来判断：

  ```java
  if (Bitmap.class.equals(clazz)) {
    return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
  } else if (Drawable.class.isAssignableFrom(clazz)) {
    return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
  } else {
    throw new IllegalArgumentException(
        "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
  }
  ```

  可见，只能有`Bitmap`和`Drawable`两种`clazz`可选，否则就会抛异常。走的默认的都是`Drawable`类型

* 终极方法`into(Target, targetListener, options, excutor)`

  * 首先`buildRequest(target, targetListener, options, callbackExecutor)`，生成`request`。

    这里最终会调用`buildRequestRecursive()`方法

    `buildThumbnailRequestRecursive(xx/xx/xx)`来生成`mainRequest`

    这里判断`thumbnailBuilder`是否为`null`，不为`null`，则构建`fullRequest和thumbRequest`,同时构造`ThumbnailRequestCoordinator`协调对象，并将`fullRequest`和`thumbRequest`添加进去，进行协作管理。后面有缩略图配置的发起点就是这个类

    `thumbnail`相关为`null`的话，则直接走`obtainRequest(xxxxx/xxx/xx)`,这里直接构建了`SingleRequest`对象，后面的开始请求网络的发起点就是这个类

  * 然后通过传入的`target`来获取`Request`，并和上面的`build`的`Request`进行比对，通过`isEquivalentTo()`进行比对，返回`true`的话，则使用`target`里的`Request`的进行请求`request.begin()`

    如果返回为`false`的话，则通过`RequestTracker`来进行`track()`，这里就就是加载图片的流程的开始了，调用了`request.begin()`方法`(调用SingleRequest或者ThumbnailRequestCoordinator这个的begin()方法)`
    
    ------------------

#### 图片`url`(路径)的加载流程

* 这里从`SingleRequest.begin()`方法，开启了真正的请求之路

  判断`model`是否为`null`，这里的`model`其实就是`load(url/string/xxx)`里的参数；

  判断`status`的状态，如果为`Status.RUNNING`，则直接抛出不能重启一个正在运行的`request`

  判断`status为Status.COMPLETE`状态，则直接进入；`onResourceReady(resource,DataSource.MEMORY_CACHE)`。这里有个条件：就是你请求的Target或View必须是完全相同，这个方法放在后面再讲；

  这里将`status = Status.WAITING_FOR_SIZE`,表明需要等待`target`去获取对应`ImageView`的`with/ height`，如果都是`width height`都是有效的，则进入`onSizeReady(with,height)`，否则需要调用`target`的`getSize()`方法，最终调用的其实是`ViewTarget`的`getTargetWidth() getTargetHeight()`方法；

  判断`status == Status.RUNNING || status == Status.WAIT_FOR_SIZE || canNotifyStatusChanged()`，则调用`target.onLoadStarted()`方法，标明正式开始了；

  上面说过，`target.getSize(this)`，成功之后会回调`onSizeReady(with,height)`；

  重头戏就从这里开始：

  将`status = Status.RUNNING`，赋值状态

  然后通过`engine.load(xxxxxxxxxxxxxxx)`开始请求数据

* `Engine`再`GlideBuilder.build()`的时候进行初始化的，将`memoryCache,diskCacheFactory,discCacheFactory,sourceExcutor`等通过构造参数放到`Engine`内；

  首先通过`keyFactory.buildKey(xxxxxxx)`来构造需要缓存的`EngineKey`，这个`key`的`equas`方法决定了两个`key`是否相同，决定了是否直接从缓存里拿；

  首先`loadFromMemory(key,isMemoryCacheable,startTime)`，如果你在`Glide.with().skipMemory(true/false)`这里设置的`true`，则证明不走内存缓存，直接返回为`null`

  这个`loadFromMemory`得说道说道，首先调用`loadFromActiveResources(key)`，这里会调用`ActiveResources`类的方法，从它的`HashMap`集合中通过`key`来获取。既然是获取，那就 有`put`的地方，这里是在`Engine的onEngineJobComplete()`方法里调用，即再`Engine`调用流程完成后来存到内存中~第二个`put`的地方是`Engine的loadFromCache()`l里，通过`Cache`拿到对应的`resource`，不为`null`，则调用`activeResources.active(key,cached)`方法存入到内存中。如果返回的也是`null`的话，则调用`waitForExistingOrStartNewJob()`方法

* `waitForExistingOrStartNewJob()`真正的网络拉取方法

  `jobs.get(key, onlyRetrieveFromCache)`通过`key`来判断是否有已存的`EngineJob`，有的话，直接`return new LoadStatus(cb, current)`。这里的`cb`不为空，则会直接调用对应的`EngineJob的addCallback()`方法，这里会调用后面的`current.excute(new CallResourceReady(cb))`或`current.execute(new CallLoadFailed(cb))`方法来，这里是线程池的调用，会启动`CallResourceReady`的`run()`方法。这里就是`onResourceReady()`之类的方法了；

  如果`jobs.get()`返回的 是`null`的话，只能从网络获取了。这里构造了一个`EngineJob`和`DecodeJob`两个实例，注意这里`jobs.put(key, engineJob);`这里就对应了上面的`jobs.get()`的`EngineJob`，然后调用

  ```java
  engineJob.addCallback(cb, callbackExecutor);
  engineJob.start(decodeJob);	
  ```

  这里就开启了真正的请求网络以及编解码的地方，真正的方法在`decodeJob`里的`run()`方法

  **`run()`**

  首先将当前的`currentFetcher`赋予局部变量`localFetcher`，并在最后执行`clean()`方法。

  核心方法`runWrapped()`：

  ​	首先`INITIALIZE`： `getNextStage(Stage.INITIALIZE)`直到获取到`stage`为`Stage.DATA_CACHE`。

  ​	然后`getNextGenerator()`来获取对应的`ResourceCacheGenerator`

  ​	然后调用`runGenerators()`这里会调用上面的`startNext()`方法

  `startNext()`方法解析：
  
  	1. 首先`helper.getCacheKeys()`方法获取所有缓存的`keys`，如果为空的，直接返回`false`,然后`helper.getRegisteredResourceClasses()`：注意这个方法里面的`transcodeClass`一般为`as(xx)`里的`xx`的`class`类型，例如`as(Bitmap.class)`。`model`来说一般是加载是`url`路径，有`String uri inputStream`等等，`transcdoeClass`一般来说是用户在`Glide.with(xx).transcode(xxx)`里对应的`xxx`，如果从缓存拿到的是`null`，则通过`modelLoaderRegistry.getDataClasses(modelClass)`来获取`model例如GlideUrl.class`对应的所有的`dataclass 例如：OkHttpDataFetcher`，然后通过`decoderRegistry用dataClass resourceClass`获取对应注册的`ResourceClass`，再用`transcoderRegistry `通过获取到的`registerdResourceClass和trancodeClass`进一步筛选，找到符合的`registerdResourceClass`集合
   	2. ，获取所有已注册`resourceClasses`对象集合。如果获取到的是空的，则判断传入的`TranscodeClass`类	型，如果是`File`类型，则直接返回`false`
   	3. 通过`while`循环`modelLoaders`，

