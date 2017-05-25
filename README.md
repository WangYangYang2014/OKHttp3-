# OKHttp3源码学习

> 参考资源

* [官网](http://square.github.io/okhttp/)
* [国内博客](http://www.jianshu.com/p/aad5aacd79bf)
* [GitHub官网](https://github.com/square/okhttp)

鉴于一些关于OKHttp3源码的解析文档过于碎片化，本文系统的，由浅入深得，按照网络请求发起的流程顺序来讲解OkHttp3的源码。在自己学习的同时，给大家分享一些经验。

	
## 主要架构和流程


###  OKHttpClient、Call
OKHttp3在项目中发起网络请求的API如下：

	okHttpClient.newCall(request).execute();

OKHttpClient类：

* OKHttpClient 里面组合了很多的类对象。其实是将OKHttp的很多功能模块，全部包装进这个类中，让这个类单独提供对外的API，这种设计叫做[外观模式](https://zh.wikipedia.org/wiki/%E5%A4%96%E8%A7%80%E6%A8%A1%E5%BC%8F)。

* 由于内部功能模块太多，使用了[Builder模式(生成器模式)](https://zh.wikipedia.org/wiki/%E7%94%9F%E6%88%90%E5%99%A8%E6%A8%A1%E5%BC%8F)来构造。

它的方法只有一个：newCall.返回一个Call对象(一个准备好了的可以执行和取消的请求)。

Call接口：

	public interface Call {
	  
	  Request request();
	 
	  //同步的方法，直接返回Response
	  Response execute() throws IOException;
	  
	  //异步的，传入回调CallBack即可(接口，提供onFailure和onResponse方法)
	  void enqueue(Callback responseCallback);
	  
	  void cancel();
	
	  boolean isExecuted();
	
	  boolean isCanceled();
	
	  interface Factory {
	    Call newCall(Request request);
	  }
	}


Call接口提供了内部接口Factory(用于将对象的创建延迟到该工厂类的子类中进行，从而实现动态的配置，[工厂方法模式](https://zh.wikipedia.org/wiki/%E5%B7%A5%E5%8E%82%E6%96%B9%E6%B3%95))。

实际的源码中，OKHttpClient实现了Call.Factory接口，返回了一个RealCall对象。

	@Override 
	public Call newCall(Request request) {
	    return new RealCall(this, request);
	}

RealCall里面的两个关键方法是：execute 和 enqueue。分别用于同步和异步得执行网络请求。后面会详细介绍。


### 请求Request、返回数据Response
###### Request：

		public final class Request {
		  //url字符串和端口号信息，默认端口号：http为80，https为443.其他自定义信息
		  private final HttpUrl url;
		  
		  //"get","post","head","delete","put"....
		  private final String method;
		  
		  //包含了请求的头部信息，name和value对。最后的形势为：$name1+":"+$value1+"\n"+ $name2+":"+$value2+$name3+":"+$value3...
		  private final Headers headers;
		  
		  //请求的数据内容
		  private final RequestBody body;
		  
		  //请求的附加字段。对资源文件的一种摘要。保存在头部信息中：ETag: "5694c7ef-24dc"。客户端可以在二次请求的时候，在requst的头部添加缓存的tag信息（如If-None-Match:"5694c7ef-24dc"），服务端用改信息来判断数据是否发生变化。
		  private final Object tag;
		  
		  //各种附值函数和Builder类
		  ...
		 	
		 ｝

其中内部类RequestBody: 请求的数据。抽象类:

		public abstract class RequestBody {
		 
		  ...
		  
		  //返回内容类型
		  public abstract MediaType contentType();
		
		  //返回内容长度
		  public long contentLength() throws IOException {
		    return -1;
		  }
		
		  //如何写入缓冲区。BufferedSink是第三方库okio对输入输出API的一个封装，不做详解。
		  public abstract void writeTo(BufferedSink sink) throws IOException;
	
		}
	
> OKHttp3中给出了两个requestBody的实现FormBody 和 MultipartBody，分别对应了两种不同的[MIME类型](https://zh.wikipedia.org/wiki/%E4%BA%92%E8%81%94%E7%BD%91%E5%AA%92%E4%BD%93%E7%B1%BB%E5%9E%8B)："application/x-www-form-urlencoded"和"multipart/"+xxx.作为的默认实现。


###### Response：

		public final class Response implements Closeable {
		  //网络请求的信息
		  private final Request request;
		  
		  //网路协议，OkHttp3支持"http/1.0","http/1.1","h2"和"spdy/3.1"
		  private final Protocol protocol;
		  
		  //返回状态码，包括404(Not found),200(OK),504(Gateway timeout)...
		  private final int code;
		  
		  //状态信息，与状态码对应
		  private final String message;
		  
		  //TLS(传输层安全协议)的握手信息（包含协议版本，密码套件(https://en.wikipedia.org/wiki/Cipher_suite)，证书列表
		  private final Handshake handshake;
		  
		  //相应的头信息，格式与请求的头信息相同。
		  private final Headers headers;
		  
		  //数据内容在ResponseBody中
		  private final ResponseBody body;
		  
		  //网络返回的原声数据(如果未使用网络，则为null)
		  private final Response networkResponse;
		  
		  //从cache中读取的网络原生数据
		  private final Response cacheResponse;
		  
		  //网络重定向后的，存储的上一次网络请求返回的数据。
		  private final Response priorResponse;
		  
		  //发起请求的时间轴
		  private final long sentRequestAtMillis;
		  
		  //收到返回数据时的时间轴
		  private final long receivedResponseAtMillis;
		
		  //缓存控制指令，由服务端返回数据的中的Header信息指定，或者客户端发器请求的Header信息指定。key："Cache-Control"
		  //详见<a href="http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9">RFC 2616,14.9</a>
		  private volatile CacheControl cacheControl; // Lazily initialized.
			
		  //各种附值函数和Builder类型		  ...
		}

> Note：所有网络请求的头部信息的key，不是随便写的。都是RFC协议规定的。request的header与response的header的标准都不同。具体的见 [List of HTTP header fields](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields)。OKHttp的封装类Request和Response为了应用程序编程方便，会把一些常用的Header信息专门提取出来，作为局部变量。比如contentType，contentLength，code,message,cacheControl,tag...它们其实都是以name-value对的形势，存储在网络请求的头部信息中。

我们使用了[retrofit2](https://github.com/square/retrofit)，它提供了接口converter将自定义的数据对象（各类自定义的request和response）和OKHttp3中网络请求的数据类型（ReqeustBody和ResponseBody）进行转换。
而converterFactory是converter的工厂模式，用来构建各种不同类型的converter。

故而可以添加converterFactory由retrofit完成requestBody和responseBody的构造。
	
这里对[retrofit2](https://github.com/square/retrofit)不展开讨论，后续会出新的文章来详细讨论。仅仅介绍一下converterFacotry，以及它是如何构建OkHttp3中的RequestBody和ResponseBody的。

* > Note: retrofit2中的Response与okhttp3中的response不同，前者是包含了后者。既retrofit2中的response是一层封装，内部才是真正的okhttp3种的response。

我们项目中的一个converterFacotry代码如下：
	
		public class RsaGsonConverterFactory extends Converter.Factory {
	   
	   	//省略部分代码
	   	...
	   	
	    private final Gson gson;
	
	    private RsaGsonConverterFactory(Gson gson) {
	        if (gson == null) throw new NullPointerException("gson == null");
	        this.gson = gson;
	    } 
		//将返回的response的Type，注释，和retrofit的传进来，返回response的转换器。Gson只需要type就可以将responseBody转换为需要的类型。
	    @Override
	    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
	        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
	        return new RsaGsonResponseBodyConverter<>(gson, adapter);
	    }
		//将request的参数类型，参数注释，方法注释和retrofit传进来，返回request的转换器。Gson只需要type就可以将request对象转换为OKHttp3的reqeustBody类型。
	    @Override
	    public Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
	        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
	        return new RsaGsonRequestBodyConverter<>(gson, adapter);
	    }
		}
		
该Factory([工厂方法模式](https://zh.wikipedia.org/wiki/%E5%B7%A5%E5%8E%82%E6%96%B9%E6%B3%95)，用于动态的创建对象)主要是用来生产response的converter和request的converter。显然我们使用了Gson作为数据转换的桥梁。分别对应如下两个类：

* response的converter(之所以命名为Rsa，是做了一层加解密)：

		public class RsaGsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
		    private final Gson gson;
		    private final TypeAdapter<T> adapter;
		
		    RsaGsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
		        this.gson = gson;
		        this.adapter = adapter;
		    }
		
		    @Override public T convert(ResponseBody value) throws IOException {
		        JsonReader jsonReader = gson.newJsonReader(value.charStream());
		        try {
		            return adapter.read(jsonReader);
		        } finally {
		            value.close();
		        }
		    }
		}
> 直接将value中的值封装为JsonReader供Gson的TypeAdapter读取，获取转换后的对象。
* request的converter：
	
		final class RsaGsonRequestBodyConverter<T> implements Converter<T, RequestBody> {
		    private static final MediaType MEDIA_TYPE = MediaType.parse("application/json; charset=UTF-8");
		    private static final Charset UTF_8 = Charset.forName("UTF-8");
		
		    private final Gson gson;
		    private final TypeAdapter<T> adapter;
		
		    RsaGsonRequestBodyConverter(Gson gson, TypeAdapter<T> adapter) {
		        this.gson = gson;
		        this.adapter = adapter;
		    }
		
		    @Override public RequestBody convert(T value) throws IOException {
		
		        Buffer buffer = new Buffer();
		        Writer writer = new OutputStreamWriter(buffer.outputStream(), UTF_8);
		        JsonWriter jsonWriter = gson.newJsonWriter(writer);
		
		        adapter.write(jsonWriter, value);
		        jsonWriter.close();
				//如果是RsaReq的子类，则进行一层加密。
		        if(value instanceof RsaReq){
		           //加密过程
		        }
		        //不需要加密，则直接读取byte值，用来创建requestBody
		        else {
		        	//这个构造方法是okhttp专门为okio服务的构造方法。
		            return RequestBody.create(MEDIA_TYPE, buffer.readByteString());
		        }
		    }
		}	


上面的流操作使用的是第三方库[okio](https://github.com/square/okio)。可以看到，[retrofit](https://github.com/square/retrofit)，[okhttp](https://github.com/square/okhttp),[okio](https://github.com/square/okio)这三个库是完全相互兼容并互相提供了专有的API。


### 请求的分发和线程池技术
OKHttpClient类中有个成员变量dispatcher负责请求的分发。既在真正的请求RealCall的execute方法中，使用dispatcher来执行任务：

* RealCall的execute方法：

		@Override 
		public Response execute() throws IOException {
	    synchronized (this) {
	      if (executed) throw new IllegalStateException("Already Executed");
	      executed = true;
	    }
	    try {
	      //使用dispatcher 来分发任务
	      client.dispatcher().executed(this);
	      Response result = getResponseWithInterceptorChain();
	      if (result == null) throw new IOException("Canceled");
	      return result;
	    } finally {
	      client.dispatcher().finished(this);
	    }
		}

* RealCall的enqueue方法：

		@Override public void enqueue(Callback responseCallback) {
		    synchronized (this) {
		      if (executed) throw new IllegalStateException("Already Executed");
		      executed = true;
		    }
		    //使用dispatcher来将人物加入队列
		    client.dispatcher().enqueue(new AsyncCall(responseCallback));
		  }

OKHttp3中分发器只有一个类 ——Dispathcer.

![image](http://of8cu1h2w.bkt.clouddn.com/Dispathcher.png =405x185)

(1) 其中包含了线程池executorService：

	 public synchronized ExecutorService executorService() {
	    if (executorService == null) {
	      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS, new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
	    }
	    return executorService;
	  }

参数：

* 0:核心线程数量。保持在线程池中的线程数量(即使已经空闲)，为0代表线程空闲后不会保留，等待一段时间后停止。
* Integer.MAX_VALUE: 线程池可容纳线程数量。
* 60，TimeUnit.SECONDS: 当线程池中的线程数大于核心线程数时，空闲的线程会等待60s后才会终止。如果小于，则会立刻停止。
* new SynchronousQueue<Runnable>():线程的等待队列。同步队列，按序排队，先来先服务。
	Util.threadFactory("OkHttp Dispatcher", false)： 线程工厂，直接创建一个名为 “OkHttp Dispathcer”的非守护线程。
	

(2) 执行同步的Call：直接加入runningSyncCalls队列中，实际上并没有执行该Call，交给外部执行。

	  synchronized void executed(RealCall call) {
	    runningSyncCalls.add(call);
	  }

(3) 将Call加入队列：如果当前正在执行的call数量大于maxRequests,64,或者该call的Host上的call超过maxRequestsPerHost，5，则加入readyAsyncCalls排队等待。否则加入runningAsyncCalls，并执行。

	  synchronized void enqueue(AsyncCall call) {
	    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
	      runningAsyncCalls.add(call);
	      executorService().execute(call);
	    } else {
	      readyAsyncCalls.add(call);
	    }
	  }

(4) 从ready到running的轮转，在每个call 结束的时候调用finished，并：

		private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
		    int runningCallsCount;
		    Runnable idleCallback;
		    synchronized (this) {
		      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
		      //每次remove完后，执行promoteCalls来轮转。
		      if (promoteCalls) promoteCalls();
		      runningCallsCount = runningCallsCount();
		      idleCallback = this.idleCallback;
		    }
			//线程池为空时，执行回调
		    if (runningCallsCount == 0 && idleCallback != null) {
		      idleCallback.run();
		    }
		  }
		 
(5) 线程轮转：遍历readyAsyncCalls，将其中的calls添加到runningAysncCalls，直到后者满。

		private void promoteCalls() {
		    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
		    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.
		
		    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
		      AsyncCall call = i.next();
				
		      if (runningCallsForHost(call) < maxRequestsPerHost) {		        i.remove();
		        runningAsyncCalls.add(call);
		        executorService().execute(call);
		      }
		
		      if (runningAsyncCalls.size() >= maxRequests) return; 
		    }
		  }  
			
			
			
### 执行请求

同步的请求RealCall 实现了Call接口:
	可以execute,enqueue和cancle。
异步的请求AsyncCall（RealCall的内部类）实现了Runnable接口:
	只能run(调用了自定义函数execute).
	
> execute 对比：

* RealCall：
	
	  @Override public Response execute() throws IOException {
	    synchronized (this) {
	      if (executed) throw new IllegalStateException("Already Executed");
	      executed = true;
	    }
	    try {
	      //分发。实际上只是假如了队列，并没有执行
	      client.dispatcher().executed(this);
	      //实际上的执行。
	      Response result = getResponseWithInterceptorChain();
	      //返回结果
	      if (result == null) throw new IOException("Canceled");
	      return result;
	    } finally {
	      //执行完毕，finish
	      client.dispatcher().finished(this);
	    }
	  }
* AsyncCall:


		@Override protected void execute() {
		      boolean signalledCallback = false;
		      try {
		      	//实际执行。
		        Response response = getResponseWithInterceptorChain();
		        //执行回调
		        if (retryAndFollowUpInterceptor.isCanceled()) {
		          signalledCallback = true;
		          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
		        } else {
		          signalledCallback = true;
		          responseCallback.onResponse(RealCall.this, response);
		        }
		      } catch (IOException e) {
		        if (signalledCallback) {
		          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
		        } else {
		          responseCallback.onFailure(RealCall.this, e);
		        }
		      } finally {
		      	//执行完毕，finish
		        client.dispatcher().finished(this);
		      }
		    }
实际上的执行函数都是getResponseWithInterceptorChain()：


		private Response getResponseWithInterceptorChain() throws IOException {
		    //创建一个拦截器列表
		    List<Interceptor> interceptors = new ArrayList<>();
		    //优先处理自定义拦截器
		    interceptors.addAll(client.interceptors());
		    //失败重连拦截器
		    interceptors.add(retryAndFollowUpInterceptor);
		    //接口桥接拦截器(同时处理cookie逻辑)
		    interceptors.add(new BridgeInterceptor(client.cookieJar()));
		    //缓存拦截器
		    interceptors.add(new CacheInterceptor(client.internalCache()));
		    //分配连接拦截器
		    interceptors.add(new ConnectInterceptor(client));
		    //web的socket连接的网络配置拦截器
		    if (!retryAndFollowUpInterceptor.isForWebSocket()) {
		      interceptors.addAll(client.networkInterceptors());
		    }
		    //最后是连接服务器发起真正的网络请求的拦截器
		    interceptors.add(new CallServerInterceptor(
		        retryAndFollowUpInterceptor.isForWebSocket())); 
		    Interceptor.Chain chain = new RealInterceptorChain(
		        interceptors, null, null, null, 0, originalRequest);
		    //流式执行并返回response
		    return chain.proceed(originalRequest);
		  }
> 这里的拦截器的作用：将一个流式工作分解为可配置的分段流程，既实现了逻辑解耦，又增强了灵活性，使得该流程清晰，可配置。


### 各个拦截器(Interceptor)
这里的拦截器有点像安卓里面的触控反馈的Interceptor。既一个网络请求，按一定的顺序，经由多个拦截器进行处理，该拦截器可以决定自己处理并且返回我的结果，也可以选择向下继续传递，让后面的拦截器处理返回它的结果。这个设计模式叫做[责任链模式](https://zh.wikipedia.org/wiki/%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F)。

>与Android中的触控反馈interceptor的设计略有不同的是，后者通过返回true 或者 false 来决定是否已经拦截。而OkHttp这里的拦截器通过函数调用的方式，讲参数传递给后面的拦截器的方式进行传递。这样做的好处是拦截器的逻辑比较灵活，可以在后面的拦截器处理完并返回结果后仍然执行自己的逻辑；缺点是逻辑没有前者清晰。

拦截器接口的源码：

	public interface Interceptor {
	 Response intercept(Chain chain) throws IOException;
	
	 interface Chain {
	   Request request();
	
	   Response proceed(Request request) throws IOException;
	
	   Connection connection();
	 }
	}

其中的Chain是用来传递的链。这里的传递逻辑伪代码如下：
代码的最外层逻辑

	 Request request = new Request(){};
	 
	 
	 Arrlist<Interceptor> incpts = new Arrlist();
	 Interceptor icpt0 = new Interceptor(){ XXX };
	 Interceptor icpt1 = new Interceptor(){ XXX };
	 Interceptor icpt2 = new Interceptor(){ XXX };
	 ...
	 incpts.add(icpt0);
	 incpts.add(icpt1);
	 incpts.add(icpt2);
	 
	 Interceptor.Chain chain  = new MyChain(incpts);
	 chain.proceed(request);
 
 封装的Chain的内部逻辑	
 
	 public class MyChain implement Interceptor.Chain{
	 	Arrlist<Interceptor> incpts;
	 	int index = 0;
	 	
	 	public MyChain(Arrlist<Interceptor> incpts){
	 		this(incpts, 0);
	 	}
	 	
	 	public MyChain(Arrlist<Interceptor> incpts, int index){
	 		this.incpts = incpts;
	 		this.index =index;
	 	}
	 	
	 	public void setInterceptors(Arrlist<Interceptor> incpts ){
	 		this.incpts = incpts;
	 	}
	 
	 	@override
	 	Response proceed(Request request) throws IOException{
	 			Response response = null;
	 			...
	 			//取出第一个interceptor来处理
	 			Interceptor incpt = incpts.get(index);
	 			//生成下一个Chain，index标识当前Interceptor的位置。
	 			Interceptor.Chain nextChain = new MyChain(incpts,index+1);
	 			response =  incpt.intercept(nextChain);
	 			...
	 			return response;
	 	}
	  } 
	 
各个Interceptor类中的实现：

	public class MyInterceptor implement Intercetpor{
		@Override 
		public Response intercept(Chain chain) throws IOException {
			Request request = chain.request();
			//前置拦截逻辑
			...
			Response response = chain.proceed(request);//传递Interceptor
			//后置拦截逻辑
			...
			return response;
		}
	}
	
在这个链中，最后的一个Interceptor一般用作生成最后的Response操作，它不会再继续传递给下一个。

* ####失败重连以及重定向的拦截器：RetryAndFollowUpInterceptor
	失败重连拦截器核心源码：
	一个循环来不停的获取response。每循环一次都会获取下一个request，如果没有，则返回response，退出循环。而获取下一个request的逻辑，是根据上一个response返回的状态码，分别作处理。
	
intercept方法源码：

		@Override public Response intercept(Chain chain) throws IOException {
		    Request request = chain.request();
		
			...
		
		    int followUpCount = 0;
		    Response priorResponse = null;
		    
		    //循环入口
		    while (true) {
		      ...
		
		      Response response = null;
		      try {
		      	//一次请求处理，获得结果
		        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
		      } catch (RouteException e) {
		        ...
		        continue;
		      } catch (IOException e) {
		        ...
		        continue;
		      } finally {
		        ...
		      }
		
		      // 如果前一次请求结果不为空，讲它添加到新的请求结果中。通常第一次请求一定是异常请求结果，一定没有body。
		      if (priorResponse != null) {
		        response = response.newBuilder()
		            .priorResponse(priorResponse.newBuilder()
		                .body(null)
		                .build())
		            .build();
		      }
			  //获取后续的请求，比如验证，重定向，失败重连...
		      Request followUp = followUpRequest(response);
		
		      if (followUp == null) {
		        ...
		        //如果没有后续的请求了，直接返回请求结果
		        return response;
		      }
			   ...		
			  //
		      request = followUp;
		      priorResponse = response;
		    }
		  }

获取后续的的请求，比如验证，重定向，失败重连

	private Request followUpRequest(Response userResponse) throws IOException {
	    if (userResponse == null) throw new IllegalStateException();
	    Connection connection = streamAllocation.connection();
	    Route route = connection != null
	        ? connection.route()
	        : null;
	    int responseCode = userResponse.code();
	
	    final String method = userResponse.request().method();
	    switch (responseCode) {
	      case HTTP_PROXY_AUTH:
	        Proxy selectedProxy = route != null
	            ? route.proxy()
	            : client.proxy();
	        if (selectedProxy.type() != Proxy.Type.HTTP) {
	          throw new ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy");
	        }
	        return client.proxyAuthenticator().authenticate(route, userResponse);
	
	      case HTTP_UNAUTHORIZED:
	        return client.authenticator().authenticate(route, userResponse);
	
	      case HTTP_PERM_REDIRECT:
	      case HTTP_TEMP_REDIRECT:
	        // "If the 307 or 308 status code is received in response to a request other than GET
	        // or HEAD, the user agent MUST NOT automatically redirect the request"
	        if (!method.equals("GET") && !method.equals("HEAD")) {
	          return null;
	        }
	        // fall-through
	      case HTTP_MULT_CHOICE:
	      case HTTP_MOVED_PERM:
	      case HTTP_MOVED_TEMP:
	      case HTTP_SEE_OTHER:
	        // Does the client allow redirects?
	        if (!client.followRedirects()) return null;
	
	        String location = userResponse.header("Location");
	        if (location == null) return null;
	        HttpUrl url = userResponse.request().url().resolve(location);
	
	        // Don't follow redirects to unsupported protocols.
	        if (url == null) return null;
	
	        // If configured, don't follow redirects between SSL and non-SSL.
	        boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
	        if (!sameScheme && !client.followSslRedirects()) return null;
	
	        // Redirects don't include a request body.
	        Request.Builder requestBuilder = userResponse.request().newBuilder();
	        if (HttpMethod.permitsRequestBody(method)) {
	          if (HttpMethod.redirectsToGet(method)) {
	            requestBuilder.method("GET", null);
	          } else {
	            requestBuilder.method(method, null);
	          }
	          requestBuilder.removeHeader("Transfer-Encoding");
	          requestBuilder.removeHeader("Content-Length");
	          requestBuilder.removeHeader("Content-Type");
	        }
	
	        // When redirecting across hosts, drop all authentication headers. This
	        // is potentially annoying to the application layer since they have no
	        // way to retain them.
	        if (!sameConnection(userResponse, url)) {
	          requestBuilder.removeHeader("Authorization");
	        }
	
	        return requestBuilder.url(url).build();
	
	      case HTTP_CLIENT_TIMEOUT:
	        // 408's are rare in practice, but some servers like HAProxy use this response code. The
	        // spec says that we may repeat the request without modifications. Modern browsers also
        // repeat the request (even non-idempotent ones.)
        if (userResponse.request().body() instanceof UnrepeatableRequestBody) {
          return null;
        }

        return userResponse.request();

      default:
        return null;
    }
  }

 
 
	

* ####桥接拦截器BridgeInterceptor

桥接拦截器的主要作用是将：
	

1. 请求从应用层数据类型类型转化为网络调用层的数据类型。
	
	![image](http://of8cu1h2w.bkt.clouddn.com/%E6%A1%A5%E6%8E%A5%E6%8B%A6%E6%88%AA%E5%99%A8%E7%9A%84%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E8%BD%AC%E6%8D%A2.png)
	
		
2. 将网络层返回的数据类型 转化为 应用层数据类型。
	
			1. 保存最新的cookie(默认没有cookie，需要应用程序自己创建，详见
						[Cookie的API]
						(https://square.github.io/okhttp/3.x/okhttp/okhttp3/CookieJar.html)
						和
						[Cookie的持久化]
						(https://segmentfault.com/a/1190000004345545))；
			2. 如果request中使用了"gzip"压缩，则进行Gzip解压。解压完毕后移除Header中的"Content-Encoding"和"Content-Length"（因为Header中的长度对应的是压缩前数据的长度，解压后长度变了，所以Header中长度信息实效了）；
			3. 返回response。
		
>  补充：Keep－Alive 连接：
HTTP中的keepalive连接在网络性能优化中，对于延迟降低与速度提升的有非常重要的作用。
通常我们进行http连接时，首先进行tcp握手，然后传输数据，最后释放
![image](http://upload-images.jianshu.io/upload_images/98641-9c8a016af59f675f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这种方法的确简单，但是在复杂的网络内容中就不够用了，创建socket需要进行3次握手，而释放socket需要2次握手(或者是4次)。重复的连接与释放tcp连接就像每次仅仅挤1mm的牙膏就合上牙膏盖子接着再打开接着挤一样。而每次连接大概是TTL一次的时间(也就是ping一次)，在TLS环境下消耗的时间就更多了。很明显，当访问复杂网络时，延时（而不是带宽）将成为非常重要的因素。
当然，上面的问题早已经解决了，在http中有一种叫做keepalive connections的机制，它可以在传输数据后仍然保持连接，当客户端需要再次获取数据时，直接使用刚刚空闲下来的连接而不需要再次握手
![image](http://upload-images.jianshu.io/upload_images/98641-71b1fdaf78b8442c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在现代浏览器中，一般同时开启6～8个keepalive connections的socket连接，并保持一定的链路生命，当不需要时再关闭；而在服务器中，一般是由软件根据负载情况决定是否主动关闭。
		
		

* ####缓存拦截器CacheInterceptor

缓存拦截器的主要作用是将请求 和 返回 关连得保存到缓存中。客户端与服务端根据一定的机制，在需要的时候使用缓存的数据作为网络请求的响应，节省了时间和带宽。

###### 客户端与服务端之间的缓存机制：

* 作用：将HTPTP和HTTPS的网络返回数据缓存到文件系统中，以便在服务端数据没发生变化的情况下复用，节省时间和带宽；
* 工作原理：客户端发器网络请求，如果缓存文件中有一份与该请求匹配（URL相同）的完整的返回数据（比如上一次请求返回的结果），那么客户端就会发起一个带条件（例子，服务端第一次返回数据时，在response的Header中添加上次修改的时间信息：Last-Modified: Tue, 12 Jan 2016 09:31:27 GMT；客户端再次请求的时候，在request的Header中添加这个时间信息：If-Modified-Since: Tue, 12 Jan 2016 09:31:27 GMT）的获取资源的请求。此时服务端根据客户端请求的条件，来判断该请求对应的数据是否有更新。如果需要返回的数据没有变化，那么服务端直接返回 304 "not modified"。客户端如果收到响应码味304 的信息，则直接使用缓存数据。否则，服务端直接返回更新的数据。具体如下图所示：
![image](http://of8cu1h2w.bkt.clouddn.com/%E7%BC%93%E5%AD%98%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png)

###### 客户端缓存的实现：

OKHttp3的缓存类为Cache类，它实际上是一层缓存逻辑的包装类。内部有个专门负责缓存文件读写的类：DiskLruCache。于此同时，OKHttp3还定义了一个缓存接口：InternalCache。这个缓存接口类作为Cache的成员变量其所有的实现，都是调用了Cahce类的函数实现的。它们间具体的关系如下：

![image](http://of8cu1h2w.bkt.clouddn.com/%E7%BC%93%E5%AD%98%E7%B1%BB%E7%9A%84%E6%9E%B6%E6%9E%84.png)

InternalCache接口：

		public interface InternalCache {
		  Response get(Request request) throws IOException;
		
		  CacheRequest put(Response response) throws IOException;
		
		  /**
		   * 移除request相关的缓存数据
		   */
		  void remove(Request request) throws IOException;
		
		  /**
		   * 用网络返回的数据，更新缓存中数据
		   */
		  void update(Response cached, Response network);
		
		  /** 统计网络请求的数据 */
		  void trackConditionalCacheHit();
		
		  /** 统计网络返回的数据 */
		  void trackResponse(CacheStrategy cacheStrategy);
		}


>Note:setInternalCache这个方法是不对应用程序开放的，应用程序只能使用cache(Cache cache)这个方法来设置缓存。并且，Cache内部的internalCache是final的，不能被修改。总结：internalCache这个接口虽然是public的，但实际上，应用程序是无法创建它，并附值到OkHttpClient中去的。

> 那么问题来了，为什么不用Cahce实现InternalCache这个接口，而是以组合的方式，在它的内部实现都调用了Cache的方法呢？
> 因为：

* 在Cache中，InternalCache接口的两个统计方法：trackConditionalCacheHit和trackResponse (之所以要统计，是为了查看缓存的效率。比如总的请求次数与缓存的命中次数。)需要用内置锁进行同步。
* Cache中，将trackConditionalCacheHit和trackResponse方法 从public变为为privite了。不允许子类重写，也不开放给应用程序。

Cache类：

* 将网络请求的文件读写操作委托给内部的DiskLruCache类来处理。
* 负责将url转换为对应的key。
* 负责数据统计：写成功计数，写中断计数，网络调用计数，请求数量计数，命中数量计数。
* 负责网络请求的数据对象Response 与 写入文件系统中的文本数据 之间的转换。使用内部类Entry来实现。具体如下：

		网络请求的文件流文本格式如下：
				{
				     http://google.com/foo
				     GET
				     2
				     Accept-Language: fr-CA
				     Accept-Charset: UTF-8
				     HTTP/1.1 200 OK
				     3
				     Content-Type: image/png
				     Content-Length: 100
				     Cache-Control: max-age=600
				     ...
				  }	  	
		内存对象的Entry格式如下.
				private static final class Entry {
				    private final String url;
				    private final Headers varyHeaders;
				    private final String requestMethod;
				    private final Protocol protocol;
				    private final int code;
				    private final String message;
				    ...
				｝			
通过okio的读写API，实现它们之间灵活的切换。

DiskLruCache类：

简介：一个有限空间的文件缓存。

* 每个缓存数据都有一个string类型的key和一些固定数量的值。
* 缓存的数据保存在文件系统的一个目录下。这个目录必须是该缓存独占的：因为缓存运行时会删除和修改该目录下的文件，因而该缓存目录不能被其他线程使用。
* 缓存限制了总的文件大小。如果存储的大小超过了限制，会以LRU算法来移除一些数据。
* 可以通过edit,update方法来修改缓存数据。每次调用edit，都必须以commit或者abort结束。commit是原子操作。
* 客户端调用get方法来读取一个缓存文件的快照(存储了key,快照序列号，数据源和数据长度)。


缓存使用了日志文件(文件名为journal)来存储缓存的数据目录和操作记录。一个典型的日志文件的文本文档如下：

		 //第一行为缓存的名字
		 libcore.io.DiskLruCache                            
		 1										              //缓存的版本
		 100                                                 //应用版本
		 2                                                   //值的数量 		 
		 //缓存记录：操作 key 第一个数据的长度 第二个数据的长度    
		 CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054    
	     DIRTY 335c4c6028171cfddfbaae1a9c313c52
	     CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
	     REMOVE 335c4c6028171cfddfbaae1a9c313c52
	     DIRTY 1ab96a171faeeee38496d8b330771a7a
	     CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
	     READ 335c4c6028171cfddfbaae1a9c313c52
	     READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
	     
操作记录：状态＋key＋额外擦数

* CLEAN key param0 param1：该key的对应的数据为最新的有效数据，后续为额外的参数
* DIRTY key：该key对应的数据被创建或被修改。   
* REMOVE key：该key对应的数据被删除。
* READ key：改key对应的数据被读取的记录。——用于LRU算法来统计哪些数据是最新的数据。

一些冗余的操作记录，比如DIRTY,REMOVE...比较多的时候(大于2000个，或者超过总数量)，会发器线程对该日志文件进行压缩(删除这些冗余的日志记录)。此时，会创建一个journal.tmp文件作为临时的文件，供缓存继续使用。同时还有个journal.bkp文件，用作journal文件的临时备份。
换文文件结构如下：
![image](http://of8cu1h2w.bkt.clouddn.com/%E7%BC%93%E5%AD%98%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84.png)

工作原理：
 
 * 每个缓存记录在内存中的对象封装为Entry类
 
		  private final class Entry {		  
		    private final String key; //缓存对应的key
			
		    private final long[] lengths; //文件的长度
		    private final File[] cleanFiles; //有效的数据文件
		    private final File[] dirtyFiles; //正在修改的数据文件
		
		    private boolean readable;//是否可读
		
		    private Editor currentEditor;//当前的编辑器。一个Entry一个时候只能被一个编辑器修改。
	
		    private long sequenceNumber; //唯一序列号，相当于版本号，当内存中缓存数据发生变化时，该序列号会改变。
			
			...    
			｝
 * 创建缓存快照对象，作为每次读取缓存时的一个内容快照对象：
 
		  public final class Snapshot implements Closeable {
		    private final String key;  	//缓存的key
		    private final long sequenceNumber; //创建快照的时候的缓存序列号
		    private final Source[] sources;//数据源,okio的API，可以直接读取
		    private final long[] lengths;//数据长度。
		    ...
		    ｝
	内容快照作为读去缓存的对象(而不是将Entry直接返回)的作用：
	(1). 内容快照的数据结构方式更便于数据的读取(将file转换为source)，并且隐藏了Entry的细节；
	(2). 内容快照在创建的时候记录了当时的Entry的序列号。因而可以用快照的序列号与缓存的序列号对比，如果序列号不相同，则说明缓存数据发生了修改，该条数据就是失效的。
> * 缓存内容Entry的这种工作机制（单个editor，带有序列号的内容快照）以最小的代价，实现了单线程修改，多线程读写的数据对象（否则则需要使用复杂的锁机制）既添降低了逻辑的复杂性，又提高了性能（缺点就是高并发情况下，导致数据频繁失效，导致缓存的命中率降低）。
> * 变化的序列号计数在很多涉及并发读取的机制都有使用。比如：SQlite的连接。

*  DiskLruCache缓存文件工作流：
	![image](http://of8cu1h2w.bkt.clouddn.com/%E7%BC%93%E5%AD%98%E6%96%87%E4%BB%B6%E5%B7%A5%E4%BD%9C%E6%B5%81.png)
其返回的SnapShot数据快照，提供了Source接口(okio),供外部内类Cache直接转化为内存对象Cache.Entry。外部类进一步Canche.Entry转化外OKHttpClient使用的Response。

OkHttpClient从缓存中获取一个url对应的缓存数据的数据格式变化过程如下：
![image](http://of8cu1h2w.bkt.clouddn.com/%E7%BC%93%E5%AD%98%E6%95%B0%E6%8D%AE%E7%9A%84%E6%A0%BC%E5%BC%8F%E5%8F%98%E5%8C%96.png)

LRU的算法体现在：DiskLreCache的日志操作过程中，每一次读取缓存都产生一个READ的记录。由于缓存的初始化是按照日志文件的操作记录顺序来读取的，所以这相当于把这条缓存数据放置到了缓存队列的顶端，也就完成了LRU算法：last recent used，最近使用到的数据最新。

#### 连接拦截器 和 最后的请求服务器的拦截器 
这两个连接器基本上完成了最后发起网络请求的工作。追所以划分为两个拦截器，除了解耦之外，更重要的是在这两个流程之间还可以插入一个专门为WebSocket服务的拦截器( [WebSocket](https://zh.wikipedia.org/wiki/WebSocket)一种在单个 TCP 连接上进行全双工通讯的协议,本文不做详解)。

关于OKHttp如何真正发起网络请求的，下面专门详细讲解。

## 发起网络请求

* 简介：OKHttp的网络请求的实现是[socket](https://zh.wikipedia.org/wiki/Berkeley%E5%A5%97%E6%8E%A5%E5%AD%97)(应用程序与网络层进行交互的API)。socket发起网络请求的流程一般是：
(1). 创建socket对象;
(2). 连接到目标网络;
(3). 进行输入输出流操作。
* 在OKHttp框架里面，(1)(2)的实现，封装在connection接口中，具体的实现类是RealConnection。（3）是通过stream接口来实现，根据不同的网络协议，有Http1xStream和Http2xStream两个实现类。
* 由于创建网络连接的时间较久(如果是HTTP的话，需要进行三次握手)，而请求经常是频繁的碎片化的，所以为了提高网络连接的效率，OKHttp3实现了网络连接复用:
> * 新建的连接connection会存放到一个缓存池connectionpool中。网络连接完成后不会立即释放，而是存活一段时间。网络连接存活状态下，如果有相同的目标连接，则复用该连接，用它来进行写入写出流操作。
> * 统计每个connection上发起网络请求的次数，若次数为0，则一段时间后释放该连接。
> * 每个网络请求对应一个stream，connection，connectionpool等数据，将它封装为StreamAllocation对象。

具体的流程见下图：
![image](http://upload-images.jianshu.io/upload_images/3406294-649407c17ad3f694.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


类之间的关系如下：

![image](http://upload-images.jianshu.io/upload_images/3406294-03393d9820192aae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

几个主要的概念：

###### Connection:连接。
真正的底层实现网络连接的接口。它包含了连接的路线，物理层socket，连接协议，和握手信息。

	public interface Connection {
	  /** 返回连接线路信息(包涵url，dns，proxy) */
	  Route route();
	
	  /**
	   * 返回连接使用的Socket(网络层到应用层之间的一层封装)
	   */
	  Socket socket();
	
	  /**
	   * 返回传输层安全协议的握手信息
	   */
	  Handshake handshake();
	
	  /**
	   * 返回网络协议：Http1.0,Http1.1,Http2.0...
	   */
	  Protocol protocol();
	}
在OKHttp3中的实现：RealConnection.
> * 包含了连接的信息，包括socket，它是与网络层交互的接口，真正实现网络连接；
> * 内部维护了一个列表List<StreamAllocation>，相当于发起网络请求的引用计数容器。下面详细讨论。
> * 包涵输入输出流，source和sink。但是Connection只负责将socket的操作，与source和sink建立起连接，针对source和sink的写入和读取操作，交给响应的Stream完成（解耦）。将socket封装为okio的代码：
> 
		source = Okio.buffer(Okio.source(socket));
		sink = Okio.buffer(Okio.sink(socket));
这就将socket的读写操作标准华为okio的API。
> * 还有个内部类framedConnection，专门用来处理Http2和SPDY(goole推出的网络协议)的（不详细讨论）。


###### Stream: 完成网络请求的读写流程功能。
接口，源码如下：
	
	public interface HttpStream {
	
	
	  /** 创建请求的输出流*/
	  Sink createRequestBody(Request request, long contentLength);
	
	  /** 写请求的头部信息(Header) */
	  void writeRequestHeaders(Request request) throws IOException;
	
	  /** 将请求写入sokcet */
	  void finishRequest() throws IOException;
	
	  /** 读取网络返回的头部信息(header)*/
	  Response.Builder readResponseHeaders() throws IOException;
	
	  /** 返回网络返回的数据 */
	  ResponseBody openResponseBody(Response response) throws IOException;
	
	  /**
	   * 异步的取消，并释放相关的资源。(connect pool的线程会自动完成)
	   */
	  void cancel();
	}

> * 这个功能实际上是从connection中剥离出来：connection负责底层连接的实现，其写入request和读取response的功能交给stream来完成。
> * 之所以需要将该功能解耦出来，一个重要的原因：根据不同的网络请求协议，request的写入和response的读出，具有不同的实现。比如http1,http2等，响应的类为Http1xStream，Http2xStream。

###### ConnectionPool:内存连接池
负责管理HTTP连接，可以复用相同Address（包括url，dns，port，是否加密,proxy等信息）的连接，以减少网络延迟。
> * 维护了一个双端队列，默认的实现是：最多5个空闲的连接，这5个连接最多存活5分钟。
> * 直到连接被清除，底层的socket连接才会释放。

###### StreamAllocation: 网络流的分配计数器(下文简称SA)
计数功能：
每发起一个网络请求，都会新建SA对象。由于连接connection可以被复用，所以一个connection可以对应多次网络请求，即多个SA。所以RealConnection中维护了一个SA的列表。每次创建新的SA的时候，会在对应的connection的列表＋1。以下情形下，会把列表－1，并释放SA对应的资源:

* 网络请求完成；
* 网络请求时发生异常；
* 重定向，则release原来的网络请求，并新建新的；
* 重定向次数超过限制(OKHttp3最大20)；
* 网络返回信息头部包涵"Connection:close"的信息；
* 网络返回的数据信息长度未知，则响应的connection不再分配stream。
 
 该引用计数的列表的作用：统计一个connection对应的网络请求的数量，如果为空，则connection进入空闲，开始倒计时回收。未回收之前的connection都可以复用，降低网络延迟(因为创建和销毁连接的开销比较长，如果时HTTP，则需要三次握手)。
 
SA中的方法：

* newStream: 创建新的连接，并从connectionPool中找能够复用的connection，如果没有能复用的，则新建connection，并加入到connectionPool中。
* acquire：网络被创建，将对应的connection中SA的引用＋1.
* release：网络请求结束，将对应的connection中SA的引用－1.


###### 最后的发起网络请求的代码
		
		@Override 
		public Response intercept(Chain chain) throws IOException {
		
		    //获取httpstream对象，在之前的ConnectInterceptor中创建
		    HttpStream httpStream = ((RealInterceptorChain) chain).httpStream();
		    StreamAllocation streamAllocation = ((RealInterceptorChain) chain).streamAllocation();
		    Request request = chain.request();
		
		    long sentRequestMillis = System.currentTimeMillis();
		    
		    //写请求的header
		    httpStream.writeRequestHeaders(request);
		    
			//写请求的body
		    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
		      Sink requestBodyOut = httpStream.createRequestBody(request, request.body().contentLength());
		      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
		      request.body().writeTo(bufferedRequestBody);
		      bufferedRequestBody.close();
		    }
		    
			//完成网络请求
		    httpStream.finishRequest();
		
			//读取网络请求返回的header
		    Response response = httpStream.readResponseHeaders()
		        .request(request)
		        .handshake(streamAllocation.connection().handshake())
		        .sentRequestAtMillis(sentRequestMillis)
		        .receivedResponseAtMillis(System.currentTimeMillis())
		        .build();
		
			//读取网络请求返回的body
		    if (!forWebSocket || response.code() != 101) {
		      response = response.newBuilder()
		          .body(httpStream.openResponseBody(response))
		          .build();
		    }
						
		    ...
		    
			//返回结果
		    return response;
			}  
		  
## 深入网络请求Socket
未完待续...
