---
　　layout: post
　　title: 你好，世界
---

## 前言
	为什么 RestTemplate 提供简洁api同时，又可高度可展? 主要是使用了如下的设计模式:  

创建模式:  
- factory 工厂模式

行为模式:  
- Chain of responsibility 责任链模式

我们从请求的创建到返回响应给client的顺序，介绍 RestTemplate 所使用的设计模式.

### 请求创建
RestTemplate 使用 factory 模式，来**创建** http request,这样的好处是，我们可以针对 http request 选择不同的实现、添加丰富的功能。
目前常见的factory有:
- SimpleClientHttpRequestFactory (使用jdk自带的http工具,可设置代理)
- BufferingClientHttpRequestFactory(能够重复读io流的factory，基于装饰器模式)
- InterceptingClientHttpRequestFactory(基于装饰器+责任链模式,提供了修改请求、响应的功能)
- RibbonClientHttpRequestFactory(提供了负载均衡能力)

RestTemplate默认使用了SimpleClientHttpRequestFactory 和InterceptingClientHttpRequestFactory 这两个factory. SimpleClientHttpRequestFactory 使用JDK自带的http请求工具处理http底层传输，
InterceptingClientHttpRequestFactory 封装了SimpleClientHttpRequestFactory,在此基础上通过责任链模式提供ClientHttpRequestInterceptor 拦截器接口给我们拦截request、response以实现 添加权限验证、负载均衡等功能。

在下面示例的请求中，我们创建一个拦截器，通过debug源代码，我们可以了解factory、拦截器是如何应用的。

```java
//拦截器
public interface ClientHttpRequestInterceptor {
	/**
     * 通过拦截器，我们可以修改 HttpRequest 和 ClientHttpResponse
     */
	ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution)
			throws IOException;

}

public void createRequest(){
	RestTemplate restTemplate = new RestTemplate();
	List<ClientHttpRequestInterceptor> interceptors = new ArrayList<>();
	interceptors.add(new BasicAuthenticationInterceptor("userName","password"));
    // debug 进入 getForObject方法,最终到doExecute方法
	String html = restTemplate.getForObject("https://dante-sketch.github.io/",String.class);
}
```
RestTemplate 暴露对外的方法，最终都会变成调用RestTemplate的doExecute方法. doExecute 使用createRequest方法创建 ClientHttpRequest.
createRequest 是 factory模式和责任链模式真正使用的地方。
```java
	@Nullable
	protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
			@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

		Assert.notNull(url, "URI is required");
		Assert.notNull(method, "HttpMethod is required");
		ClientHttpResponse response = null;
		try {
			ClientHttpRequest request = createRequest(url, method);
			if (requestCallback != null) {
				requestCallback.doWithRequest(request);
			}
			response = request.execute();
			handleResponse(url, method, response);
			return (responseExtractor != null ? responseExtractor.extractData(response) : null);
		}
		catch (IOException ex) {
			String resource = url.toString();
			String query = url.getRawQuery();
			resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
			throw new ResourceAccessException("I/O error on " + method.name() +
					" request for \"" + resource + "\": " + ex.getMessage(), ex);
		}
		finally {
			if (response != null) {
				response.close();
			}
		}
	}
	protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
		// getRequestFactory()方法通过工厂模式加拦截器
		ClientHttpRequest request = getRequestFactory().createRequest(url, method);
		if (logger.isDebugEnabled()) {
			logger.debug("HTTP " + method.name() + " " + url);
		}
		return request;
	}

	// 这个方法使用了装饰器模式，用InterceptingClientHttpRequestFactory 封装 SimpleClientHttpRequestFactory
	@Override
	public ClientHttpRequestFactory getRequestFactory() {
		// getInterceptors 就是用于扩展的接口，我们使用权限验证 BasicAuthenticationInterceptor 就是添加到这里
		List<ClientHttpRequestInterceptor> interceptors = getInterceptors();
		if (!CollectionUtils.isEmpty(interceptors)) {
			ClientHttpRequestFactory factory = this.interceptingRequestFactory;
			if (factory == null) {
				factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
				this.interceptingRequestFactory = factory;
			}
			return factory;
		}
		else {
			return super.getRequestFactory();
		}
	}
```
我们看看 InterceptingClientHttpRequestFactory 这个factory，是如何使用 SimpleClientHttpRequestFactory 和 拦截器 创建 ClientHttpRequest 的。
```java
public class InterceptingClientHttpRequestFactory extends AbstractClientHttpRequestFactoryWrapper {

	private final List<ClientHttpRequestInterceptor> interceptors;


	/**
	 * Create a new instance of the {@code InterceptingClientHttpRequestFactory} with the given parameters.
	 * @param requestFactory the request factory to wrap
	 * @param interceptors the interceptors that are to be applied (can be {@code null})
	 */
	public InterceptingClientHttpRequestFactory(ClientHttpRequestFactory requestFactory,
			@Nullable List<ClientHttpRequestInterceptor> interceptors) {

		super(requestFactory);
		this.interceptors = (interceptors != null ? interceptors : Collections.emptyList());
	}


	@Override
	protected ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod, ClientHttpRequestFactory requestFactory) {
		// 返回了具有拦截功能的 request
		return new InterceptingClientHttpRequest(requestFactory, this.interceptors, uri, httpMethod);
	}
}
```
InterceptingClientHttpRequestFactory 返回了 ClientHttpRequst 的具体实现类: InterceptingClientHttpRequest.
InterceptingClientHttpRequest 封装了 ClientHttpRequest 的具体实现。
当调用 ClientHttpRequest的execute方法时，委托的factory(比如SimpleClientHttpRequestFactory)开始创建请求.
然后，InterceptingClientHttpRequest 使用了责任链模式，将 拦截器封装在 InterceptingRequestExecution(类比Servlet的FilterChain),
然后拦截 request 和 response 。
```java
class InterceptingClientHttpRequest extends AbstractBufferingClientHttpRequest {
	
	@Override
	protected final ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
		InterceptingRequestExecution requestExecution = new InterceptingRequestExecution();
		return requestExecution.execute(this, bufferedOutput);
	}


	private class InterceptingRequestExecution implements ClientHttpRequestExecution {

		private final Iterator<ClientHttpRequestInterceptor> iterator;

		public InterceptingRequestExecution() {
			this.iterator = interceptors.iterator();
		}

		@Override
		public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
			if (this.iterator.hasNext()) {
				ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
				return nextInterceptor.intercept(request, body, this);
			}
			else {
				HttpMethod method = request.getMethod();
				Assert.state(method != null, "No standard HTTP method");
				ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
				request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
				if (body.length > 0) {
					if (delegate instanceof StreamingHttpOutputMessage) {
						StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
						streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
					}
					else {
						StreamUtils.copy(body, delegate.getBody());
					}
				}
				return delegate.execute();
			}
		}
	}
}

```

