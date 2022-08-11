# springboot-Web 内容协商原理

## 内容协商是什么？

springmvc内容协商主要用在@ResponseBody标注的方法的返回值上。可以根据**请求中指定的客户端支持的数据类型**和**服务器能够返回的数据类型**来决定最终返回的数据类型（json? Xml? text?）。

## 简单使用

```xml
				<!--导入xml数据解析格式化支持-->
				<dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-xml</artifactId>
        </dependency
```

```java
@RestController
public class HelloController {

  	// 请求头中指定客户端需要的数据类型为application/json
  	// curl -XGET http://localhost:8080/hello -H 'accept:application/json'
  	// {"name":"zs","age":18,"type":null}
  
  	// 请求头中指定客户端需要的数据类型为application/xml
  	// curl -XGET http://localhost:8080/hello -H 'accept:application/xml'
  	// <A><name>zs</name><age>18</age><type/></A>
    @GetMapping("hello")
    public A hello2() throws IOException {
        A a = new A();
        a.setName("zs");
        a.setAge(18);
        return a;
    }

}
```

可以看到，我们在不增加接口实现的情况下返回了不同格式的数据，这就是内容协商，**接口返回什么格式的数据由客户端和服务端共同决定**

## 实现原理

1. 我们知道返回值的处理是由返回值解析器（HandlerMethodReturnValueHandler）完成的，而标注了@ResponseBody注解的返回值是由RequestResponseBodyMethodProcessor处理的

2. RequestResponseBodyMethodProcessor类的writeWithMessageConverters方法中就包含了内容协商的处理逻辑

   1. 从请求中获取客户端需要的媒体类型
   2. 获取服务端能够产生的媒体类型
   3. 获取交集，并从中取出第一个具体的类型（类型和子类型都不是*）
   4. 寻找能够支持这种类型的消息转换器，利用它来写数据

   ```java
   protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
   			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
   			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
   
   		// ......
   
   		MediaType selectedMediaType = null;
   		MediaType contentType = outputMessage.getHeaders().getContentType();
   		boolean isContentTypePreset = contentType != null && contentType.isConcrete();
   		if (isContentTypePreset) {
   			if (logger.isDebugEnabled()) {
   				logger.debug("Found 'Content-Type:" + contentType + "' in response");
   			}
   			selectedMediaType = contentType;
   		}
   		else {
   			HttpServletRequest request = inputMessage.getServletRequest();
   			List<MediaType> acceptableTypes;
   			try {
           // 获取请求中指定的客户端需要的内容类型
   				acceptableTypes = getAcceptableMediaTypes(request);
   			}
   			catch (HttpMediaTypeNotAcceptableException ex) {
   				int series = outputMessage.getServletResponse().getStatus() / 100;
   				if (body == null || series == 4 || series == 5) {
   					if (logger.isDebugEnabled()) {
   						logger.debug("Ignoring error response content (if any). " + ex);
   					}
   					return;
   				}
   				throw ex;
   			}
         // 获取服务端能够产生的内容类型
   			List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
   
   			if (body != null && producibleTypes.isEmpty()) {
   				throw new HttpMessageNotWritableException(
   						"No converter found for return value of type: " + valueType);
   			}
         // 找到交集
   			List<MediaType> mediaTypesToUse = new ArrayList<>();
   			for (MediaType requestedType : acceptableTypes) {
   				for (MediaType producibleType : producibleTypes) {
   					if (requestedType.isCompatibleWith(producibleType)) {
   						mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
   					}
   				}
   			}
   			if (mediaTypesToUse.isEmpty()) {
   				if (body != null) {
   					throw new HttpMediaTypeNotAcceptableException(producibleTypes);
   				}
   				if (logger.isDebugEnabled()) {
   					logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
   				}
   				return;
   			}
   
   			MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
   
         // 决定真正返回的数据类型（具体的而不是通配的），匹配到了马上就跳出
   			for (MediaType mediaType : mediaTypesToUse) {
   				if (mediaType.isConcrete()) {
   					selectedMediaType = mediaType;
   					break;
   				}
   				else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
   					selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
   					break;
   				}
   			}
   
   			if (logger.isDebugEnabled()) {
   				logger.debug("Using '" + selectedMediaType + "', given " +
   						acceptableTypes + " and supported " + producibleTypes);
   			}
   		}
   
   		if (selectedMediaType != null) {
   			selectedMediaType = selectedMediaType.removeQualityValue();
         // 看哪个消息转换器能够支持写这种格式的数据，使用它来写数据
   			for (HttpMessageConverter<?> converter : this.messageConverters) {
   				GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
   						(GenericHttpMessageConverter<?>) converter : null);
   				if (genericConverter != null ?
   						((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
   						converter.canWrite(valueType, selectedMediaType)) {
   					body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
   							(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
   							inputMessage, outputMessage);
   					if (body != null) {
   						Object theBody = body;
   						LogFormatUtils.traceDebug(logger, traceOn ->
   								"Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
   						addContentDispositionHeader(inputMessage, outputMessage);
   						if (genericConverter != null) {
   							genericConverter.write(body, targetType, selectedMediaType, outputMessage);
   						}
   						else {
   							((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
   						}
   					}
   					else {
   						if (logger.isDebugEnabled()) {
   							logger.debug("Nothing to write: null body");
   						}
   					}
   					return;
   				}
   			}
   		}
   
     	// ......
   	}
   ```

