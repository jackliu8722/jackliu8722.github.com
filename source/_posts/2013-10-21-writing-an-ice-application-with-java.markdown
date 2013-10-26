---
layout: post
title: "用Java实现一个ICE应用"
date: 2013-10-21 23:30
comments: true
categories: 分布式{distributed}
---
注意:本文翻译自ice官方文档

本文实现了一个简单的（但是完整的）客户端和服务器。

实现一个ICE应用程序包括以下几个步骤：

   1. 完成slice定义并编译。
   2. 实现一个服务器并编译。
   3. 实现一个客户端并编译。

如果已经实现了服务器，那么只需要写一个客户端，并且不需要写slice定义，只进行编译就行（很明显，在此情况下也不用实现服务器）。

本文描述的应用可以将客户端发送的文本在远程的服务器进行有序打印。为简单起见，我们的打印机会将文本简单地打印到终端，而不是真正的通过打印机打印。这样的目的只是为了显示客户端如何与服务器进行通信。一旦控制线程执行到了服务器应用程序的代码，就可以做任何它喜欢的事（包括将文本发送到真正的打印机上）。如何做到这一点是ICE提供的功能，本文并不涉及。

####Slice定义
编写ICE应用程序的第一步写一个slice文件，其中包含了应用程序使用的接口。对于我们这个小小的打印程序，我们编写以下Slice定义：

{% codeblock%}
     
	module Demo{
		interface Printer{
			void printString(string s);
		}
	} 
{% endcodeblock%}

我们将这个文本文件保存为Printer.ice.

Slice定义中，包含模块Demo,该模块仅仅包含一个称为Printer的接口。对于这个应用程序，接口只提供了一个单一的操作printString.该操作接受一个字符串作为其唯一的输入参数。字符串的文本内容是什么（可能在远程），打印机就打印什么。

####编译Slice文件
创建Java应用程序的第一步是编译通过Slice定义的数据结构，以生成Java代理和框架。在Unix下，可以通过以下方式进行编译：

    $ mkdir generated
    $ slice2java --output-dir generated Printer.ice

选项--output-dir将编译生成的文件输出到指定的目录下。这就避免了工作目录因生成大量的文件而造成混乱的现象。编译命令slice2java将slice定义的文件生成一些Java源文件。我们现在不用关心这些文件中的内容——它们包含了打印接口的代码，而这些接口我们定义在Printer.ice文件中。

####用Java编写并编译服务器
为了实现我们的打印机接口，必须要创建一个servant类。按照惯例，servant类使用其接口名加一个后缀I。因此，我们的servant类称为PrinterI,并且源文件命名为PrinterI.java:
{%codeblock lang:java%}

	public class PrinterI extends Demo._PrinterDisp{
		public void printString(String s,Ice.Current current){
			System.out.println(s);
		}
	}

{%endcodeblock%}

类PrinterI继承自一个称为_PrinterDisp的基类，该基类由slice2java编译器生成的，是一个抽象类，并且包含一个printString方法。该方法接受一个需要打印的字符串和一个类型为Ice.Current的对象（现在我们将忽略参数Ice.Current）.我们实现的printString方法，只是将字符串打印到终端。

服务器的其它代码在源文件Server.java中，其完整的代码如下：
{%codeblock lang:java%}

	public class Server{
		public static void main(String []args){
			int status = 0;
			Ice.Communicator ic = null;
			try{
				ic = Ice.Util.initialize(args);
				Ice.ObjectAdapter adapter = ic.createObjectAdapterWithEndpoints("SimplePrinterAdapter","defalut -p 10000");
				Ice.Object object = new PrinterI();
				adapter.add(object,ic.stringToIdentity("SimplePrinterAdapter"));
				adapter.activate();
				ic.waitForShutdown();
			}catch(Ice.LocalException e){
				e.printStackTrace();
				status = 1;
			}catch(Exception e){
				System.err.println(e.getMessage());
				status = 1;
			}
			if(ic != null){
				try{
					ic.destory();
				}catch(Exception e){
					System.err.println(e.getMessage());
					status = 1;
				}
			}
			System.exit(status);
		}
	}
{%endcodeblock%}

