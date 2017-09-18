title: 安卓下载任务管理
categories: Android
tags:
  - Android
  - 下载管理
date: 2017-04-19 20:45:19
---

> 前言：上年开发了一个壁纸，音乐，应用，视频等资源浏览和下载安卓应用，准备分解功能模块做下笔记。下载页面UI设计参照 **网易云音乐**

下载功能

* 多任务并行下载
* 断点续传（需服务器支持）

项目地址：[https://github.com/4ndroidev/DownloadManager.git](https://github.com/4ndroidev/DownloadManager.git)

<!-- more -->

效果图

![image](/images/android-download-manager/download-screenshot.jpg)

# 实现原理

下载任务流程图

![image](/images/android-download-manager/download-task-flow.png)

由上图可知，任务执行流程大致如下

1. 创建任务，并做准备，设置监听器等操作
2. 根据任务创建实际下载工作，添加到任务队列，等待或直接执行
3. 用户操作，进行暂停，恢复，或删除

# 核心类分析

|类|功能|
|---|---|
|DownloadTask|下载任务，保存部分关键信息，非实际下载工作|
|DownloadInfo|下载信息，保存所有信息|
|DownloadJob|实现Runnable接口，实际下载工作，负责网络请求，数据库信息更新|
|DownloadManager|单例，创建下载任务，提供获取正在下载任务，所有下载信息，设置监听器等接口|
|DownloadEngine|负责创建线程池，根据任务创建下载工作，调度工作及通知|
|DownloadProvider|负责下载信息数据库增删查改|

# 类关联关系

|关联|关系|
|---|---|
| **DownloadTask** - **DownloadInfo** | n - 1 |
| **DownloadTask** - **DownloadJob** | n - 0...1 |
| **DownloadJob** - **DownloadInfo** | 1 - 1 |

# 下载工作

断点续传的关键点：

- 使用**Range**这个**Header**来指定开始下载位置
- 文件读写则使用**RandomAccessFile**，可在指定偏移量读写文件
- 注意**RandomAccessFile**打开模式不要加入`s`，同步模式会拖慢下载速度
  ```java
  package com.grocery.download.library;

  import java.io.File;
  import java.io.IOException;
  import java.io.InputStream;
  import java.io.RandomAccessFile;
  import java.net.HttpURLConnection;
  import java.net.URL;
  import java.util.ArrayList;
  import java.util.List;

  import static com.grocery.download.library.DownloadState.STATE_FAILED;
  import static com.grocery.download.library.DownloadState.STATE_FINISHED;
  import static com.grocery.download.library.DownloadState.STATE_PAUSED;
  import static com.grocery.download.library.DownloadState.STATE_RUNNING;
  import static com.grocery.download.library.DownloadState.STATE_WAITING;

  /**
   * Created by 4ndroidev on 16/10/6.
   */

  // one-to-one association with DownloadInfo
  public class DownloadJob implements Runnable {

    private boolean isPaused;
    private DownloadInfo info;
    private DownloadEngine engine;
    private List<DownloadListener> listeners;

    private Runnable changeState = new Runnable() {
      @Override
      public void run() {
        synchronized (DownloadJob.class) {
          for (DownloadListener listener : listeners) {
            listener.onStateChanged(info.key, DownloadJob.this.info.state);
          }
          switch (info.state) {
            case STATE_RUNNING:
              engine.onJobStarted(info);
              break;
            case STATE_FINISHED:
              engine.onJobCompleted(true, info);
              clear();
              break;
            case STATE_FAILED:
            case STATE_PAUSED:
              engine.onJobCompleted(false, info);
              break;
          }
        }
      }
    };

    private Runnable changeProgress = new Runnable() {
      @Override
      public void run() {
        synchronized (DownloadJob.class) {
          for (DownloadListener listener : listeners) {
            listener.onProgressChanged(info.key, DownloadJob.this.info.finishedLength, DownloadJob.this.info.contentLength);
          }
        }
      }
    };

    public DownloadJob(DownloadEngine engine, DownloadInfo info) {
      this.engine = engine;
      this.info = info;
      this.listeners = new ArrayList<>();
    }

    DownloadInfo getInfo() {
      return info;
    }

    void addListener(DownloadListener listener) {
      synchronized (DownloadJob.class) {
        if (listener == null || listeners.contains(listener)) return;
        listener.onStateChanged(info.key, info.state);
        listeners.add(listener);
      }
    }

    void removeListener(DownloadListener listener) {
      synchronized (DownloadJob.class) {
          if (listener == null || !listeners.contains(listener)) return;
          listeners.remove(listener);
      }
    }

    boolean isRunning() {
      return STATE_RUNNING == info.state;
    }

    void enqueue() {
      resume();
    }

    void pause() {
      isPaused = true;
      if (info.state != STATE_WAITING) return;
      onStateChanged(STATE_PAUSED, false);
    }

    void resume() {
      if (isRunning()) return;
      onStateChanged(STATE_WAITING, false);
      isPaused = false;
      engine.executor.submit(this);
    }

    private void clear() {
      listeners.clear();
      engine = null;
      info = null;
    }

    private void onStateChanged(int state, boolean updateDb) {
      info.state = state;
      if (updateDb) engine.provider.update(info);
      engine.handler.removeCallbacks(changeState);
      engine.handler.post(changeState);
    }

    private void onProgressChanged(long finishedLength, long contentLength) {
      info.finishedLength = finishedLength;
      info.contentLength = contentLength;
      engine.handler.removeCallbacks(changeProgress);
      engine.handler.post(changeProgress);
    }

    private boolean prepare() {
      if (isPaused) {
        onStateChanged(STATE_PAUSED, false);
        if (!engine.provider.exists(info)) {
          engine.provider.insert(info);
        } else {
          engine.provider.update(info);
        }
        return false;
      } else {
        onStateChanged(STATE_RUNNING, false);
        onProgressChanged(info.finishedLength, info.contentLength);
        if (engine.interceptors != null) {
          for (DownloadManager.Interceptor interceptor : engine.interceptors) {
            interceptor.updateDownloadInfo(info);
          }
        }
        if (!engine.provider.exists(info)) {
          engine.provider.insert(info);
        }
        return true;
      }
    }

    @Override
    public void run() {
      if (!prepare()) return;
      long finishedLength = info.finishedLength;
      long contentLength = info.contentLength;
      HttpURLConnection connection = null;
      InputStream inputStream = null;
      RandomAccessFile randomAccessFile = null;
      try {
        connection = (HttpURLConnection) new URL(info.url).openConnection();
        connection.setAllowUserInteraction(true);
        connection.setConnectTimeout(5000);
        connection.setReadTimeout(5000);
        connection.setRequestMethod("GET");
        if (finishedLength != 0 && contentLength > 0) {
          connection.setRequestProperty("Range", "bytes=" + finishedLength + "-" + contentLength);
        } else {
          contentLength = connection.getContentLength();
        }
        int responseCode = connection.getResponseCode();
        if (contentLength > 0 && (responseCode == HttpURLConnection.HTTP_OK || responseCode == HttpURLConnection.HTTP_PARTIAL)) {
          inputStream = connection.getInputStream();
          File file = new File(info.path);
          randomAccessFile = new RandomAccessFile(file, "rw");
          randomAccessFile.seek(finishedLength);
          byte[] buffer = new byte[20480];
          int len;
          long bytesRead = finishedLength;
          while (!this.isPaused && (len = inputStream.read(buffer)) != -1) {
            randomAccessFile.write(buffer, 0, len);
            bytesRead += len;
            finishedLength = bytesRead;
            onProgressChanged(finishedLength, contentLength);
          }
          connection.disconnect();
          if (this.isPaused) {
            onStateChanged(STATE_PAUSED, true);
          } else {
            info.finishTime = System.currentTimeMillis();
            onStateChanged(STATE_FINISHED, true);
            return;
          }
        } else {
          onStateChanged(STATE_FAILED, true);
        }
      } catch (final Exception e) {
        onStateChanged(STATE_FAILED, true);
      } finally {
        try {
          if (randomAccessFile != null)
            randomAccessFile.close();
          if (inputStream != null)
            inputStream.close();
        } catch (IOException e) {
        }
        if (connection != null)
          connection.disconnect();
      }
    }
  }
  ```

# 任务调度
  ```java
  package com.grocery.download.library;

  import android.content.Context;
  import android.os.Environment;
  import android.os.Handler;
  import android.os.Looper;

  import java.io.File;
  import java.util.ArrayList;
  import java.util.HashMap;
  import java.util.List;
  import java.util.Map;
  import java.util.concurrent.LinkedBlockingQueue;
  import java.util.concurrent.ThreadPoolExecutor;
  import java.util.concurrent.TimeUnit;

  /**
   * Created by 4ndroidev on 16/10/6.
   */
  public class DownloadEngine {

      public static final String DOWNLOAD_PATH = Environment.getExternalStorageDirectory().getPath() + File.separator + "Download";
      private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
      private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
      private static final int KEEP_ALIVE = 10;

      /**
       * record all jobs those are not completed
       */
      private Map<String, DownloadJob> jobs;

      /**
       * record all download info
       */
      private Map<String, DownloadInfo> infos;

      /**
       * record all active jobs in order for notification, some jobs are created, but not running
       */
      private List<DownloadJob> activeJobs;

      private ThreadPoolExecutor singleExecutor;
      /**
       * for some server, the url of resource if temporary
       * maybe need setting interceptor to update the url
       */
      List<DownloadManager.Interceptor> interceptors;

      /**
       * download ThreadPoolExecutor
       */
      ThreadPoolExecutor executor;

      /**
       * provider for inserting, deleting, querying or updating the download info with the database
       */
      DownloadProvider provider;
      Handler handler;
      Context context;

      DownloadEngine(Context context, int maxTask) {
        this.context = context.getApplicationContext();
        jobs = new HashMap<>();
        infos = new HashMap<>();
        activeJobs = new ArrayList<>();
        interceptors = new ArrayList<>();
        handler = new Handler(Looper.getMainLooper());
        if (maxTask > CORE_POOL_SIZE) maxTask = CORE_POOL_SIZE;
        executor = new ThreadPoolExecutor(maxTask, maxTask, KEEP_ALIVE, TimeUnit.SECONDS, new LinkedBlockingQueue());
        executor.allowCoreThreadTimeOut(true);
        singleExecutor = new ThreadPoolExecutor(1, 1, KEEP_ALIVE, TimeUnit.SECONDS, new LinkedBlockingQueue());
        singleExecutor.allowCoreThreadTimeOut(true);
        provider = new DownloadProvider(this.context);
      }
      
      /**
       * prepare for the task, while creating a task, should callback the download info to the listener
       * @param task
       */
      void prepare(DownloadTask task) {
        String key = task.key;
        if (!infos.containsKey(key)) {  // do not contain this info, means that it will create a download job
          if (task.listener == null) return;
          task.listener.onStateChanged(key, DownloadState.STATE_UNKNOWN);
          return;
        }
        DownloadInfo info = infos.get(key);
        task.size = info.contentLength;
        task.createTime = info.createTime;
        if (!jobs.containsKey(key)) {  // uncompleted jobs do not contain this job, means the job had completed
          if (task.listener == null) return;
          task.listener.onStateChanged(key, info.state); // info.state == DownloadState.STATE_FINISHED
        } else {
          jobs.get(key).addListener(task.listener);
        }
      }

      /**
       * if downloadJobs contains the relative job, and the job is not running, enqueue it
       * otherwise create the job and enqueue it
       * @param task
       */
      void enqueue(DownloadTask task) {
        String key = task.key;
        if (jobs.containsKey(key)) {                   // has existed uncompleted job
          DownloadJob job = jobs.get(key);
          if (job.isRunning()) return;
          job.enqueue();
          activeJobs.add(job);
        } else {
          if (infos.containsKey(key)) return;         // means the job had completed
          DownloadInfo info = task.generateInfo();
          DownloadJob job = new DownloadJob(this, info);
          infos.put(key, info);
          jobs.put(key, job);
          job.addListener(task.listener);
          job.enqueue();
          activeJobs.add(job);
        }
      }

      /**
       * remove the downloadJob and delete the relative info
       * @param task
       */
      void remove(DownloadTask task) {
        String key = task.key;
        if (!jobs.containsKey(key)) return;
        DownloadJob job = jobs.remove(task.key);
        delete(job.getInfo());
        if (!activeJobs.contains(job)) return;
        activeJobs.remove(job);
      }

      /**
       * pause the downloadJob
       * @param task
       */
      void pause(DownloadTask task) {
        String key = task.key;
        if (!jobs.containsKey(key)) return;
        jobs.get(key).pause();
      }

      /**
       * resume the downloadJob if it has not been running
       * @param task
       */
      void resume(DownloadTask task) {
        String key = task.key;
        if (!jobs.containsKey(key)) return;
        DownloadJob job = jobs.get(key);
        if (job.isRunning()) return;
        job.resume();
        activeJobs.add(job);
      }

      /**
       * delete download info, remove file
       * @param info
       */
      void delete(final DownloadInfo info) {
        if (info == null || !infos.containsValue(info)) return;
        infos.remove(info.key);
        if (isMainThread()) {
          singleExecutor.submit(new Runnable() {
            @Override
            public void run() {
              provider.delete(info);
              File file = new File(info.path);
              if (file.exists()) file.delete();
            }
          });
        } else {
          provider.delete(info);
          File file = new File(info.path);
          if (file.exists()) file.delete();
        }
      }

      /**
       * @return whether is in main thread
       */
      private boolean isMainThread() {
        return Looper.getMainLooper() == Looper.myLooper();
      }
  }
  ```

# 使用说明
  ```java
  //创建任务
  DownloadTask task = DownloadManager.get(context)
    .download(id, url, name).listener(listener).create();

  //启动任务
  task.start();

  //暂停任务
  task.pause();

  //恢复任务
  task.resume();

  //删除任务
  task.delete();

  //暂停监听, 当activity或fragment onPause时调用
  task.pauseListener();

  //恢复监听，当activity或fragment onResume时调用
  task.resumeListener();

  //清理监听，当activity或fragment onDestroy时调用
  task.clear();
  ```