---
layout:     post
title:      "retrofit日志拦截"
subtitle:   " \"Hello World, Hello Android\""
date:       2016-12-16 13:00:00
author:     "Cuieney"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 源码
---

> 在开发中经常会出现，莫名其妙的bug，突然你的app crash了，这时候我们就需要前后台的联调，bug也不知道出现在哪里，这是你肯定会是不是后台api的问题啊，一个接着一个的打印log，如果你使用retrofit的话可能找起来比较麻烦。不用怕教你两招，二步搞定。


### 第一步
	
1.retrofit大家肯定都有所了解吧，一个很知名的网络请求框架，但是它是以注解方式，来把url分成两部分，一个是BaseUrl和请求的地址拼接在一起的，但是呢，你调试的时候不可能每次都去这边打一下log 那边断点调试。很是麻烦。
	
2.retrofit嘛他是一个基于okhttp的网络请求框架，那么我们要想拿到这些数据，肯定通过okhttp了。从retrofit这里进行拿的话可能很费劲，但是如果我们从okhttp这里拿这些数据的话，那是方便了很多。

3.说了也不少了，大家现在知道应该从哪里出发如何获取这些信息，方便我们调试，那么我们在用okhttp的时候，知道要配置一些信息吧，构建一些东西，这时候我们只需要配置一下他的拦截器就好。如果你仔细往里面找的话，可能会发现okhttp有这么一个类HttpLoggingInterceptor，哈哈哈没错就是log日志拦截器，他已经帮我们写好了。我们只需要吧相应的东西打印出来就好

```

HttpLoggingInterceptor loggingInterceptor = new HttpLoggingInterceptor();

loggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);

return new OkHttpClient.Builder()
                .cache(cache)//添加缓存
                .addInterceptor(loggingInterceptor)
                .addInterceptor(cacheInterceptor)
                .sslSocketFactory(sslContext.getSocketFactory())
                .hostnameVerifier(DO_NOT_VERIFY)
//                .cookieJar(cookiesManager)
                .build();


```

从上面的代码可以看到这个类，只不过我已经吧这个类提取出来了，同时对立面的log进行了一些处理，就变成了我自己的log拦截器了。

### 第二步

那就是这个类了，我直接把类扔上来大家自己看看。立面其实写的很详细