## 如何拓展

首先必须将我们自定义的媒体类型注**册到服务器能够产生的媒体类型中**，并且我们要**能够解析客户端传上来自定义的媒体类型**。第二点就是需要**有一个消息转换器来处理我们自定义的媒体类型**，通过它将数据写回去。

通过源码分析，解析客户端类型的方法是getAcceptableMediaTypes，获取服务端能够产生的媒体类型的方法是getProducibleMediaTypes。而getProducibleMediaTypes方法的逻辑是**获取消息转换器支持的媒体类型作为能够生产的类型**。所以我们分析getAcceptableMediaTypes方法即可。通过debug设置断点的方式可以知道，他是通过ContentNegotiationStrategy类来处理的。所以怎么拓展就很清晰了

1. 将我们自定义的媒体类型加入到ContentNegotiationStrategy支持的类型中

2. 编写支持该媒体类型的消息转换器

   ```java
   @Configuration
   public class WebConfig implements WebMvcConfigurer {
   
   
       @Override
       public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
           // 将我们自定义的媒体类型加入到ContentNegotiationStrategy支持的类型中
           configurer.mediaType("jjggzz",MediaType.parseMediaType("application/x-jjggzz"));
       }
   
       @Override
       public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
           // 自定义消息转换器
           HttpMessageConverter<A> myConverter = new HttpMessageConverter<A>() {
               @Override
               public boolean canRead(Class<?> clazz, MediaType mediaType) {
                   return false;
               }
   
               @Override
               public boolean canWrite(Class<?> clazz, MediaType mediaType) {
                 	// 允许写
                   return true;
               }
   
               @Override
               public List<MediaType> getSupportedMediaTypes() {
                 	// 返回支持的媒体类型
                   return MediaType.parseMediaTypes("application/x-jjggzz");
               }
   
               @Override
               public A read(Class<? extends A> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
                   return null;
               }
   
               @Override
               public void write(A a, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
                 	// 此处就是简单的将参数拼接一下
                   String str = a.getName() + "_" + a.getAge();
                   outputMessage.getBody().write(str.getBytes());
               }
           };
           // 将自定义的消息转换器加入到消息转换器列表中并排在前列
           converters.add(0,myConverter);
       }
   }
   ```

3. 发起请求进行测试

```shell
~ % curl -XGET http://localhost:8080/hello -H 'accept:application/x-jjggzz'
zs_18                           
```



