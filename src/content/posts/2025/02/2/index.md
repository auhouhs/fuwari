---
title: Spring Boot应用内接口转发
published: 2025-02-27 11:57:44
description: 实现一个应用内网关，统一的对外接口接收数据并把请求分发到对应的控制器.
#image: ./cover.jpg
tags: [Java, SpringBoot]
category: 后端
draft: false
---

>最近有一个需求，为了应对应用不能随时升级（升级会中断使用，但是不影响）的这种情况，微服务专门设置了一个网关会把接到的请求转发给单独部署的一个服务，这个服务不再属于微服务架构内的应用，算是单体应用，这个服务除了服务注册和服务发现相关代码其他和线上版本相同，这个服务也不涉及远程调用，这样可以随时升级，但是网关在请求转发时并不会转给应用的具体接口，只会转给应用的一个统一的接口，这样就需要应用内对接到的数据进行识别，并把请求转给对应的处理器，这个操作不好说具体叫什么名字，但是实际操作起来类似一个应用内网关，负责请求的转发或者说分发。

## 操作步骤
### 1、前后端统一标准
首先，具体访问哪个接口，与前段组的同事协商好，web传参时明确指定访问的具体接口，比如

```java
{
    "command": "/service/conf/list",
    ... // 该接口需要的其他参数
}
```

### 2、开发实现
#### 2.1、HttpServletRequest包装
因为需要读取body内的数据，判断具体调用哪个处理器，而HttpServletRequest的输入流只能读取一次，这样我们需要对HttpServletRequest进行一次包装，保证输入流可以重复读取（代码随便在网上找，很多类似的实现）：
```java
public class BodyCopyHttpServletRequestWrapper extends HttpServletRequestWrapper {
 
    private final String UTF_8 = "UTF-8";
 
    /**
     * 输入流
     */
    private final byte[] bytes;
 
    public BodyCopyHttpServletRequestWrapper(HttpServletRequest request) throws IOException {
        super(request);
        // 备份
        // 如不关心字符集，也可以直接在方法cloneInputStream()中byteArrayOutputStream.toByteArray()
        bytes = getBodyString(request).getBytes(StandardCharsets.UTF_8);
    }
 
    /**
     * 获取请求Body
     * @param request
     * @return
     */
    private String getBodyString(final ServletRequest request) {
        StringBuilder sb = new StringBuilder();
        try (
                InputStream inputStream = cloneInputStream(request.getInputStream());
                BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, StandardCharsets.UTF_8))
        ) {
            String line = "";
            while (Objects.nonNull((line = reader.readLine()))) {
                sb.append(line);
            }
        } catch (IOException e) {
            throw new RuntimeException("输入流读取出错");
        }
        return sb.toString();
    }
 
    /**
     * 输入流复制
     * @param inputStream
     * @return
     */
    private InputStream cloneInputStream(ServletInputStream inputStream) {
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        byte[] buffer = new byte[1024];
        int len;
        try {
            while ((len = inputStream.read(buffer)) > -1) {
                byteArrayOutputStream.write(buffer, 0, len);
            }
            byteArrayOutputStream.flush();
        } catch (IOException e) {
            throw new RuntimeException("复制输入流读取出错");
        }
        return new ByteArrayInputStream(byteArrayOutputStream.toByteArray());
    }
 
    @Override
    public BufferedReader getReader() throws IOException {
        return new BufferedReader(new InputStreamReader(getInputStream()));
    }
 
    /**
     * 重写父方法，返回新的输入流
     * @return
     * @throws IOException
     */
    @Override
    public ServletInputStream getInputStream() throws IOException {
        final ByteArrayInputStream copyStream = new ByteArrayInputStream(bytes);
        /**
         * 新的输入流
         */
        return new ServletInputStream() {
 
            @Override
            public int read() throws IOException {
                return copyStream.read();
            }
 
            /**
             * 未读状态
             * @return
             */
            @Override
            public boolean isFinished() {
                return false;
            }
 
            @Override
            public boolean isReady() {
                return false;
            }
 
            @Override
            public void setReadListener(ReadListener readListener) {
            }
        };
    }
}
```
然后我们需要在过滤器中对HttpServletRequest进行包装，其中的/portal即外部转发过来的统一入口地址：
```java
@WebFilter(filterName = "inputStreamFilter", urlPatterns = "/portal")
public class InputStreamFilter implements Filter {
 
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        String contentType = request.getContentType();
        if (request instanceof HttpServletRequest) {
            HttpServletRequest requestWrapper = new BodyCopyHttpServletRequestWrapper((HttpServletRequest) request);
            if (contentType != null && contentType.contains("multipart/form-data")) {
                chain.doFilter(request, response);
            } else {
                chain.doFilter(requestWrapper, response);
            }
            return;
        }
        chain.doFilter(request, response);
    }
}
```
完成后记得在启动类添加注解@ServletComponentScan("com.request.filter"),com.request.filter为过滤器所处位置的包名
#### 2.2、请求的转发

