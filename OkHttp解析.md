# OkHttp 解析

## OkHttp 的基本使用

> implementation 'com.squareup.okhttp3:okhttp:3.10.0'

```text
    // 异步请求
    private void getFuncEnqueue() {
        String url = "https://www.baidu.com";
        OkHttpClient client = new OkHttpClient();

        Request request = new Request.Builder()
                .url(url)
                .get()
                .build();

        Call call = client.newCall(request);
        Callback callback = new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.i(TAG, "getFuncEnqueue onFailure");
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                ResponseBody body = response.body();
                if (body == null) {
                    Log.i(TAG, "getFuncEnqueue onResponse error");
                    return;
                }
                Log.i(TAG, "getFuncEnqueue onResponse:" + body);
            }
        };
        call.enqueue(callback);
    }

    // 同步请求
    private void getFuncExecute() {
        String url = "https://www.baidu.com";
        OkHttpClient client = new OkHttpClient();

        Request request = new Request.Builder()
                .url(url)
                .get()
                .build();

        Call call = client.newCall(request);

        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                try {
                    Response response = call.execute();
                    ResponseBody body = response.body();

                    if (body == null) {
                        Log.i(TAG, "getFuncExecute onResponse error");
                        return;
                    }
                    Log.i(TAG, "getFuncExecute onResponse:" + body);
                } catch (IOException e) {
                    Log.i(TAG, "getFuncExecute onResponse IOException");
                }

            }
        };
        new Thread(runnable).start();
    }

    // post 请求
    private void postFuncEnqueue() {
        String url = "https://www.wanandroid.com/navi/json";
        OkHttpClient client = new OkHttpClient();

        RequestBody requestBody = new FormBody.Builder()
                .add("key1","value1")
                .add("key2","value3")
                .build();

        Request request = new Request.Builder()
                .url(url)
                .post(requestBody)
                .build();

        Call call = client.newCall(request);
        Callback callback = new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.i(TAG, "postFuncEnqueue onFailure");
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                ResponseBody body = response.body();
                if (body == null) {
                    Log.i(TAG, "postFuncEnqueue onResponse error");
                    return;
                }
                Log.i(TAG, "postFuncEnqueue onResponse:" + body);
            }
        };
        call.enqueue(callback);
    }
```

## OkHttp 流程

```text
OkHttpClient#build
==>创建客户端
Request#Builder
==>创建请求
OkHttpClient.newCall(Request)
==>==>执行异步请求
Call.enqueue(callback)
==>==>执行同步请求
Call.execute()
==>分发器分发请求任务
Dispatcher
==>拦截器进行请求和返回结果
Interceptor
==>返回结果
Response
```

## RealCall

RealCall 实现了 Call 接口，且是 Call 接口的唯一实现

### Call 接口

```text
public interface Call extends Cloneable {
  // 一个 Call 对应一个 Request
  Request request();

  // 同步请求
  Response execute() throws IOException;

  // 异步请求  
  void enqueue(Callback responseCallback);

  // 取消请求
  void cancel();

  // Call 是否已经执行
  boolean isExecuted();

  // 请求是否取消
  boolean isCanceled();

  Call clone();

  // 工厂方法；被 OkHttpClient 实现；返回 Call 的唯一实现 RealCall
  interface Factory {
    Call newCall(Request request);
  }
}
```

### RealCall#request

```text
  // 私有构造函数
  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }

  // 供外部调用创建 RealCall 实例
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }

  // 返回成员变量 originalRequest
  @Override public Request request() {
    return originalRequest;
  }
```

### RealCall#execute

同步请求