请注意，代码的结构通常如下：
{%codeblock lang:java%}

	public class Server{
		public static void main(String []args){
			int status = 0;
			Ice.Communicator ic = null;
			try{
			
				//在这里实现服务器
			}catch(Ice.LocalException e){
				e.printStakcTrace();
				status = 1;
			}catch(Exception e){
				System.err.println(e.getMessage());
				status = 1;
			}
			if( ic != null){
				//清除
				//
				try{
					ic.destory();
				}catch(Exception e){
					System.err.printlne(e.getMessage());
					status = 1;
				}
			}
			System.exit(status);
		}
	}
{%endcodeblock%}

在main函数的主体部分中，包含了一个捕获异常的try语句，并且在try语句中包含了所有服务器代码，随后有两个catch语句。其中第一个catch捕获所有ICE运行环境可能抛出的异常。该部分代码的目的是不管代码在任何地方遇到了意想不到的ICE运行时异常，退出当前的堆栈回到主函数中。第二个捕获异常的catch语句的目的是当代码某个地方遇到一个致命的错误时，可以简单地抛出一个异常，并携带一个错误信息。同样的，代码退出当前的堆栈回到主函数当中，并打印错误信息，然后向操作系统返回失败的结果。

在代码退出之前，如果通信器(communicator)创建成功，则需要销毁它。这样做的目的是为了正确的终止ICE运行时环境：程序必须调用communicator对象的destory，否则会产生未知的结果。

try代码块中包含的服务器代码如下：
{%codeblock lang:java%}

	ic = Ice.Util.initialize(args);
	Ice.ObjectAdapter = ic.createObjectAdapterWithEndpoints("SimplePrinterAdapter","defalut -p 10000");
	Ice.Object object = new PrinterI();
	adapter.add(object,ic.stringToIdentity("SimplePrinterAdapter"));
	adpater.activate();
	ic.waitForShutdown();
{%endcodeblock%}

服务器代码包含了以下几个步骤：

1. 我们通过Ice.Util.initialize方法初始化ICE运行时环境（由于服务器可能以命令行的方法运行，因此，在调用该方法时传递了参数args，在本例中，服务器不需要命令行参数）。初始化方法返回一个类Ice.Communicator对象的引用，这是ICE运行时环境主要的对象。
2. 通过Communicator对象实例的createObjectAdapterWithEndpoints方法创建一个适配器对象，我们传递了两个字符串参数，第一个参数为SimplePrinterAdpater(适配器的名称)，另外一个参数是字符串default -p 10000,指示了适配器使用默认的协议（TCP/IP）监端口号10000的网络请求。
3. 此时，服务端运行环境初始化完成，然后实例化一个PrinterI对象为我们的打印机接口创建一个服务。
4. 通过调用适配器对象的add方法添加servant到运行时环境中，add方法的第一个参数为我们实例化的servant对象，另外一个参数是一个标识符，表示servat的名称，在这个例子中，即字符串"SimplePrinterAdapter"（如果有我们有多个打印机，每个将有一个不同的名字，或者更确切地说，拥有不同的对象标识）。
5. 接下来，通过调用adapter对象的activate方法激活适配器（适配器最初处于未激活状态，假设有多个servant，并且共享同一个适配器，如果我想要甩的servant未被实例化之后才处理请求，那么适配器最初为未激活是非常有用的）。
6. 最后，调用waitForShutdown挂用调用的线程，直到服务因调用一个方法终止运行时环境或者接收到一个终止。（在本例子中，当我们不再需要它时，可以通过命令行简单地中断服务器。）

需要注意的是，虽然本例中的代码非常少，但这些代码对所有的服务器都适用的。为了方便，可以将这些代码放在一个辅助类中，此后，就不必再修改它（ICE提供了这样的辅助类Ice.Application）。对于本例的应用程序代码，实际上只包住了几行：定义PrinterI类用了7行代码，而实例化PrinterI对象并将其注册到适配器中只用了三行代码。