### 真正的request处理
刚才我们讲到，factory 用于创建 request。RestTemplate 使用了 InterceptingClientHttpRequestFactory 来添加拦截器，以实现功能的扩展(request,response修改)
SimpleClientHttpRequestFactory 则用于实现真正的reques请求(实现http数据传输)

```java
public class SimpleClientHttpRequestFactory implements ClientHttpRequestFactory, AsyncClientHttpRequestFactory {
	@Override
	public ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException {
		//HttpURLConnection 是JDK自带的http工具
		HttpURLConnection connection = openConnection(uri.toURL(), this.proxy);
		prepareConnection(connection, httpMethod.name());

		if (this.bufferRequestBody) {
			return new SimpleBufferingClientHttpRequest(connection, this.outputStreaming);
		}
		else {
			return new SimpleStreamingClientHttpRequest(connection, this.chunkSize, this.outputStreaming);
		}
	}	
}
// 将 HttpURLConnection 封装成 spring自己的 ClientHttpRequest
final class SimpleBufferingClientHttpRequest extends AbstractBufferingClientHttpRequest {
	// 请求的真正执行，基于 HttpURLConnection
	@Override
	protected ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
		addHeaders(this.connection, headers);
		// JDK <1.8 时， http 的 delete 方法不支持传输消息体，所以是一个bug。
		// 这就是为什么我们使用 RestTemplate的一个原因之一，因为它可以通过factory模式，替换 request 的具体实现，比如我们可以替换成基于netty的Netty4ClientHttpRequestFactory
		if (getMethod() == HttpMethod.DELETE && bufferedOutput.length == 0) {
			this.connection.setDoOutput(false);
		}
		if (this.connection.getDoOutput() && this.outputStreaming) {
			this.connection.setFixedLengthStreamingMode(bufferedOutput.length);
		}
		this.connection.connect();
		if (this.connection.getDoOutput()) {
			FileCopyUtils.copy(bufferedOutput, this.connection.getOutputStream());
		}
		else {
			// Immediately trigger the request in a no-output scenario as well
			this.connection.getResponseCode();
		}
		return new SimpleClientHttpResponse(this.connection);
	}
}
```

到了这里，我们基于探索完了整个的request创建和请求的过程。
需要简单补充的是，RestTemplate以插件的方式，提供了HttpMessageConverter 接口，用于让我们自定义reques和response能够支持的读写方法。
在没有解读Resttemplate方法以前，每每看看 ```postForObject(String url, @Nullable Object request, Class<T> responseType,```方法中的 Object request 参数时，
总是让我纳闷不已，不知道如何下手，看了代码才知道，Object request 参数是http 的body。需要我们提供适当的 HttpMessageConverter 来将其写入到HttpOutputMessage里面。
由于RestTemplate 默认提供了很多的 HttpMessageConverter ，所以多数情况下，我们无需自己编写 HttpMessageConverter.

### 基于拦截器的应用

#### 权限验证
RestTemplate 基于工厂模式和 责任链模式，提供了很好的扩展性。基于此我们可以做很多的事情，比如刚才的权限验证:

{% highlight ruby %}
```java
public class BasicAuthenticationInterceptor implements ClientHttpRequestInterceptor {

	private final String username;

	private final String password;

	@Nullable
	private final Charset charset;

	public BasicAuthenticationInterceptor(String username, String password) {
		this(username, password, null);
	}

	public BasicAuthenticationInterceptor(String username, String password, @Nullable Charset charset) {
		Assert.doesNotContain(username, ":", "Username must not contain a colon");
		this.username = username;
		this.password = password;
		this.charset = charset;
	}


	@Override
	public ClientHttpResponse intercept(
			HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {

		HttpHeaders headers = request.getHeaders();
		//添加请求验证信息
		if (!headers.containsKey(HttpHeaders.AUTHORIZATION)) {
			headers.setBasicAuth(this.username, this.password, this.charset);
		}
		//将请求通过责任链ClientHttpRequestExecution交给下一个拦截器处理
		return execution.execute(request, body);
	}
}
```
{% endhighlight %}

#### 负载均衡LoadBalance
