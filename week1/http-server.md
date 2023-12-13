# HTTP Server

## contents

### HTTP Server Example with Java

#### 1. 작업 디렉토리로 가서 gradle init

[Http Client 예제에서 gralde init](./http-client.md)와 동일하다.

#### 2. Listen

다른 곳에 접속하는 게 아니라 우리가 만든 서버에 접속하기 때문에 포트 번호만 정하면 된다. 만약 80 포트를 사용 중이라면 8080 등 다른 포트 번호를 쓰면 된다.

```java
int port = 8080;
```

Java에서는 ServerSocket이라는 별도의 클래스를 사용한다(Socket을 상속한 게 아니라, 완전히 구별된다는 점에 주의하자).  
이렇게 구분하는 이유로는 ServerSocket은 단지 Listen만 하는 용도라서 Java에서 별도로 지원해준다.

```java
ServerSocket listener = new ServerSocket(port);
```

##### backlog

backlog는 서버 소켓이 들어오는 연결 요청을 대기하는 큐의 크기를 나타낸다. 즉 매개변수 backlog의 값은 ServerSocket 클래스에서 서버 소켓이 수용할 수 있는 최대 크기를 말한다. backlog 큐가 가득 찬 상태에서 새로운 연결 요청이 들어온다면 거부된다.

리스너는 클라이언트의 접속을 기다린다. 클라이언트가 접속하면 통신용 소켓을 새로 반환해서 돌려준다.

```java
Socket socket = listener.accept();
```

##### accept

