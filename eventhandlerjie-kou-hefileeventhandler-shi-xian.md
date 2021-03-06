# EventHandler接口和FileEventHandler实现

首先重构代码，让事件监听和事件处理分离开，各自责任更加独立。否则想将Echo功能替换为返回静态文件，又需要到处改代码。将责任分开后，只需要传入不同的事件处理器，即可实现不同效果。

增加EventHandler接口专门进行事件处理，SocketEventListener类中事件处理抽取到专门的EchoEventHandler实现中。

提前出AbstractEventListener类，规定了事件处理的模板

```java
public abstract class AbstractEventListener<T> implements EventListener<T> {
    /**
     * 事件处理流程模板方法
     * @param event 事件对象
     * @throws EventException
     */
    @Override
    public void onEvent(T event) throws EventException {
        EventHandler<T> eventHandler = getEventHandler(event);
        eventHandler.handle(event);
    }

    /**
     * 返回事件处理器
     * @param event
     * @return
     */
    protected abstract EventHandler<T> getEventHandler(T event);
}
```

 SocketEventListener重构为通过构造器传入事件处理器

```java
public class SocketEventListener extends AbstractEventListener<Socket> {

    private final EventHandler<Socket> eventHandler;

    public SocketEventListener(EventHandler<Socket> eventHandler) {
        this.eventHandler = eventHandler;
    }

    @Override
    protected EventHandler<Socket> getEventHandler(Socket event) {
        return eventHandler;
    }
}
```

EchoEventHandler实现Echo

```java
public class EchoEventHandler extends AbstractEventHandler<Socket> {

    @Override
    protected void doHandle(Socket socket) {
        InputStream inputstream = null;
        OutputStream outputStream = null;
        try {
            inputstream = socket.getInputStream();
            outputStream = socket.getOutputStream();
            Scanner scanner = new Scanner(inputstream);
            PrintWriter printWriter = new PrintWriter(outputStream);
            printWriter.append("Server connected.Welcome to echo.\n");
            printWriter.flush();
            while (scanner.hasNextLine()) {
                String line = scanner.nextLine();
                if (line.equals("stop")) {
                    printWriter.append("bye bye.\n");
                    printWriter.flush();
                    break;
                } else {
                    printWriter.append(line);
                    printWriter.append("\n");
                    printWriter.flush();
                }
            }
        } catch (IOException e) {
            throw new HandlerException(e);
        } finally {
            IoUtils.closeQuietly(inputstream);
            IoUtils.closeQuietly(outputStream);
        }
    }

}
```

再次将对具体实现的依赖限制到Factory中

```java
public class ServerFactory {
    /**
     * 返回Server实例
     *
     * @return
     */
    public static Server getServer(ServerConfig serverConfig) {
        List<Connector> connectorList = new ArrayList<>();
        //传入Echo事件处理器
        SocketEventListener socketEventListener = new SocketEventListener(new EchoEventHandler());
        ConnectorFactory connectorFactory =
                new SocketConnectorFactory(new SocketConnectorConfig(serverConfig.getPort()), socketEventListener);
        connectorList.add(connectorFactory.getConnector());
        return new SimpleServer(serverConfig, connectorList);
    }
}
```

执行单元测试，一切正常。运行Server，用telnet进行echo，也是正常的。

现在添加返回静态文件功能。功能大致如下：

1. 服务器使用user.dir作为根目录。
2. 控制台输入文件路径，如果文件是目录，则打印目录中的文件列表；如果文件不是目录，且可读，则返回文件内容;如果不满足前面两种场景，返回文件找不到

新增FileEventHandler

```java
public class FileEventHandler extends AbstractEventHandler<Socket> {

    private final String docBase;

    public FileEventHandler(String docBase) {
        this.docBase = docBase;
    }

    @Override
    protected void doHandle(Socket socket) {
        getFile(socket);
    }

    /**
     * 返回文件
     *
     * @param socket
     */
    private void getFile(Socket socket) {
        InputStream inputstream = null;
        OutputStream outputStream = null;
        try {
            inputstream = socket.getInputStream();
            outputStream = socket.getOutputStream();
            Scanner scanner = new Scanner(inputstream, "UTF-8");
            PrintWriter printWriter = new PrintWriter(outputStream);
            printWriter.append("Server connected.Welcome to File Server.\n");
            printWriter.flush();
            while (scanner.hasNextLine()) {
                String line = scanner.nextLine();
                if (line.equals("stop")) {
                    printWriter.append("bye bye.\n");
                    printWriter.flush();
                    break;
                } else {
                    Path filePath = Paths.get(this.docBase, line);
                    //如果是目录，就打印文件列表
                    if (Files.isDirectory(filePath)) {
                        printWriter.append("目录 ").append(filePath.toString())
                                .append(" 下有文件：").append("\n");
                        Files.list(filePath).forEach(fileName -> {
                            printWriter.append(fileName.getFileName().toString())
                                    .append("\n").flush();
                        });
                        //如果文件可读，就打印文件内容
                    } else if (Files.isReadable(filePath)) {
                        printWriter.append("File ").append(filePath.toString())
                                .append(" 的内容是：").append("\n").flush();
                        Files.copy(filePath, outputStream);
                        printWriter.append("\n");
                        //其他情况返回文件找不到
                    } else {
                        printWriter.append("File ").append(filePath.toString())
                                .append(" is not found.").append("\n").flush();
                    }

                }
            }
        } catch (IOException e) {
            throw new HandlerException(e);
        } finally {
            IoUtils.closeQuietly(inputstream);
            IoUtils.closeQuietly(outputStream);
        }
    }
}
```

修改ServerFactory，使用FileEventHandler

```java
public class ServerFactory {
    /**
     * 返回Server实例
     *
     * @return
     */
    public static Server getServer(ServerConfig serverConfig) {
        List<Connector> connectorList = new ArrayList<>();
        //使用FileEventHandler
        SocketEventListener socketEventListener =
                new SocketEventListener(new FileEventHandler(System.getProperty("user.dir")));
        ConnectorFactory connectorFactory =
                new SocketConnectorFactory(new SocketConnectorConfig(serverConfig.getPort()), socketEventListener);
        connectorList.add(connectorFactory.getConnector());
        return new SimpleServer(serverConfig, connectorList);
    }
}
```

运行BootStrap启动Server进行验证：

![](/assets/file-server.jpg)

绿色框：输入回车，返回目录下文件列表。

黄色框：输入R EADME.MD，返回文件内容

蓝色框：输入不存在的文件，返回文件找不到。

完整代码：https://github.com/pkpk1234/BeggarServletContainer/tree/step5

分支step5

![](/assets/git-br-step5.jpg)

