## 实现简易Retrofit



### 注解部分

#### 方法注解

@OnClick注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface OnClick {
    int[] value();
}
```

@GET注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface GET {
    String value() default "";
}
```

@POST注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface POST {
    String value() default "";
}
```

#### 变量注解

@BindView注解

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface BindView {
    int value();
}
```

#### 方法参数注解

@Field注解

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Field {
    String value();
}
```

@Query注解

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Query {
    String value();
}
```

### 页面使用部分

```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "easy";
    @BindView(R.id.tv_request)
    TextView tvRequest;
    @BindView(R.id.tv_response)
    TextView tvResponse;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        InjectUtil.inject(this);
    }

    @OnClick(R.id.tv_request)
    public void clickRequest(View view) {
        Toast.makeText(this, "click request", Toast.LENGTH_SHORT).show();
        Retrofit retrofit = new Retrofit.Builder().baseUrl("https://api.seniverse.com/").build();
        ServiceApi serviceApi = retrofit.create(ServiceApi.class);
        serviceApi.getWeather("S5AaXUmC6YtmVNhix", "beijing").enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.d(TAG, "onFailure: ");
            }

            @Override
            public void onResponse(Call call, final Response response) throws IOException {
                Log.d(TAG, "onResponse: ");
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            tvResponse.setText(response.body().string());
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                });
            }
        });
    }
}
```

### 注入类

```java
public class InjectUtil {

    public static void inject(final Activity activity) {
        Class<? extends Activity> object = activity.getClass();
        // 获取activity中的成员变量
        Field[] fields = object.getDeclaredFields();
        for (Field field : fields) {
            // 判断如果有被BindView注解修饰
            if (field.isAnnotationPresent(BindView.class)) {
                BindView annotation = field.getAnnotation(BindView.class);
                if (null != annotation) {
                    field.setAccessible(true);
                    try {
                        // 设置值给activity中的变量，第一个参数为要设置的属性所属对象，如果是静态变量，可以传null
                        field.set(activity, activity.findViewById(annotation.value()));
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

        Method[] methods = object.getDeclaredMethods();
        for (final Method method : methods) {
            if (method.isAnnotationPresent(OnClick.class)) {
                OnClick annotation = method.getAnnotation(OnClick.class);
                Object clickListener = Proxy.newProxyInstance(View.OnClickListener.class.getClassLoader(), new Class[]{View.OnClickListener.class}, new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method1, Object[] args) throws Throwable {
                        // method是属于activity的方法，所以这里第一个参数是activity
                        return method.invoke(activity, args);
                    }
                });

                int[] ids = annotation.value();
                for (int id : ids) {
                    final View view = activity.findViewById(id);
                    final Method clickMethod;
                    try {
                        clickMethod = view.getClass().getMethod("setOnClickListener", View.OnClickListener.class);
                        // 动态代理创建View.OnClickListener对象
                        clickMethod.invoke(view, clickListener);
                    } catch (NoSuchMethodException | InvocationTargetException | IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

    }
}
```

### 反射实现retrofit部分

Retrofit类

```java
/**
 * Retrofit 入口类
 * ServiceMethod 记录请求信息
 * ParameterHandler 各种类型参数处理的抽象类
 */
public class Retrofit {
    private HttpUrl baseUrl;
    private Call.Factory callFactory;
    private final HashMap<Method, ServiceMethod> serviceMethodCache = new HashMap<>();

    public Retrofit(HttpUrl baseUrl, Call.Factory callFactory) {
        this.baseUrl = baseUrl;
        this.callFactory = callFactory;
    }

    public HttpUrl baseUrl() {
        return baseUrl;
    }

    public Call.Factory callFactory() {
        return callFactory;
    }

    public <T> T create(final Class<T> service) {
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class[]{service}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                ServiceMethod serviceMethod = loadServiceMethod(method);
                return serviceMethod.invoke(args);
            }
        });
    }

    public ServiceMethod loadServiceMethod(Method method) {
        ServiceMethod result = serviceMethodCache.get(method);
        if (null != result) {
            return result;
        }
        synchronized (serviceMethodCache) {
            result = serviceMethodCache.get(method);
            if (null == result) {
                result = new ServiceMethod.Builder(this, method).build();
                serviceMethodCache.put(method, result);
            }
        }
        return result;
    }

    public static class Builder {
        private HttpUrl baseUrl;
        private Call.Factory callFactory;

        public Builder() {
        }

        public Builder baseUrl(String baseUrl) {
            this.baseUrl = HttpUrl.parse(baseUrl);
            return this;
        }

        public Builder callFactory(Call.Factory callFactory) {
            this.callFactory = callFactory;
            return this;
        }

        public Retrofit build() {
            if (null == baseUrl) {
                throw new IllegalArgumentException("Base URl required");
            }
            Call.Factory factory = callFactory;
            if (null == factory) {
                factory = new OkHttpClient();
            }
            return new Retrofit(baseUrl, factory);
        }
    }

}
```