```text
  @Override public Response execute() throws IOException {
    // 一个 Call 只能执行一次
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      // 通过 Dispatcher 执行 Call
      client.dispatcher().executed(this);
      // 通过 Interceptor 返回 Response
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

### RealCall#enqueue

异步请求

```text
  @Override public void enqueue(Callback responseCallback) {
    // 一个 Call 只能执行一次
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    // 通过 Dispatcher 执行 Call；异步请求使用 AsyncCall
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

### RealCall$AsyncCall

AsyncCall 是 RealCall 内部类；用于异步请求

```text
  final class AsyncCall extends NamedRunnable {
    // Callback 回调接口，需外部实现两个方法：onFailure/onResponse；以提供响应体结果给调用方
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        // 通过 Interceptor 返回 Response
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          // 通过 Callback 返回响应体结果
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          // 通过 Callback 返回响应体结果
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        // 请求完成
        client.dispatcher().finished(this);
      }
    }
  }
```

### RealCall#getResponseWithInterceptorChain

通过 Interceptor 拦截器获取响应体；
在 RealCall 的同步请求 RealCall#execute 和 异步请求 RealCall$AsyncCall#execute 中执行；
是真正执行请求工作的核心函数；

```text
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 使用者自定义的拦截器
    interceptors.addAll(client.interceptors());
    // 重试及重定向拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    // 桥接拦截器
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    // 缓存拦截器
    interceptors.add(new CacheInterceptor(client.internalCache()));
    // 链接拦截器
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      // 使用者自定义的网络拦截器
      interceptors.addAll(client.networkInterceptors());
    }
    // 请求服务器拦截器
    interceptors.add(new CallServerInterceptor(forWebSocket));

    // RealInterceptorChain 是 Interceptor.Chain 拦截器责任链条的唯一实现
    // 通过 Interceptor.Chain 将各个拦截器连接起来
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

## Dispatcher

请求任务分发器

### Dispatcher 构造函数

```text
public final class Dispatcher {
  // 异步请求允许同时存在的最大请求数
  private int maxRequests = 64;

  // 异步请求同一域名同时存在的最大请求数
  private int maxRequestsPerHost = 5;

  // 闲置任务(没有请求时可执行一些任务，由使用者设置)
  private @Nullable Runnable idleCallback;

  // 异步请求使用的线程池
  private @Nullable ExecutorService executorService;

  // 异步请求等待执行队列
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  // 异步请求正在执行队列
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  // 同步请求正在执行队列
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      // 默认线程池的实现：核心线程数为0 + SynchronousQueue 以满足最大吞吐量
      // 核心线程数为0，不会缓存线程；
      // 使用 SynchronousQueue 作为线程等待队列；SynchronousQueue 是没有容量的队列；没有容量则每次添加任务到队列都会失败；
      // 添加任务到队列失败，则没有空闲线程时，创建新线程执行任务；有空闲线程则重复利用空闲线程；
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
}
```

### Dispatcher#executed

同步请求

```text
  // 把 RealCall 添加到 runningSyncCalls 同步请求执行队列
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

### Dispatcher#enqueue

异步请求

```text
  synchronized void enqueue(AsyncCall call) {
    // 正在执行的任务小于64，且同一域名请求小于5，则添加到正在执行任务队列，并加入线程池执行；否则添加到等待执行队列
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```

### Dispatcher#finished

每次执行完一个任务后，会执行 finished 方法

```text
  // 异步请求完成
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

  // 同步请求完成
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      // 异步和同步的正在执行队列中，移除本次请求
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      // 异步请求需要重新配置队列
      if (promoteCalls) promoteCalls();
      // 异步和同步的正在执行队列请求数统计
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    // 异步和同步的正在执行队列没有请求，且闲置任务不为空时，执行闲置任务
    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
```

### Dispatcher#promoteCalls

重新配置异步任务队列；
将等待执行队列中的请求，添加到正在执行队列中；

```text
  private void promoteCalls() {
    // 正在执行队列达到阈值64，直接返回
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    // 等待执行队列中没有请求，直接返回
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    // 将等待执行队列中的请求，添加到正在执行队列中
    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();
      // 满足【同一域名同时存在的最大请求数小于5】的条件
      if (runningCallsForHost(call) < maxRequestsPerHost) {
        // 等待执行队列移除该请求，正在执行队列添加该请求，并加入线程池执行
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }
      // 正在执行队列达到阈值64，结束添加
      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```

## Interceptor

OkHttp 默认实现了5个拦截器，其都实现了拦截器接口 Interceptor

### Interceptor 接口

```text
public interface Interceptor {
  // 拦截方法；拦截器的核心功能函数；各拦截器各自实现自己的功能
  Response intercept(Chain chain) throws IOException;

  // 拦截器内部类；通过 Chain 内部类将各个拦截器关联形成责任链
  interface Chain {
    // 拦截器关联的请求
    Request request();
    // 传递到下一个拦截器方法；将拦截器链条传到下一个拦截器进行处理
    Response proceed(Request request) throws IOException;

    /**
     * Returns the connection the request will be executed on. This is only available in the chains
     * of network interceptors; for application interceptors this is always null.
     */
    @Nullable Connection connection();

    Call call();

    int connectTimeoutMillis();

    Chain withConnectTimeout(int timeout, TimeUnit unit);

    int readTimeoutMillis();

    Chain withReadTimeout(int timeout, TimeUnit unit);

    int writeTimeoutMillis();

    Chain withWriteTimeout(int timeout, TimeUnit unit);
  }
}
```

### RealInterceptorChain

拦截器链条类，是 Interceptor$Chain 唯一实现

```text
public final class RealInterceptorChain implements Interceptor.Chain {
    /** 省略部分代码，保留核心逻辑 */

  // RealCall 中传来的拦截器集合
  private final List<Interceptor> interceptors;

  @Override public Request request() {
    return request;
  }

  // 传递到责任链上的下一个拦截器
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    // 链条上的索引超出了拦截器集合大小，抛出错误
    if (index >= interceptors.size()) throw new AssertionError();

    // 通过对链条上的索引递增 [index + 1]，关联到链条的下一个拦截器
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    // 获取当前索引下的拦截器
    Interceptor interceptor = interceptors.get(index);
    // 执行当前拦截器的拦截方法，并传递给链条中下一个拦截器，同时获取下一个拦截器的响应结果
    Response response = interceptor.intercept(next);

    return response;
  }
}
```

### RetryAndFollowUpInterceptor

重试及重定向拦截器
负责判断这个请求，是否需要重试
以及根据响应码判断是否需要重定向，需要重定向则重启执行所有拦截器

#### RetryAndFollowUpInterceptor#intercept

拦截方法具体实现

```text
  @Override public Response intercept(Chain chain) throws IOException {
    /** 省略部分代码，保留核心逻辑 */

    // 重定向次数计数
    int followUpCount = 0;
    // 上次请求的响应结果
    Response priorResponse = null;

    // while (true) 循环，保持一直执行，直到 return
    while (true) {
      // 取消了请求，则抛出异常直接结束
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;

      // 通过 try/catch 中的异常类型，判断是否需要重试请求；具体判断在 recover 方法
      try {
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // 之前响应如果有结果，则附加之前的响应结果
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      // 判断是否需要重定向请求
      Request followUp = followUpRequest(response, streamAllocation.route());

      // 如果 followUp 返回 null，则不需要重定向，返回本次响应结果
      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      /**
       * How many redirects and auth challenges should we attempt? Chrome follows 21 redirects; Firefox,
       * curl, and wget follow 20; Safari follows 16; and HTTP/1.0 recommends 5.
       */
      // private static final int MAX_FOLLOW_UPS = 20;
      // 允许重定向的最大次数为 20；参考了各浏览器的实践
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      // 本次重定向请求赋值给新的请求
      request = followUp;
      // 本次响应结果赋值给 priorResponse，以保存本次响应结果
      priorResponse = response;
    }
  }
```

#### RetryAndFollowUpInterceptor#recover

请求重试判断

```text
  private boolean recover(IOException e, StreamAllocation streamAllocation,
      boolean requestSendStarted, Request userRequest) {
    streamAllocation.streamFailed(e);

    // 如果 OkHttpClient 设置了不允许重试，则不重试；该参数默认允许重试
    if (!client.retryOnConnectionFailure()) return false;

    // 请求体属于 UnrepeatableRequestBody，则不重试
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

    // 如果是不允许重试的异常，则不重试
    if (!isRecoverable(e, requestSendStarted)) return false;

    // 没有可以用来连接的路由路线，则不重试
    if (!streamAllocation.hasMoreRoutes()) return false;

    // 其余情况，允许重试
    return true;
  }
```

```text
  private boolean isRecoverable(IOException e, boolean requestSendStarted) {
    // 协议异常，不允许重试
    if (e instanceof ProtocolException) {
      return false;
    }

    // Socket 超时异常，允许重试
    if (e instanceof InterruptedIOException) {
      return e instanceof SocketTimeoutException && !requestSendStarted;
    }

    // SSL 握手异常，且是证书异常，不允许重试
    if (e instanceof SSLHandshakeException) {
      // If the problem was a CertificateException from the X509TrustManager,
      // do not retry.
      if (e.getCause() instanceof CertificateException) {
        return false;
      }
    }
    // SSL 握手未授权异常，不允许重试
    if (e instanceof SSLPeerUnverifiedException) {
      // e.g. a certificate pinning error.
      return false;
    }

    // 其余情况，允许重试
    return true;
  }
```

#### RetryAndFollowUpInterceptor#followUpRequest

请求重定向判断

```text
  private Request followUpRequest(Response userResponse, Route route) throws IOException {
    /** 省略部分代码，保留核心逻辑 */

    // 获取请求HTTP响应码，和HTTP请求方法
    if (userResponse == null) throw new IllegalStateException();
    int responseCode = userResponse.code();
    final String method = userResponse.request().method();

    // HTTP响应码分类：
    // 100类，服务器收到请求，客户端继续发送请求；
    // 200类，请求发送并完成处理；
    // 300类，需要重定向，指向新地址进行进一步操作；
    // 400类，客户端异常；
    // 500类，服务端异常；
    switch (responseCode) {
      // 407 响应码
      case HTTP_PROXY_AUTH:
        return client.proxyAuthenticator().authenticate(route, userResponse);
      // 401 响应码
      case HTTP_UNAUTHORIZED:
        return client.authenticator().authenticate(route, userResponse);

      // 307/308 响应码
      case HTTP_PERM_REDIRECT:
      case HTTP_TEMP_REDIRECT:
        if (!method.equals("GET") && !method.equals("HEAD")) {
          return null;
        }
      // 300-303 响应码
      case HTTP_MULT_CHOICE:
      case HTTP_MOVED_PERM:
      case HTTP_MOVED_TEMP:
      case HTTP_SEE_OTHER:
        return requestBuilder.url(url).build();

      // 408 响应码
      case HTTP_CLIENT_TIMEOUT:
        return userResponse.request();

      // 503 响应码
      case HTTP_UNAVAILABLE:
        if (retryAfter(userResponse, Integer.MAX_VALUE) == 0) {
          return userResponse.request();
        }
        return null;

      // 默认返回 null，则不需要重定向
      default:
        return null;
    }
  }
```

### BridgeInterceptor

桥接拦截器
负责拼接HTTP协议请求头，以及判断是否需要gzip压缩和gzip解析，以及Cookie处理

#### BridgeInterceptor#intercept

```text
  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    // 拼接 "Content-Type"
    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    // 拼接 "Host"
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    // 拼接 "Connection"
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // 拼接 "Accept-Encoding"；并判断是否使用 gzip 压缩
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    // 拼接 "Cookie"
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    // 拼接 "User-Agent"
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    // 判断是否使用 gzip 解压
    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }
```

### CacheInterceptor

缓存拦截器
负责判断是否使用缓存，如果使用缓存，则使用缓存结果作为请求响应结果
OkHttp 默认只支持缓存 GET 请求

#### CacheInterceptor#intercept

```text
  @Override public Response intercept(Chain chain) throws IOException {
    // 判断是否需要缓存候选
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();

    // 缓存策略类：CacheStrategy
    // CacheStrategy.networkRequest 为 null，则此次请求不需要使用网络
    // CacheStrategy.cacheResponse 为 null，则此次请求不需要使用缓存
    // networkRequest == null/cacheResponse != null;不需要使用网络，需要使用缓存；直接使用缓存
    // networkRequest != null/cacheResponse == null;需要使用网络，不需要使用缓存；直接向服务器发起请求
    // networkRequest == null/cacheResponse == null;不需要使用网络，不需要使用缓存；返回504错误码
    // networkRequest != null/cacheResponse != null;需要使用网络，需要使用缓存；发起请求，若响应为304，则更新缓存并返回结果
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    // 缓存候选不为null，但 cacheResponse 为 null，则不需要使用缓存；缓存候选不需要使用，关闭缓存候选
    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // 不需要使用网络，不需要使用缓存；返回504错误码
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // 不需要使用网络，需要使用缓存；直接使用缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // 需要使用网络，需要使用缓存；发起请求，若响应为304，则更新缓存并返回结果
    if (cacheResponse != null) {
      // 若响应为304，则更新缓存并返回结果
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        // 若响应不为304，则清空缓存内容
        closeQuietly(cacheResponse.body());
      }
    }

    // 其余情况，需要使用网络，不需要使用缓存；直接向服务器发起请求
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

#### CacheStrategy

缓存策略类

```text
    public CacheStrategy get() {
      // 缓存策略类：CacheStrategy
      // CacheStrategy.networkRequest 为 null，则此次请求不需要使用网络
      // CacheStrategy.cacheResponse 为 null，则此次请求不需要使用缓存
      // networkRequest == null/cacheResponse != null;不需要使用网络，需要使用缓存；直接使用缓存
      // networkRequest != null/cacheResponse == null;需要使用网络，不需要使用缓存；直接向服务器发起请求
      // networkRequest == null/cacheResponse == null;不需要使用网络，不需要使用缓存；返回504错误码
      // networkRequest != null/cacheResponse != null;需要使用网络，需要使用缓存；发起请求，若响应为304，则更新缓存并返回结果
 
      CacheStrategy candidate = getCandidate();
      // networkRequest != null 请求需要使用网络，但请求又设置了只是用缓存；
      // 则返回 networkRequest == null/cacheResponse == null
      if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
        // We're forbidden from using the network and the cache is insufficient.
        return new CacheStrategy(null, null);
      }

      return candidate;
    }

    /** Returns a strategy to use assuming the request can use the network. */
    private CacheStrategy getCandidate() {
      // cacheResponse == null，不需要使用缓存；
      // 返回 networkRequest != null/cacheResponse == null
      if (cacheResponse == null) {
        return new CacheStrategy(request, null);
      }

      // 本次请求是HTTPS，但是缓存中没有对应的握手信息，那么缓存无效
      // 返回 networkRequest != null/cacheResponse == null
      if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
      }

      // 依据响应码判断缓存是否可用；不可用则直接返回；可用则再往下继续判断缓存有效性
      // 返回 networkRequest != null/cacheResponse == null
      if (!isCacheable(cacheResponse, request)) {
        return new CacheStrategy(request, null);
      }

      // 对发起的请求进行设置判断，如果不需要使用缓存则直接返回
      // 返回 networkRequest != null/cacheResponse == null
      CacheControl requestCaching = request.cacheControl();
      if (requestCaching.noCache() || hasConditions(request)) {
        return new CacheStrategy(request, null);
      }

      // 如果缓存的响应中包含 Cache-Control: immutable，则发起请求的响应内容保持不变，直接使用缓存
      // 返回 networkRequest == null/cacheResponse != null
      CacheControl responseCaching = cacheResponse.cacheControl();
      if (responseCaching.immutable()) {
        return new CacheStrategy(null, cacheResponse);
      }

      long ageMillis = cacheResponseAge();
      long freshMillis = computeFreshnessLifetime();

      if (requestCaching.maxAgeSeconds() != -1) {
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
      }

      long minFreshMillis = 0;
      if (requestCaching.minFreshSeconds() != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
      }

      long maxStaleMillis = 0;
      if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
      }

      // 判断缓存的有效时间，缓存有效期内使用缓存
      // 返回 networkRequest == null/cacheResponse != null
      if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        return new CacheStrategy(null, builder.build());
      }

      // 从服务器获取 conditionValue 数据，拼接 conditionName/conditionValue 参数
      String conditionName;
      String conditionValue;
      if (etag != null) {
        conditionName = "If-None-Match";
        conditionValue = etag;
      } else if (lastModified != null) {
        conditionName = "If-Modified-Since";
        conditionValue = lastModifiedString;
      } else if (servedDate != null) {
        conditionName = "If-Modified-Since";
        conditionValue = servedDateString;
      } else {
        // 如果服务器没有有效参数返回，则没有可以判断的条件，直接发起新请求
        // 返回 networkRequest != null/cacheResponse == null
        return new CacheStrategy(request, null); // No condition! Make a regular request.
      }

      // 如果服务器包含有效参数返回，拼接参数，发起请求，同时带上缓存
      // 则返回 networkRequest != null/cacheResponse != null
      Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
      Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

      Request conditionalRequest = request.newBuilder()
          .headers(conditionalRequestHeaders.build())
          .build();
      return new CacheStrategy(conditionalRequest, cacheResponse);
    }