1. 서버 소켓은 bind 메서드로 지정된 포트에 바인딩 되어 대기상태(listening)에 들어간다. 참고로 소켓을 생성할 때 포트 번호를 매개변수로 넣어줬기 때문에 [bind](https://stackoverflow.com/questions/6090891/what-is-socket-bind-and-how-to-bind-an-address) 과정이 포함되었다.

2. accept는 네트워크 소켓 프로그래밍에서 서버 소켓이 클라이언트의 연결 요청을 받아들이는 역할을 하는 메서드다. 이 메서드는 Blocking 호출이라 클라이언트가 연결 요청을 보낼 때 까지 실행이 차단(block)된다.  
   파일 읽기, 쓰기 등도 모두 Blocking 동작이다. 특히 TCP 통신에서는 네트워크 상태 같은 요인에 의해 연결이 크게 지연될 수 있고, accept를 사용했을 때 Blocking 방식이라 상대방의 요청이 없으면 영원히 기다리는 일이 벌어질 수 있다. 이것이 멀티스레드나 비동기, 이벤트 기반 처리 등이 필요한 이유다.

3. 클라이언트의 연결 요청이 도착하면 accept 메서드는 새로운 소켓 객체(Socket 클래스의 객체) 을 생성하고, 이 소켓을 통해 클라이언트와 통신하게 된다.

##### Listen & Accept Test

현재 Listen과 Accept만 구현한 상태에서 테스트를 해보자.

```<java>
/*
 * This Java source file was generated by the Gradle 'init' task.
 */
package com.gyo.http.server;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

public class App {

    public static void main(String[] args) throws IOException {
        App app = new App();
        app.run();
    }

    private void run() throws IOException {
        // 1. Listen
        ServerSocket listener = new ServerSocket(8080);

        System.out.println("Listen!");

        // 2. Accept
        Socket socket = listener.accept();

        System.out.println("Accept!");

        // 3. Request -> 처리 -> Response

        // 4. Close
    }
}
```

어플리케이션 실행 하면 Listen!을 출력하고 Blocking이기 때문에 기다리고 있다. 그리고 터미널에서 curl 명령어로 요청을 보내보자.

```<bash>
curl localhost:8080
```

아래와 같은 결과를 확인할 수 있다. 이건 accept후 어플리케이션이 정상적으로 종료됬다는 말이다. 연결하고 서버에서 바로 연결을 끊어버렸다. 어플리케이션 콘솔에서는 Accept!를 확인할 수 있다.

```<bash>
curl: (56) Recv failure: Connection reset by peer
```

#### 3. Request

서버 입장에서 요청을 받으면 이를 읽으면 된다. 어플리케이션에서 읽어서 콘솔에 출력하는 과정을 진행한다.

```<java>
/*
 * This Java source file was generated by the Gradle 'init' task.
 */
package com.gyo.http.server;

import java.io.IOException;
import java.io.InputStreamReader;
import java.io.Reader;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.CharBuffer;

public class App {

    public static void main(String[] args) throws IOException {
        App app = new App();
        app.run();
    }

    private void run() throws IOException {
        // 1. Listen
        ServerSocket listener = new ServerSocket(8080);

        System.out.println("Listen!");

        // 2. Accept
        Socket socket = listener.accept();

        System.out.println("Accept!");

        // 3. Request -> 처리 -> Response
        Reader reader = new InputStreamReader(socket.getInputStream());
        CharBuffer charBuffer = CharBuffer.allocate(1_000_000);
        reader.read(charBuffer);
        charBuffer.flip();  // 읽고 flip 필요
        // 읽은 요청을 출력
        System.out.println(charBuffer);

        // 4. Close
    }
}
```

이제 어플리케이션 실행 후 curl 요청을 보내면 어플리케이션 콘솔에서 curl 요청 내용을 확인할 수 있다.

```<bash>
>> 입력
curl localhost:8080
>> 출력
curl: (52) Empty reply from server
```

```<bash>
>> 어플리케이션 콘솔

Listen!
Accept!
GET / HTTP/1.1
Host: localhost:8080
User-Agent: curl/8.1.1
Accept: */*
```

#### 4. Response & Close

curl에서 Empty reply from server라고 한다. 즉 응답이 없다는 말이다. 서버에서 요청에 따라 처리 후 응답하는 과정을 실습한다. 응답 HTTP 메세지를 만들어서 전송해주면 된다.  
Content-Type, Content-Length를 넣어주면 좋다.

```<java>
/*
 * This Java source file was generated by the Gradle 'init' task.
 */
package com.gyo.http.server;

import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.Reader;
import java.io.Writer;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.CharBuffer;

public class App {

    public static void main(String[] args) throws IOException {
        App app = new App();
        app.run();
    }

    private void run() throws IOException {
        // 1. Listen
        ServerSocket listener = new ServerSocket(8080);

        System.out.println("Listen!");

        // 2. Accept
        Socket socket = listener.accept();

        System.out.println("Accept!");

        // 3. Request
        Reader reader = new InputStreamReader(socket.getInputStream());
        CharBuffer charBuffer = CharBuffer.allocate(1_000_000);
        reader.read(charBuffer);
        charBuffer.flip();  // 읽고 flip 필요
        // 읽은 요청을 출력
        System.out.println(charBuffer);

        // 4. Response

        String body = "Hello, world!";
        byte[] bytes = body.getBytes();
        String message = "HTTP/1.1 200 OK\n" +
                "Content-Type: text/html; charset=UTF-8\n" +
                "Content-Length: " + bytes.length + "\n" +
                "\n" +
                body;

        Writer writer = new OutputStreamWriter(socket.getOutputStream());
        writer.write(message);
        writer.flush();

        // 5. Close
        socket.close();
        listener.close();
    }
}

```

curl 요청을 보내면 body 내용이 잘 나온다.

```<bash>
>> 입력
curl localhost:8080
>> 출력
Hello, world!%
```

참고로 HTTP 메세지를 작성할 때 Content-Type과 Content-Length를 넣어야하는 이유는 아래의 형태로 썼을 때 실수할 여지가 있기 때문이다.

```<java>
String message = "" +
	"HTTP/1.1 200 OK\n" +
	"\n" +
	"Hello, world!\n";
```

위 코드를 보면 메세지의 마지막줄에 newline을 넣은 것을 확인할 수 있는데 newline을 넣지 않으면 클라이언트에 따라서 이상하게 메세지가 전달 될 수 있다.