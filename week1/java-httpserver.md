# Java HttpServer

## contents

### Java HttpServer란?

Java 에서 제공하는 내부적으로 [NIO](<https://en.wikipedia.org/wiki/Non-blocking_I/O_(Java)>)를 사용하는 [고수준 HTTP 서버](https://docs.oracle.com/en/java/javase/17/docs/api/jdk.httpserver/com/sun/net/httpserver/package-summary.html) API.

### Java HttpServer Example

#### 1. 서버 객체 준비 & Listen

```<Java>
/*
 * This Java source file was generated by the Gradle 'init' task.
 */
package com.gyo.java.httpserver;

import com.sun.net.httpserver.HttpServer;

import java.io.IOException;
import java.net.InetSocketAddress;

public class App {

    public static void main(String[] args) throws IOException {
        App app = new App();
        app.run();
    }

    private void run() throws IOException {
        // Listener 구현
        InetSocketAddress address = new InetSocketAddress(8080);
        HttpServer httpServer = HttpServer.create(address, 0);  // backlog는 기본값으로 설정

        // Listen
        httpServer.start();
    }
}

```

어플리케이션을 실행하면 서버가 listen한다. 그리고 curl로 요청을 보내보자.

```<bash>
>> 입력
 curl localhost:8080
 >> 출력
<h1>404 Not Found</h1>No context found for request
```

HttpServer 객체에서 기본적으로 404 Not Found 응답을 해주는 것을 알 수 있다.

#### 2. URL(정확히는 path)에 핸들러 지정

서버(리스너)를 start하기 전에 핸들러를 지정해야 한다.

##### HttpContext

HttpContext는 어떤 어플리케이션의 루트 URI 경로와 그 경로로 들어오는 요청을 처리하는 HttpHandler 사이의 mapping이다. 즉, 특정 URI 경로로 들어오는 요청이 처리될 때 호출되는 HttpHandler를 관리하는 역할을 한다.

##### handler 와 람다 표현식

핸들러는 [HTTP Exchange](https://docs.oracle.com/en/java/javase/17/docs/api/jdk.httpserver/com/sun/net/httpserver/HttpExchange.html)를 처리하기 위해 호출(invoked)된다.

```<java>
package com.sun.net.httpserver;

import java.io.IOException;

public interface HttpHandler {

    public abstract void handle (HttpExchange exchange) throws IOException;
}

```

이 인터페이스는 추상 메서드가 하나 뿐이다. 즉 [함수형 인터페이스](https://www.baeldung.com/java-8-functional-interfaces)이다. 따라서 람다 표현식을 사용할 수 있다. 이런 인터페이스는 명시적으로 @FunctionalInterface 어노테이션이 붙어 있지 않더라도, 컴파일러가 함수형 인터페이스로 판단하게 된다.

#### 3. Request

Http Exchange에 Request에 관한 데이터가 포함되어 있다.

##### HTTP Method

```<java>
String method = exchange.getRequestMethod();
System.out.println(method);	// 출력해서 확인
```

##### HTTP path

```<java>
URI uri = exchange.getRequestURI();
String path = uri.getPath();
System.out.println(path); // 출력해서 확인
```

##### Headers

```<java>
Headers headers = exchange.getRequestHeaders();
for (String key : headers.keySet()) {
System.out.println(key + ": " + headers.get(key)); // 출력해서 확인
}
```

##### Body

```<java>
InputStream inputStream = exchange.getRequestBody();
String body = new String(inputStream.readAllBytes());
System.out.println(body); // 출력해서 확인
```

##### HTTPie를 통해 body 넣어서 요청보내기

어플리케이션 코드는 아래와 같다.

```<java>
/*
 * This Java source file was generated by the Gradle 'init' task.
 */
package com.gyo.java.httpserver;

import com.sun.net.httpserver.Headers;
import com.sun.net.httpserver.HttpServer;

import java.io.IOException;
import java.io.InputStream;
import java.net.InetSocketAddress;
import java.net.URI;
import java.util.Arrays;
import java.util.List;

public class App {

    public static void main(String[] args) throws IOException {
        App app = new App();
        app.run();
    }

    private void run() throws IOException {
        // Listener 구현
        InetSocketAddress address = new InetSocketAddress(8080);
        HttpServer httpServer = HttpServer.create(address, 0);  // backlog는 기본값으로 설정

        // Handler 지정
        httpServer.createContext("/", exchange -> {
            // Request

            // method
            String method = exchange.getRequestMethod();
            System.out.println("Method : " + method); // 메서드 출력

            // path
            URI uri = exchange.getRequestURI();
            String path = uri.getPath();
            System.out.println("uri path : " + path); // 메서드 출력

            // headers key, value
            Headers requestHeaders = exchange.getRequestHeaders(); // map
            for (String key : requestHeaders.keySet()) {
                List<String> values = requestHeaders.get(key);
                System.out.println(key + " : " + values);
            }

            // body
            InputStream inputStream = exchange.getRequestBody(); // stream으로 들어온다.
            byte[] bytes = inputStream.readAllBytes();
            String body = Arrays.toString(bytes);
            System.out.println(body);

            // Response
        });

        // Listen
        httpServer.start();
    }

}

```

curl로 기본적인 GET요청을 보내면 Body가 없다.

```<bash>
 curl localhost:8080
```

```<bash>
>> 어플리케이션 콘솔
Method : GET
uri path : /
Accept : [*/*]
Host : [localhost:8080]
User-agent : [curl/8.1.2]
```

HTTPie를 통해서 Request Body를 추가해보자.

```<bash>
 http localhost:8080 name=tester
```

```<bash>
>> 어플리케이션 콘솔
Method : POST
uri path : /
Accept-encoding : [gzip, deflate]
Accept : [application/json, */*;q=0.5]
Connection : [keep-alive]
Host : [localhost:8080]
User-agent : [HTTPie/3.2.2]
Content-type : [application/json]
Content-length : [18]
{"name": "tester"}
```

###### 참고

```<java>
String body = Arrays.toString(bytes);
```

이렇게 코드를 작성하면, byte 배열에서 각 요소에 toString()을 호출해서 []안에 담게된다.
이렇게 하면 문제점은 byte타입의 toString()은 아스키코드 값을 호출하게 되서 [72, 101, 108, 108, 111] 이런 값이 출력된다.

#### 4. Response

```<java>
// Response

// content
String content = "Hello, world!";
// Content-Lenght를 위해서 byte 배열로 변경
byte[] contentBytes = content.getBytes();

// Response Header : HTTP Status Code와 Content-Length 지정.
exchange.sendResponseHeaders(200, contentBytes.length);

// ResponseBody에 body 입력 : outputStream을 얻고, 스트림에 write
OutputStream outputStream = exchange.getResponseBody();
outputStream.write(contentBytes);
outputStream.flush();
```

HTTP Status Code와 Content-Length 지정해야한다.
Content-Lenght 때문에 바이트 배열을 통해서 length값을 얻어야 하는 것에 주의하자.

이제 curl로 요청을 보내면 body를 확인할 수 있다.

```<bash>
>> 입력
curl localhost:8080
>> 출력
Hello, world!%
```

##### 하위 path에 대한 헨들러를 추가해보자.

```<java>
// root 하위 path에 대한 handler 지정
        httpServer.createContext("/sub", exchange -> {
            // Response

            // content
            String content = "Hello, sub!\n";
            // Content-Lenght를 위해서 byte 배열로 변경
            byte[] contentBytes = content.getBytes();

            // Response Header : HTTP Status Code와 Content-Length 지정.
            exchange.sendResponseHeaders(200, contentBytes.length);

            // ResponseBody에 body 입력 : outputStream을 얻고, 스트림에 write
            OutputStream outputStream = exchange.getResponseBody();
            outputStream.write(contentBytes);
            outputStream.flush();
        });

```

```<bash>
>> 입력
curl localhost:8080/sub
>> 출력
Hello, sub!
```