```

#### Cache 缓存类

只支持缓存 GET 请求

```text
  @Nullable CacheRequest put(Response response) {
    /** 省略部分代码，保留核心逻辑 */

    String requestMethod = response.request().method();
    // 对请求方法做校验，清理 POST/PUT/DELETE 等等方法
    if (HttpMethod.invalidatesCache(response.request().method())) {
      try {
        remove(response.request());
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
      return null;
    }
    // 不对非 GET 请求做缓存；技术上允许缓存HEAD请求和一些POST请求，但这样做的复杂性很高，收益很低
    // 返回 null；则不支持缓存
    if (!requestMethod.equals("GET")) {
      // Don't cache non-GET responses. We're technically allowed to cache
      // HEAD requests and some POST requests, but the complexity of doing
      // so is high and the benefit is low.
      return null;
    }
  }
```

### ConnectInterceptor

连接拦截器
负责创建 Connection 连接；主要目标是在连接池中找到可复用连接，或者创建新连接

#### ConnectInterceptor#intercept

```text
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    // 核心逻辑都封装在 StreamAllocation.newStream 方法
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    // 经过 newStream 方法赋值后；返回 StreamAllocation 的成员变量 connection 给 ConnectInterceptor
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

#### StreamAllocation#newStream

```text
  public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    /** 省略部分代码，保留核心逻辑 */

    // 核心功能在 findHealthyConnection 中实现
    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }

  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      // 核心功能在 findConnection 中实现
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```

#### StreamAllocation#findConnection

```text
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    /** 省略部分代码，保留核心逻辑 */

    synchronized (connectionPool) {
      if (result == null) {
        // 尝试从连接池中获取连接；最终会调用 ConnectionPool.get
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    if (result != null) {
      // 能从池中获取到连接，则返回这个连接
      return result;
    }

    synchronized (connectionPool) {
      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }
        route = selectedRoute;
        refusedStreamCount = 0;
        // 如果连接池中没有可复用的连接，则创建一个新连接
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    synchronized (connectionPool) {
      // 创建的新连接保存到连接池；最终会调用 ConnectionPool.put
      Internal.instance.put(connectionPool, result);
    }
    // 最终返回这个新连接
    return result;
  }
```

#### ConnectionPool#get/put

```text
  @Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      // 判断连接池中的连接是否符合复用条件
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
  }
```

```text
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      // 开启连接池的清理连接任务
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    // 把连接放入连接池
    connections.add(connection);
  }
```

#### RealConnection#isEligible

判断连接是否符合复用条件

```text
  public boolean isEligible(Address address, @Nullable Route route) {
    // 连接到达最大并发流，或者该连接不允许建立新的流；则不允许复用
    if (allocations.size() >= allocationLimit || noNewStreams) return false;

    // 不考虑Host情况下，如果DNS/地址/端口/协议等有不同，则不允许复用
    if (!Internal.instance.equalsNonHost(this.route.address(), address)) return false;

    // 满足上满地址相同的情况下，如果Host相同，则连接可以复用
    if (address.url().host().equals(this.route().address().url().host())) {
      return true; // This connection is a perfect match.
    }

    // 其余情况，在HTTP/2的某些场景下仍可以复用
    // 1. 必须是 HTTP/2 请求
    if (http2Connection == null) return false;

    // 2. 对路由/代理/地址等判断
    if (route == null) return false;
    if (route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (this.route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (!this.route.socketAddress().equals(route.socketAddress())) return false;

    // 3. 服务器证书的判断
    if (route.address().hostnameVerifier() != OkHostnameVerifier.INSTANCE) return false;
    if (!supportsUrl(address.url())) return false;

    // 4. SSL认证信息的判断
    try {
      address.certificatePinner().check(address.url().host(), handshake().peerCertificates());
    } catch (SSLPeerUnverifiedException e) {
      return false;
    }

    // 综合判断：如果在连接池中找到个连接参数一致并且未被关闭没被占用的连接，则可以复用
    return true; // The caller's address can be carried by this connection.
  }
```

### CallServerInterceptor

请求服务器拦截器
拦截器的主要目标是完成HTTP协议报文的封装与解析
利用 HttpCodec 发出请求到服务器并且解析生成 Response

#### CallServerInterceptor#intercept

```text
  @Override public Response intercept(Chain chain) throws IOException {
    /** 省略部分代码，保留核心逻辑 */

    realChain.eventListener().requestHeadersStart(realChain.call());
    // 写入请求头信息，开始拼接请求信息
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // 上传大容量请求体时，可能需要先确认服务器时是否接受请求体；
      // 如果请求头包含"Expect: 100-continue"，表明需要先确认服务器时是否接受请求体；
      // 如果服务器返回100，则服务器同意接受请求体，客户端继续发送请求体
      // 如果服务器返回其他，则服务器不同意接受请求体，直接把本次响应结果返回
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        // 请求头包含"Expect: 100-continue"信息，刷新请求信息
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // 如果是"Expect: 100-continue"需要请求，则缓存请求体信息
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        streamAllocation.noNewStreams();
      }
    }

    // 本次"Expect: 100-continue"请求数据拼接结束
    httpCodec.finishRequest();

    // responseBuilder 为 null；表明本次请求不带"Expect: 100-continue"
    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    // 开启请求并返回响应结果
    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // 如果"Expect: 100-continue"请求服务器返回100，则服务器同意接受请求体，客户端发送真正请求体的请求
      responseBuilder = httpCodec.readResponseHeaders(false);

      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }

    return response;
  }
```