我们通过以下方式编译服务器代码：

	$ mkdir classes
	$ javac -d classes -classpath classes:$ICE_HOME/lib/Ice.jar \
	  Server.java PrinterI.java generated/Demo/*.java

通过以上方式将应用程序代码和IE编译器生成的代码都进行了编译，其中环境变量ICE_HOME为包含ICE运行时环境的根目录（例如，如果将ICE安装在/opt/ice目录下，则ICE_HOME的值应该设置为/opt/ice）。值得注意的是，对于JAVA的ICE环境中，使用了ant环境对源代码进行控制（ant类似于make，但是对java应用来说比较灵活），你可以看看ant的示例程序，以掌握如何使用该工具。

####编写并编译Java客户端
客户端代码包含在Client.java源文件中，跟服务器代码非常相似，完整的代码如下：
{%codeblock lang:java%}

	public class Client{
		public static void main(String []args){
			int status = 0;
			Ice.Communicator ic = null;
			try{
				ic = Ice.Util.initialize(args);
				Ice.ObjectPrx base = ic.stringToProxy("SimplePrinterAdapter:default -p 10000");
				Demo.PrinterPrx printer = Demo.PrinterPrxHelper.checkedCast(base);
				if(printer == null){
					throw new Error("Invalid proxy");
				}
			}catch (Ice.LocalException e){
				e.printStackTrace();
				status = 1;
			}catch(Exception e){
				System.err.println(e.getMessage());
				status = 1;
			}
			if(ic != null){
				// 清除
				//
				try{
					ic.destory();
				}catch(Exception e){
					System.err.println(e.getMessage());
					status = 1;
				}
			}
			System.exit(status);
		}
	}
{%endcodeblock %}

**注意**：客户端的代码布局总体上跟服务器的布局是一样的，使用一个try catch块来处理错误，在try代码块中执行以下的操作：

1. 跟服务器一样，客户端通过调用Ice.Util.initialize方法初始化ICE运行时环境。
2. 下一步的操作是取得远程打印的代理，通过调用通信器对象上的stringToProxy方法创建这个代理对象，并传字符串参数"SimplePrinterAdapter:default - 10000"。请注意该字符串中包含了对象的标识和服务器所使用的端口（显然，在将对象标识和端口硬编码到应用是不是一个好的主意，但在我们的应用中是可以工作的，当我们讨论IceGrid时，会遇到更加合理的方式）。
3. 方法stringToProxy返回的代码对象的类型是Ice.ObjectProxy,它处在接口和类的父类。但实际上我们只与打印机交互，需要一个打印机接口的代理，而不是一个Object接口的代理。为了做到这一点，我们通过调用PrinterPrxHelper。checkedCast方法向下转型。该方法会向服务器发送检查和转型相关的请求，相当是向服务器询问“这是一个打印机接口的代理吗？”如果是的话，将返回一个Demo.Printer对象的代理，否则，如果是一些其它类型的接口，返回null。
4. 接下来，需要测试下类型向下转型是否成功，如果没有，则抛出一个异常，终止客户端。
5. 现在在我们的地址空间有一个可用的代理对象，并且可以调用printString方法，传递一个具有悠久历史的"Hello World!"字符串作为参数，服务器将会在终端在打印该字符串。

然后像编译服务器代码一样编译客户端代码：

	$ javas -d classes -classpath classes::$ICE_HOME/lib/Ice.jar \
	  Client.java PrinterI.java generated/Demo/*.java
	  
####运行客户端和服务器
为了运行客户端和服务器，我们首先在一个窗口中运行服务器：
	
	$ java Server
	
目前，在服务端我们将看不任何东西，因为服务器只是简单地等待客户端连接它。我们在另外一个窗口中运行客户端：

	$ java Client
	$
	
客户端运行并退出后，并不会输出任何东西。但是，在服务器的窗口中，我们看到打印了字符串“Hello World！”。如果要终止服务器，可以通过命令行中断程序。（在讨论Ice.Application中，我们将看到更简洁的方式来终止服务器。）

如果出现任何未知的错误，客户端将打印错误信息。例如，如果在运行客户端之前未启动服务哭喊，我们将得到类似下面的错误信息：

    Ice.ConnectionRefusedException
       error = 0
		at IceInternal.ConnectRequestHandler.getConnection(ConnectRequestHandler.java:240)
   	    at IceInternal.ConnectRequestHandler.sendRequest(ConnectRequestHandler.java:138)
		at IceInternal.Outgoing.invoke(Outgoing.java:66)
   		at Ice._ObjectDelM.ice_isA(_ObjectDelM.java:30)
   		at Ice.ObjectPrxHelperBase.ice_isA(ObjectPrxHelperBase.java:111)
   		at Ice.ObjectPrxHelperBase.ice_isA(ObjectPrxHelperBase.java:77)
   		at Demo.HelloPrxHelper.checkedCast(HelloPrxHelper.java:228)
   		at Client.run(Client.java:65)
		Caused by: java.net.ConnectException: Connection refused
         ...
        
需要注意的是，服务器和客户端要运行成功，CLASSPATH路径中必须包含ICE类库和类的目录，例如：

	$ export CLASSPATH=$CLASSPATH:./classes:$ICE_HOME/lib/Ice.jar
	

