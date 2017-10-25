title: android-coding
date: 2017-10-23 21:45:38
tags:
  - Android
  - Coding
  - Github Restful API
  - OAuth
---

## 简述

使用 okhttp 很长时间了，但一直没时间弄懂其中各种奥妙。上周相对空闲，于是阅读了 okhttp 源码，准备写写笔记。习惯写笔记时，写写比较核心的demo代码，想起前阵子朋友使用 github 的 rest api v3 开放接口实现了评论系统 comment.js ，于是 demo 代码中的网络请求就试试 github 的 rest api v3 开放接口。又因为之前也实现过类似 coding 一样的功能，一发不可收拾，便实现一个可关注动态，下载源码，查看文档和查看代码的 coding ，本文是 coding 的分享笔记，以后再做 okhttp 源码笔记。

`开放接口` : [`https://developer.github.com/v3/`](`https://developer.github.com/v3/`)

`项目地址` : [`https://github.com/4ndroidev/Coding`](https://github.com/4ndroidev/Coding)

如果你的项目想集成该功能，拷贝 coding 代码后，像 sample 一样，添加一行代码即可。

> **声明：感谢真coding，此假coding使用了真coding开源代码中的3个html和一些图标**

## 效果

{% dplayer "width=360px" "height=640px" "url=/images/android-coding/coding.mp4" "pic=/images/android-coding/coding.png" "loop=yes" "autoplay=false" %}

<!-- more -->

## 代码结构

### 项目依赖

coding 使用了一些热门库：

- [`okhttp`](https://github.com/square/okhttp)
- [`retrofit`](https://github.com/square/retrofit)
- [`rxjava`](https://github.com/ReactiveX/RxJava)
- [`jackson`](https://github.com/FasterXML/jackson)
- [`glide`](https://github.com/bumptech/glide)

此外，项目使用了`lambda`表达式结合`rxjava`，使用假`mvp`结构（后续说明），代码比较简洁

### 目录结构

```
coding/src/main/java/com/androidev/coding/
├── misc             // 常量和工具类
├── model            // 实体类                                                  
├── module           // 功能模块
│   ├── auth         // github登录授权
│   ├── base         // 基础类
│   ├── code         // 代码目录及展示页面
│   ├── commit       // commit列表与单个commit的diff列表页面
│   ├── image        // 图片展示页面
│   └── main         // 主页
├── network          // 网络模块
│   └── interceptor  // 网络请求拦截器，主要用于登录鉴权，增大访问限制
└── widget           // UI组件
```

### mvp结构

> 假`mvp`，这里没有像`google`推荐的那样`view`层和`presenter`层定义成接口存于一个`contract`类中。because `view`层和`presenter`层目前都只有单一实现，无需定义成接口，方便定位问题。老实说，个人认为不应随便定义接口，项目大了后容易出现混乱，接口最好在多态或回调时定义，尽可能减少单一实现的接口定义，maybe 这是个歪理。

```code
coding/src/main/java/com/androidev/coding/module
├── auth
│   ├── AuthActivity.java
│   └── AuthPresenter.java
├── base
│   └── BaseActivity.java
├── code
│   ├── CodeActivity.java
│   ├── CodePresenter.java
│   ├── TreeActivity.java
│   ├── TreePresenter.java
│   └── adapter
│       └── TreeAdapter.java
├── commit
│   ├── CommitActivity.java
│   ├── CommitPresenter.java
│   ├── CommitsActivity.java
│   ├── CommitsPresenter.java
│   └── adapter
│       ├── CommitAdapter.java
│       └── CommitsAdapter.java
├── image
│   └── ImageActivity.java
└── main
    ├── MainFragment.java
    └── MainPresenter.java
```

## 开发题纲

- 定义请求接口，创建 retrofit 对象
- 编写 view 和 presenter，实现查阅功能
- 实现 oauth 授权功能，增大访问限制

### 请求接口定义

github rest api v3 支持挺丰富的，而目前需要的信息: repo, commit, tree 和 blob，其他以后再关注。

在定义接口时，往往需要知道实体类内容，可以直接到[`https://developer.github.com/v3/`](`https://developer.github.com/v3/`)找相关 json 格式，或自行在浏览器中发起请求。实体类可以通过 Android Studio 的 GsonFormat 插件直接生成。


定义请求接口代码

```java
public interface RestApi {

    String REPO_FORMAT = "repos/{owner}/{repo}";

    @GET(REPO_FORMAT)
    Observable<Repo> repo(@Path("owner") String owner, @Path("repo") String repo);

    @GET(REPO_FORMAT + "/commits")
    Observable<List<Commit>> commits(@Path("owner") String owner, @Path("repo") String repo, @QueryMap Map<String, Object> data);

    @GET(REPO_FORMAT + "/commits/{sha}")
    Observable<Commit> commit(@Path("owner") String owner, @Path("repo") String repo, @Path("sha") String sha);

    // 暂时用不到
    @GET(REPO_FORMAT + "/branches/{branch}")
    Observable<Branch> branch(@Path("owner") String owner, @Path("repo") String repo, @Path("branch") String branch);

    @GET(REPO_FORMAT + "/git/trees/{sha}")
    Observable<Tree> tree(@Path("owner") String owner, @Path("repo") String repo, @Path("sha") String sha);

    // 暂时用不到
    @GET(REPO_FORMAT + "/git/blobs/{sha}")
    Observable<Blob> blob(@Path("owner") String owner, @Path("repo") String repo, @Path("sha") String sha);

    @GET(REPO_FORMAT + "/git/blobs/{sha}")
    @Headers({HEADER_ACCEPT + ": " + MEDIA_TYPE_RAW})
    Observable<ResponseBody> raw(@Path("owner") String owner, @Path("repo") String repo, @Path("sha") String sha);
}
```

创建 retrofit 对象代码

```java
public void initialize(Context context) {
        authorizeInterceptor = new AuthorizeInterceptor();
        authorizeInterceptor.setToken(context.getSharedPreferences(APP, Context.MODE_PRIVATE).getString(KEY_TOKEN, ""));
        File cachePath = new File(context.getExternalCacheDir(), "coding");
        okHttpClient = new OkHttpClient.Builder()
                .cache(new Cache(cachePath, 30 * 1024 * 1024/* 30MB */))
                .addInterceptor(authorizeInterceptor)
                .addNetworkInterceptor(new RateLimitInterceptor())
                .build();
        objectMapper = new ObjectMapper();
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .callFactory(okHttpClient)
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .addConverterFactory(JacksonConverterFactory.create(objectMapper))
                .build();
        restApi = retrofit.create(RestApi.class);
        rateLimit = new RateLimit();
        uiHandler = new Handler(Looper.getMainLooper());
        onRateLimitChangedListeners = new ArrayList<>();
    }
```

### view & presenter

mvp 目前比较热门，将业务功能从 view 层抽离到 presenter 中才处理，presenter 作为中间层，联系 model 和 view， view 不会对 model 写数据，只会读取 model 数据进行展示。这样一方面代码相对清晰，另一方面更便于维护。

view 层代码中，如大部分 app 一样，coding 使用 recyclerview 展示列表数据，包括 commit 列表，diff 列表和代码目录；代码展示和 markdown 展示则使用 webview 加载模板与内容。

CommitsActivity.java

```java
public class CommitsActivity extends BaseActivity {

    private RefreshLayout mRefreshLayout;
    private CommitsAdapter mAdapter;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        SwipeBackLayout.attachTo(this);
        setContentView(R.layout.coding_activity_commits);
        CommitsPresenter presenter = new CommitsPresenter(this);
        mAdapter = new CommitsAdapter();
        mAdapter.setOnLoadListener(presenter::load);
        mAdapter.setOnItemClickListener((v, position, data) -> {
            Intent intent = getIntent();
            intent.putExtra(SHA, data.sha);
            intent.setClass(this, CommitActivity.class);
            startActivity(intent);
        });
        mRefreshLayout = (RefreshLayout) findViewById(R.id.coding_refresh_layout);
        mRefreshLayout.setOnRefreshListener(presenter::refresh);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.coding_recycler_view);
        recyclerView.setLayoutManager(new LinearLayoutManager(this, LinearLayoutManager.VERTICAL, false));
        recyclerView.setAdapter(mAdapter);
        presenter.refresh();
        showLoading(); //just one time
    }

    void setData(List<Commit> commits) {
        dismissLoading();
        mRefreshLayout.setRefreshing(false);
        mAdapter.setData(commits);
    }

    void setError(Throwable throwable) {
        dismissLoading();
        mRefreshLayout.setRefreshing(false);
        throwable.printStackTrace();
    }

    void appendData(List<Commit> commits) {
        setLoading(false);
        mAdapter.appendData(commits);
    }

    void setLoading(boolean loading) {
        mAdapter.setLoading(loading);
    }
}
```

CommitsPresenter.java 

```java
class CommitsPresenter {

    private final static int PAGE_NO = 1;
    private final static int PER_PAGE = 20;

    private String mSha;
    private String mOwner;
    private String mRepo;
    private CommitsActivity mView;

    CommitsPresenter(CommitsActivity view) {
        mView = view;
        Intent intent = mView.getIntent();
        mOwner = intent.getStringExtra(OWNER);
        mRepo = intent.getStringExtra(REPO);
    }

    void refresh() {
        Map<String, Object> data = new HashMap<>();
        data.put("page", PAGE_NO);
        data.put("per_page", PER_PAGE);
        GitHub.getApi().commits(mOwner, mRepo, data)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .doAfterNext(this::setSha)
                .subscribe(mView::setData, mView::setError);
    }

    void load() {
        mView.setLoading(true);
        Map<String, Object> data = new HashMap<>();
        data.put("page", PAGE_NO);
        data.put("per_page", PER_PAGE);
        data.put("sha", mSha);
        GitHub.getApi().commits(mOwner, mRepo, data)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .doAfterNext(this::setSha)
                .subscribe(mView::appendData, mView::setError);
    }

    private void setSha(List<Commit> commits) {
        if (commits == null || commits.size() == 0) return;
        List<Commit.Parents> parents = commits.get(commits.size() - 1).parents;
        if (parents == null || parents.size() == 0) return;
        mSha = parents.get(0).sha;
    }

}

```