ServiceMethod类

```java
public class ServiceMethod {
    private Call.Factory callFactory;
    private ParameterHandler[] parameterHandlers;
    private HttpUrl.Builder urlBuilder;
    private HttpUrl baseUrl;
    private String relativeUrl;
    private String httpMethod;
    private FormBody.Builder bodyBuilder;
    private boolean hasBody;

    public ServiceMethod(Builder builder) {
        this.callFactory = builder.retrofit.callFactory();
        this.hasBody = builder.hasBody;
        this.relativeUrl = builder.relativeUrl;
        this.baseUrl = builder.retrofit.baseUrl();
        this.httpMethod = builder.httpMethod;
        this.parameterHandlers = builder.parameterHandlers;
        if (hasBody) {
            bodyBuilder = new FormBody.Builder();
        }
    }

    public void addQueryParam(String key, String value) {
        if (null == urlBuilder) {
            urlBuilder = baseUrl.newBuilder(relativeUrl);
        }
        urlBuilder.addQueryParameter(key, value);
    }

    public void addFieldParam(String key, String value) {
        bodyBuilder.add(key, value);
    }

    public static class Builder {
        private Retrofit retrofit;
        private String httpMethod;
        private boolean hasBody;
        private String relativeUrl;

        private Annotation[] methodAnnotations;
        private Annotation[][] paramAnnotations;
        private ParameterHandler[] parameterHandlers;

        public Builder(Retrofit retrofit, Method method) {
            this.retrofit = retrofit;
            methodAnnotations = method.getAnnotations();
            paramAnnotations = method.getParameterAnnotations();
        }

        public ServiceMethod build() {
            for (Annotation methodAnnotation : methodAnnotations) {
                if (methodAnnotation instanceof GET) {
                    this.httpMethod = "GET";
                    this.hasBody = false;
                    this.relativeUrl = ((GET) methodAnnotation).value();
                } else if (methodAnnotation instanceof POST) {
                    this.httpMethod = "POST";
                    this.hasBody = true;
                    this.relativeUrl = ((POST) methodAnnotation).value();
                }
            }
            // 参数个数
            int paramCount = paramAnnotations.length;
            parameterHandlers = new ParameterHandler[paramCount];
            for (int i = 0; i < paramCount; i++) {
                Annotation[] annotations = paramAnnotations[i];
                for (Annotation annotation : annotations) {
                    if (annotation instanceof Query) {
                        parameterHandlers[i] = new ParameterHandler.QueryParamHandler(((Query) annotation).value());
                    } else if (annotation instanceof Field) {
                        parameterHandlers[i] = new ParameterHandler.FieldParamHandler(((Field) annotation).value());
                    }
                }
            }
            return new ServiceMethod(this);
        }
    }

    private String url;

    public Call invoke(Object[] args) {
        // 构建请求url
        if (null == urlBuilder) {
            urlBuilder = baseUrl.newBuilder(relativeUrl);
        }

        for (int i = 0; i < parameterHandlers.length; i++) {
            parameterHandlers[i].apply(this, args[i].toString());
        }
        FormBody body = null;
        if (null != bodyBuilder) {
            body = bodyBuilder.build();
        }
        // 构建请求request
        Request request = new Request.Builder().method(httpMethod, body).url(urlBuilder.toString()).build();
        return callFactory.newCall(request);
    }
}
```

ParameterHandler

```java
public abstract class ParameterHandler {


    abstract void apply(ServiceMethod serviceMethod, String value);

    static class QueryParamHandler extends ParameterHandler {
        private String key;

        public QueryParamHandler(String key) {
            this.key = key;
        }

        @Override
        void apply(ServiceMethod serviceMethod, String value) {
            serviceMethod.addQueryParam(key, value);
        }
    }

    static class FieldParamHandler extends ParameterHandler {
        private String key;

        public FieldParamHandler(String key) {
            this.key = key;
        }

        @Override
        void apply(ServiceMethod serviceMethod, String value) {
            serviceMethod.addFieldParam(key, value);
        }
    }
}
```