然后就是最重要的转发逻辑
```java
public class PortalHandlerMapping extends AbstractHandlerMapping implements Serializable {

    private final ApplicationContext applicationContext;
    private final BeanDefinitionRegistry beanDefinitionRegistry;


    public PortalHandlerMapping(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
        this.beanDefinitionRegistry = (BeanDefinitionRegistry) ((ConfigurableApplicationContext) applicationContext).getBeanFactory();
    }

    @Override
    protected Object getHandlerInternal(HttpServletRequest httpServletRequest) throws Exception {

        String uri = httpServletRequest.getRequestURI();
        if (!uri.startsWith("/portal")) {
            return null;
        }

        String content = StreamUtils.copyToString(httpServletRequest.getInputStream(), StandardCharsets.UTF_8);
        ObjectMapper objectMapper = new ObjectMapper();
        JsonNode jsonNode = objectMapper.readTree(content);
        String type = jsonNode.get("command").asText();
        Map<String, Object> beansWithAnnotation = applicationContext.getBeansWithAnnotation(RestController.class);
        
        for (Map.Entry<String, Object> value : beansWithAnnotation.entrySet()) {
            Method[] declaredMethods = value.getValue().getClass().getDeclaredMethods();
            RequestMapping annotation = value.getValue().getClass().getAnnotation(RequestMapping.class);
            String classPath;
            if (annotation != null) {
                classPath = annotation.value()[0];
            } else {
                classPath = "";
            }
            // 以下是查找逻辑
            for (Method method : declaredMethods) {
                if (method.isAnnotationPresent(PostMapping.class)) {
                    String[] value1 = method.getAnnotation(PostMapping.class).value();
                    Optional<String> first = Arrays.stream(value1).map(v -> {
                        if (v.endsWith("/")) {
                            return combinePath(classPath, v.substring(0, v.length() - 1));
                        }
                        return combinePath(classPath, v);
                    }).filter(v -> v.toUpperCase().endsWith("/" + type.toUpperCase()) || v.equalsIgnoreCase(type)).findFirst();
                    if (first.isPresent()) {
                        return new HandlerMethod(applicationContext.getBean(value.getKey()), method);
                    }
                }
            }
        }
        return null;
    }
    
    public String combinePath(String start, String end) {
        if (end.startsWith("/")) {
            end = end.substring(1);
        }
        
        if (start.endsWith("/")) {
            start = start.substring(0, start.length() - 1);
        }
        
        return start + "/" + end;
    }
}
```
#### 2.3、注册PortalHandlerMapping
最后记得将这个PortalHandlerMapping注册一下
```java
@Bean
public HandlerMapping customHandlerMapping(ApplicationContext applicationContext) {
    PortalHandlerMapping portalHandlerMapping = new PortalHandlerMapping(applicationContext);
    portalHandlerMapping.setOrder(0);
    return portalHandlerMapping;
}
```

## 结尾
代码写的可能不是很细致，因为我们的接口全部为@PostMapping所以实现起来很简单，查找对应处理的过程应当springboot中有可以直接调用的方法，这个没有细究，如需使用可以查一下