```
public final class HttpLoggingInterceptor implements Interceptor {
    private static final Charset UTF8 = Charset.forName("UTF-8");

    public enum Level {
        /** No logs. */
        NONE,
        /**
         * Logs request and response lines.
         *
         * <p>Example:
         * <pre>{@code
         * --> POST /greeting http/1.1 (3-byte body)
         *
         * <-- 200 OK (22ms, 6-byte body)
         * }</pre>
         */
        BASIC,
        /**
         * Logs request and response lines and their respective headers.
         *
         * <p>Example:
         * <pre>{@code
         * --> POST /greeting http/1.1
         * Host: example.com
         * Content-Type: plain/text
         * Content-Length: 3
         * --> END POST
         *
         * <-- 200 OK (22ms)
         * Content-Type: plain/text
         * Content-Length: 6
         * <-- END HTTP
         * }</pre>
         */
        HEADERS,
        /**
         * Logs request and response lines and their respective headers and bodies (if present).
         *
         * <p>Example:
         * <pre>{@code
         * --> POST /greeting http/1.1
         * Host: example.com
         * Content-Type: plain/text
         * Content-Length: 3
         *
         * Hi?
         * --> END GET
         *
         * <-- 200 OK (22ms)
         * Content-Type: plain/text
         * Content-Length: 6
         *
         * Hello!
         * <-- END HTTP
         * }</pre>
         */
        BODY
    }

    public interface Logger {
        void log(String message);

        /** A {@link Logger} defaults output appropriate for the current platform. */
        Logger DEFAULT = new Logger() {
            @Override public void log(String message) {
                Platform.get().log(message);
            }
        };
    }

    public HttpLoggingInterceptor() {
        this(Logger.DEFAULT);
    }

    public HttpLoggingInterceptor(Logger logger) {
        this.logger = logger;
    }

    private final Logger logger;

    private volatile Level level = Level.NONE;

    /** Change the level at which this interceptor logs. */
    public HttpLoggingInterceptor setLevel(Level level) {
        if (level == null) throw new NullPointerException("level == null. Use Level.NONE instead.");
        this.level = level;
        return this;
    }

    public Level getLevel() {
        return level;
    }

    @Override public Response intercept(Chain chain) throws IOException {
        Level level = this.level;

        Request request = chain.request();
        if (level == Level.NONE) {
            return chain.proceed(request);
        }

        boolean logBody = level == Level.BODY;
        boolean logHeaders = logBody || level == Level.HEADERS;

        RequestBody requestBody = request.body();
        boolean hasRequestBody = requestBody != null;

        Connection connection = chain.connection();
        Protocol protocol = connection != null ? connection.protocol() : Protocol.HTTP_1_1;
        String requestStartMessage = "--> " + request.method() + ' ' + request.url() + ' ' + protocol;
        if (!logHeaders && hasRequestBody) {
            requestStartMessage += " (" + requestBody.contentLength() + "-byte body)";
        }
        logger.log(requestStartMessage);

        if (logHeaders) {
            if (hasRequestBody) {
                // Request body headers are only present when installed as a network interceptor. Force
                // them to be included (when available) so there values are known.
                if (requestBody.contentType() != null) {
                    logger.log("Content-Type: " + requestBody.contentType());
                }
                if (requestBody.contentLength() != -1) {
                    logger.log("Content-Length: " + requestBody.contentLength());
                }
            }

            Headers headers = request.headers();
            for (int i = 0, count = headers.size(); i < count; i++) {
                String name = headers.name(i);
                // Skip headers from the request body as they are explicitly logged above.
                if (!"Content-Type".equalsIgnoreCase(name) && !"Content-Length".equalsIgnoreCase(name)) {
                    logger.log(name + ": " + headers.value(i));
                }
            }

            if (!logBody || !hasRequestBody) {
                logger.log("--> END " + request.method());
            } else if (bodyEncoded(request.headers())) {
                logger.log("--> END " + request.method() + " (encoded body omitted)");
            } else {
                Buffer buffer = new Buffer();
                requestBody.writeTo(buffer);

                Charset charset = UTF8;
                MediaType contentType = requestBody.contentType();
                if (contentType != null) {
                    charset = contentType.charset(UTF8);
                }

                logger.log("");
                if (isPlaintext(buffer)) {
                    logger.log(buffer.readString(charset));
                    logger.log("--> END " + request.method()
                            + " (" + requestBody.contentLength() + "-byte body)");
                } else {
                    logger.log("--> END " + request.method() + " (binary "
                            + requestBody.contentLength() + "-byte body omitted)");
                }
            }
        }

        long startNs = System.nanoTime();
        Response response;
        try {
            response = chain.proceed(request);
        } catch (Exception e) {
            logger.log("<-- HTTP FAILED: " + e);
            throw e;
        }
        long tookMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startNs);

        ResponseBody responseBody = response.body();
        long contentLength = responseBody.contentLength();
        String bodySize = contentLength != -1 ? contentLength + "-byte" : "unknown-length";
        logger.log("<-- " + response.code() + ' ' + response.message() + ' '
                + response.request().url() + " (" + tookMs + "ms" + (!logHeaders ? ", "
                + bodySize + " body" : "") + ')');

        if (logHeaders) {
            Headers headers = response.headers();
            for (int i = 0, count = headers.size(); i < count; i++) {
                logger.log(headers.name(i) + ": " + headers.value(i));
            }

            if (!logBody || !HttpEngine.hasBody(response)) {
                logger.log("<-- END HTTP");
            } else if (bodyEncoded(response.headers())) {
                logger.log("<-- END HTTP (encoded body omitted)");
            } else {
                BufferedSource source = responseBody.source();
                source.request(Long.MAX_VALUE); // Buffer the entire body.
                Buffer buffer = source.buffer();

                Charset charset = UTF8;
                MediaType contentType = responseBody.contentType();
                if (contentType != null) {
                    try {
                        charset = contentType.charset(UTF8);
                    } catch (UnsupportedCharsetException e) {
                        logger.log("");
                        logger.log("Couldn't decode the response body; charset is likely malformed.");
                        logger.log("<-- END HTTP");

                        return response;
                    }
                }

                if (!isPlaintext(buffer)) {
                    logger.log("");
                    logger.log("<-- END HTTP (binary " + buffer.size() + "-byte body omitted)");
                    return response;
                }

                if (contentLength != 0) {
                    logger.log("");
                    logger.log(buffer.clone().readString(charset));
                    StringBuilder sb = new StringBuilder();
                    sb.append("method:")
                            .append(request.method())
                            .append(";")
                            .append("url:")
                            .append(request.url())
                            .append(";");

                    if (!(!logBody || !hasRequestBody||bodyEncoded(request.headers()))){
                        Buffer buffer1 = new Buffer();
                        requestBody.writeTo(buffer1);

                        if (isPlaintext(buffer1)) {
                            sb.append("body:")
                                    .append(buffer1.readString(charset))
                                    .append(";");
                        }
                    }


                    sb.append("response:")
                            .append(buffer.clone().readString(charset))
                            .append(";");
//                    L.sendLogToServer("httpDetail",sb.toString());
                }else{
                    StringBuilder sb = new StringBuilder();
                    sb.append("method:")
                            .append(request.method())
                            .append(";")
                            .append("\n")
                            .append("url:")
                            .append(request.url())
                            .append(";")
                            .append("\n");

                    if (!(!logBody || !hasRequestBody||bodyEncoded(request.headers()))){
                        Buffer buffer1 = new Buffer();
                        requestBody.writeTo(buffer1);

                        MediaType contentType1 = requestBody.contentType();
                        if (contentType != null) {
                            charset = contentType1.charset(UTF8);
                        }
                        if (isPlaintext(buffer1)) {
                            sb.append("body:")
                                    .append(buffer.readString(charset))
                                    .append(";")
                                    .append("\n");
                        }
                    }
                    sb.append("response:")
                            .append(response.code())
                            .append(";")
                            .append("\n");
//                    L.sendLogToServer("httpDetail",sb.toString());
                }

                logger.log("<-- END HTTP (" + buffer.size() + "-byte body)");
            }
        }

        return response;
    }



    public void printLogToService(Chain chain, int code) throws IOException {
        StringBuilder sb = new StringBuilder();

        boolean logBody = level == Level.BODY;
        Request request = chain.request();
        RequestBody requestBody = request.body();
        boolean hasRequestBody = requestBody != null;

        sb.append("method:")
                .append(request.method())
                .append(";")
                .append("url:")
                .append(request.url())
                .append(";");


        if (!logBody || !hasRequestBody) {
            android.util.Log.i("djx","--> END " + request.method());
        } else if (bodyEncoded(request.headers())) {
            android.util.Log.i("djx","--> END " + request.method() + " (encoded body omitted)");
        }else{
            Buffer buffer = new Buffer();
            requestBody.writeTo(buffer);

            Charset charset = UTF8;
            MediaType contentType = requestBody.contentType();
            if (contentType != null) {
                charset = contentType.charset(UTF8);
            }

            if (isPlaintext(buffer)) {
                sb.append("body:")
                        .append(buffer.readString(charset))
                        .append(";");
            }
        }


        Response response;
        try {
            response = chain.proceed(request);
        } catch (Exception e) {
            logger.log("<-- HTTP FAILED: " + e);
            throw e;
        }


        ResponseBody responseBody = response.body();
        BufferedSource source = responseBody.source();
        source.request(Long.MAX_VALUE); // Buffer the entire body.
        Buffer rspButter = source.buffer();
        if (code != 200){
            sb.append("response:")
                    .append(code)
                    .append(";");
        }else{

            Charset charset = UTF8;
            if (!isPlaintext(rspButter)) {
                sb.append("response:")
                        .append(rspButter.clone().readString(charset))
                        .append(";");
            }
        }

//        L.sendLogToServer("httpDetail",sb.toString());
        android.util.Log.i("djx", "printLogToService: "+sb.toString());

    }
    /**
     * Returns true if the body in question probably contains human readable text. Uses a small sample
     * of code points to detect unicode control characters commonly used in binary file signatures.
     */
    static boolean isPlaintext(Buffer buffer) throws EOFException {
        try {
            Buffer prefix = new Buffer();
            long byteCount = buffer.size() < 64 ? buffer.size() : 64;
            buffer.copyTo(prefix, 0, byteCount);
            for (int i = 0; i < 16; i++) {
                if (prefix.exhausted()) {
                    break;
                }
                int codePoint = prefix.readUtf8CodePoint();
                if (Character.isISOControl(codePoint) && !Character.isWhitespace(codePoint)) {
                    return false;
                }
            }
            return true;
        } catch (EOFException e) {
            return false; // Truncated UTF-8 sequence.
        }
    }

    private boolean bodyEncoded(Headers headers) {
        String contentEncoding = headers.get("Content-Encoding");
        return contentEncoding != null && !contentEncoding.equalsIgnoreCase("identity");
    }
}

```
### 结束

复制粘贴即可最后放一下效果图,从请求开始到结束，完整的url，头信息，body，response，都很清楚

![log.png](http://upload-images.jianshu.io/upload_images/3415839-ffb8aef49ffd3627.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果喜欢可以关注一下。不定期更新技术文章


—— Cuieney 后记于 2017.03


