Isolate之间如何通信
-----
```
  loadData() async {
    ReceivePort receivePort = new ReceivePort();
    await Isolate.spawn(dataLoader, receivePort.sendPort);

    // The 'echo' isolate sends it's SendPort as the first message
    SendPort sendPort = await receivePort.first;

    List msg = await sendReceive(sendPort, "https://jsonplaceholder.typicode.com/posts");

    setState(() {
      widgets = msg;
    });
  }

  // the entry point for the isolate
  static dataLoader(SendPort sendPort) async {
    // Open the ReceivePort for incoming messages.
    ReceivePort port = new ReceivePort();

    // Notify any other isolates what port this isolate listens to.
    sendPort.send(port.sendPort);

    await for (var msg in port) {
      String data = msg[0];
      SendPort replyTo = msg[1];

      String dataURL = data;
      http.Response response = await http.get(dataURL);
      // Lots of JSON to parse
      replyTo.send(JSON.decode(response.body));
    }
  }

  Future sendReceive(SendPort port, msg) {
    ReceivePort response = new ReceivePort();
    port.send([msg, response.sendPort]);
    return response.first;
  }
```
##### 说明
```
loadData被声明为一个异步方法，其内部的代码运行于main isolate中。
该方法首先声明了一个用于main isolate从其他isolate接受消息的ReceivePort。
接着通过spawn命名构造方法生成了一个isolate，为了后续描述简单这里姑且叫它x isolate。
该isolate将会以构造时传入的第一个参数dataLoader方法作为运行的入口函数。
即生成x isolate后，在x isolate中会开始执行dataLoader方法。
构造x isolate时传入的第二个参数是通过main isolate中的ReceivePort获得的一个SendPort，
这个SendPort会在dataLoader被执行时传递给它。
在x isolate中可以用该SendPort向main isolate发送消息进行通信。
接下来通过receivePort.first获取x isolate发送过来的消息，这里获取到的其实是一个x isolate的SendPort对象，
在main isolate中可以利用这个SendPort对象向x isolate中发送消息。
接下来调用sendReceive方法并传入刚刚获得的x isolate的SendPort对象和一个字符串作为参数。
最后调用setState方法触发界面更新。
```
```
dataLoader也被声明为一个异步方法，其内部的代码运行于x isolate中。
在构建了x isolate后该方法开始在x isolate中执行，要注意的是dataLoader方法的参数是一个SendPort类型的对象，
这正是前面构造x isolate时传入的第二个参数，也就是说，前面通过Isolate.spawn命名构造方法构造一个isolate时，
传入的第二个参数的用途就是将其传递给第一个参数所表示的入口函数。
在这里该参数表示的是main isolate对应的SendPort，通过它就可以在x isolate中向main isolate发送消息。
在dataLoader方法中首先生成了一个x isolate的ReceivePort对象，
然后就用main isolate对应的SendPort向main isolate发送了一个消息，该消息其实就是x isolate对应的SendPort对象，
所以回过头去看loadData方法中通过receivePort.first获取到的一个SendPort就是这里发送出去的。
在main isolate中接收到这个SendPort后，就可以利用该SendPort向x isolate发送消息了。
接下来dataLoader方法则挂起等待x isolate的ReceivePort接受到消息。
```
```
sendReceive被声明为一个普通方法，该方法运行于main isolate中，它是在loadData中被调用的。
调用sendReceive时传入的第一个参数就是在main isolate中从x isolate接收到的其对应的SendPort对象，
所以在sendReceive方法中利用x isolate对应的这个SendPort对象就可以在main isolate中向x isolate发送消息。
在这里发送的消息是一个数组[msg, response.sendPort]。
消息发送后在dataLoader方法中await挂起的代码就会开始唤醒继续执行，取出传递过来的参数，
于是在x isolate中开始执行网络请求的逻辑。
接着将请求结果再通过main isolate对应的SendPort传递给main isolate。
于是在sendReceive方法中通过response.first获取到x isolate传递过来的网络请求结果。
最终在setState方法中使用网络请求回来的结果更新数据集触发界面更新。
```
