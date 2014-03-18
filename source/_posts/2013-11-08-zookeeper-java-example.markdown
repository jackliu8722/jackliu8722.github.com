---
layout: post
title: "Zookeeper之java实例"
date: 2013-11-08 23:45
comments: true
categories: 分布式{distributed} Java{java}
---
#####注：本文翻译自zookeeper官方文档，原文请点击[这里](http://zookeeper.apache.org/doc/r3.4.5/javaExample.html)。

##一个简单的Watch客户端
为了介绍Zookeeper的Java API，我们开发一个非常简单的Watch客户端。该Zookeeper客户观察Zookeeper的节点变化，并通过启动或者终止程序进行响应。

####要求
该客户端会达到以下4点要求：

 *  需要以下参数：Zookeeper服务的地址，被观察节点的名称和和个可执行的对象。
 *  获取与节点相关的数据并启动可行对象。
 *  如果节点发生变化，客户端重新获取节点内容并启动可执行对象。
 *  如果被观察的节点消息，客户端将会终止可招待对象。

##程序设计
按照惯例，Zookeeper应用程序有两部分组成，一部分管理网络连接，另一部分用于监听数据。在这个应用当中，一个称为Executor的类保持与Zookeeper服务器的连接，而监控类DataMonitor监听Zookeeper文件目录树中的数据变化。此外，类Executor包含了主线程和执行逻辑，负责用户与服务器的交互行为，同时根据Zookeeper节点的状态，关闭或者重启可执行对象。

####Executor类
Executor对象是该示例程序的主容器，它包含Zookeeper对象和DataMonitor对象。

{% codeblock lang:java%}
	
	//from the Executor class...
	
	public static void main(String[] args){
		if (args.length < 4){
			System.err.println("USAGE: Executor hostPort znode filename  paogran [args ...]");
			System.exit(2);
		}
		String hostPort = args[0];
		String znode = args[1]
		String filename = args[2]
		String exec[] = new String[args.length - 3];
		System.arraycopy(args,3,exec,exec.length);
		try{
			new Executor(hostPort,znode,filename,exec).run();
		}catch(Exception e){
			e.printStackTrace();
		}
	}
	
	public Executor(String hostPort,String znode,String filename, String exec[]) throws KeeperException, IOException{
		this.filename = filename;
		this.exec = exec;
		zk = new ZooKeeper(hostPort,3000,this);
		dm = new DataMonitor(zk,anode,null,this);
	}
	
	public void run(){
		try{
			synchronized (this){
				while(!dm.dead){
					wait();
				}
			}
		}catch(InterruptedException e){
		}
	}
{% endcodeblock %}

回想一下，类Execotur的工作是启动和停止可执行对象，从而响应Zookeeper发送的事件，而该对象的名称是通过命令行传递的参数进行设置的。正如在上面看到的代码中，Executor对象将自身的引用作为ZooKeeper构造函数的参数，代表一个Watcher对象，同时它还通过将自身的引用作为Monitor构造函数的参数，代表DataMonitorListener类的一个对象。每个Executor的定义中，都实现了这丙个接口：

{%codeblock lang:java%}

	public class Executor implements Watcher,Runnable, Monitor.DataMonitorListener{
	...
{%endcodeblock%}

接口Watcher由ZooKeeper的java API定义，ZooKeeper通过该接口将事件传给客户端容器，它只支持一个process方法，它将处理客户端感兴趣的事件，比如ZooKeeper连接的状态或者ZooKeeper会话。在这个例子当中，Executor只是简单的将这些事件下发给DataMonitor对象，由DataMonitor对象决定做什么，这样做的目的仅仅为了方便说明Executor对象或者类Executor这样的对象持有与ZooKeeper的连接，但是它会将事件委派给其它对象进行处理，同时在发起观察事件的处理上选择该方式做为默认的通道（详见后面部分）。

{%codeblock lang:java%}
	
	public void process(WatchedEvent event){
		dm.process(event);
	}
{%endcodeblock%}

另一方面，接口DataMonitorListener不是ZooKeeper API的一部分，它完全是为这个示例程序专门自定义的接口，对象DataMonitro使用它与容器通信，也就是Executor对象。DataMonitorListener接口看起来如下：

{%codeblock lang:java%}

	public interface DataMonitorListener {
    	/**
    	* The existence status of the node has changed.
    	*/
    	void exists(byte data[]);

    	/**
    	* The ZooKeeper session is no longer valid.
    	* 
    	* @param rc
    	* the ZooKeeper reason code
    	*/
    	void closing(int rc);
	}
{%endcodeblock%}

该接口被定义在DataMonitor类中，并且由Executor类实现。当Executor.exists方法被调用时，Executor对象按要求决定是启动还是关闭可执行对象。如果znode节点不存在了，则需要关闭可执行对象。

当Executor.closing方法被调用时，Executor对象决定是否关闭自身以响应ZooKeeper连接的永久消息。

读者可能已经猜到了，为响应ZooKeeper状态变化的事件，DataMonitor是调用这些方法的对象。

类Executor实现DataMonitorListener接口的exists和closing方法的代码如下：

{%codeblock lang:java%}

	public void exists( byte [] data){
		if(data == null){
			if(child != null){
				System.out.println("Killing process");
				child.destory();
				try{
					child.waitFor();
				}catch(InterruptedException e){
				}
			}
			child = null;
		}else{
			if (child != null){
				System.out.println("Stopping child");
				child.destory();
				try{
					child.waitFor();
				}catch(InterruptedException e){
					e.printStackTrace();
				}
			}
			try{
				FileOutputStream fos = new FileOutputStream(filename);
				fos.write(data);
				fos.close();
			}catch(IOException e){
				e.printStackTrace();
			}
			
			try{
				System.out.println("Starting child");
				child = Runtime.getRuntime().exec(exec);
				new StreamWriter(child.getInputStream(),System.out);
				new StreamWriter(child.getErrorStream(),System.err);
			}catch(IOException e){
				e.printStackTrace();
			}
		}
	}
	
	public void closing(int rc){
		synchronized(this){
			notifyAll();
		}
	}
{%endcodeblock%}

####DataMonitor类
类DataMonitor实现了ZooKeeper逻辑，它大部分是异步的和事件驱动的，其构造函数如下：

{%codeblock lang:java%}
	
	public DataMonitor(ZooKeeper zk,String anode,Watcher chainedWatcher,DataMonitorListener listener){
		this.zk = zk;
		this.znode = anode;
		this.chainedWatcher = chainedWatcher;
		this.listener = listener;
		// Get things started by checking if the node exists. We are going
    	// to be completely event driven
    	zk.exists(znode, true, this, null);
	}
{%endcodeblock%}

通过调用ZooKpeeper对象上的exists方法检查anode节点是否存在，并通过对自身的引用(this)设置一个监听对象作为回调函数。当事件被触发，将会进行真正的执行过程。

######注意：不要混淆完成回调与监听回调。完成回调方法ZooKeeper.exists恰好是StatCallbask.processResult方法，并且由类DataMonitor实现。当在ZooKeeper服务器上异步设置监听操作完成之后调用完成回调方法。另一方面，因为Executor已注册为ZooKeeper对象的监听器，因此，通过给Executor发送一个事件以触发监听回调方法。顺便说一句，读者可能会注意到，DataMonitor对象也可以将自身注册为事件监听器，这是ZooKeeper 3.3.0的新特性（支持多外监听器）。然后，在这个例子中，DataMonitor并没有注册为监听器。

当服务器完成ZooKeeper.esists操作之后，ZooKeeper API将调用客户端的完成回调函数：

{%codeblock lang:java%}

	public void processResult(int rc, String path, Object ctx, Stat stat){
		boolean esists;
		switch(tr){
		case Code.Ok:
			exists = true;
			break;
		case Code.NoNode:
			exists = false;
			break;
		case Code.SessionExpired:
		case Code.NoAuth:
			dead = true;
			listener.closing(rc);
			return;
		default:
			zk.exists(anode,true,this,null);
			return;
		}
		
		byte b[] = null;
		if(exists){
			try{
				b = zk.getData(znode,false,null);
			}catch(KeeperException e){
				// We don't need to worry about recovering now. The watch
            	// callbacks will kick off any exception handling
            	e.printStackTrace();
			}catch(interruptedException e){
				return ;
			}
		}
		if((b == null && b != prevData]]) || (b != null && !Arrays.equals(prevData,b))){
			listener.exists(b);
			prevData = b;
		}
	}
{%endcodeblock%}

该代码首先通过返回的状态代码检查节点是否存在、是否发生致命错误或者可恢复的错误。如果节点存在，则从节点上获取数据，并且如果节点的状态发生变化，将调用Executor对象的回调函数exists。注意，在调用方法getData时，不做任何异常处理，因为它有处理错误的监听器。如果在调用ZooKeeper.getData方法之前删除了节点，监听事件将触发一个回调。如果遇到了一个传输错误，当连接传回时将触发一个监听事件。

{% codeblock lang:java%}

	 public void process(WatchedEvent event) {
        String path = event.getPath();
        if (event.getType() == Event.EventType.None) {
            // We are are being told that the state of the
            // connection has changed
            switch (event.getState()) {
            case SyncConnected:
                // In this particular example we don't need to do anything
                // here - watches are automatically re-registered with 
                // server and any watches triggered while the client was 
                // disconnected will be delivered (in order of course)
                break;
            case Expired:
                // It's all over
                dead = true;
                listener.closing(KeeperException.Code.SessionExpired);
                break;
            }
        } else {
            if (path != null && path.equals(znode)) {
                // Something has changed on the node, let's find out
                zk.exists(znode, true, this, null);
            }
        }
        if (chainedWatcher != null) {
            chainedWatcher.process(event);
        }
    }
{% endcodeblock%}

如果客户端库在会话过期（超时事件）之前与ZooKeeper服务器重新建立了通信通道(SyncConnected事件)，则所有的会话事件将自动地与服务器重建连接（自动复位是ZooKeeper 3.0.0的特性）。更多关于监听的介绍可以ZooKeeper的编程指南中找到。在这个函数的后面部分中，当DataMonitor收到一个节点事件时，将会调用ZooKeeper的exists函数，以处理变化。

####所有的源码

{% codeblock Executor.java lang:java%}
	
	/**
 	* A simple example program to use DataMonitor to start and
 	* stop executables based on a znode. The program watches the
 	* specified znode and saves the data that corresponds to the
 	* znode in the filesystem. It also starts the specified program
 	* with the specified arguments when the znode exists and kills
 	* the program if the znode goes away.
 	*/
	import java.io.FileOutputStream;
	import java.io.IOException;
	import java.io.InputStream;
	import java.io.OutputStream;

	import org.apache.zookeeper.KeeperException;
	import org.apache.zookeeper.WatchedEvent;
	import org.apache.zookeeper.Watcher;
	import org.apache.zookeeper.ZooKeeper;

	public class Executor
    	implements Watcher, Runnable, DataMonitor.DataMonitorListener
	{
    	String znode;

    	DataMonitor dm;

    	ZooKeeper zk;

    	String filename;

    	String exec[];

    	Process child;

    	public Executor(String hostPort, String znode, String filename,
            String exec[]) throws KeeperException, IOException {
        	this.filename = filename;
        	this.exec = exec;
        	zk = new ZooKeeper(hostPort, 3000, this);
        	dm = new DataMonitor(zk, znode, null, this);
    	}

    	/**
    	 * @param args
     	*/
    	public static void main(String[] args) {
        	if (args.length < 4) {
            	System.err
                    .println("USAGE: Executor hostPort znode filename program 	[args ...]");
            	System.exit(2);
        	}
        	String hostPort = args[0];
        	String znode = args[1];
        	String filename = args[2];
        	String exec[] = new String[args.length - 3];
        	System.arraycopy(args, 3, exec, 0, exec.length);
        	try {
            	new Executor(hostPort, znode, filename, exec).run();
        	} catch (Exception e) {
            	e.printStackTrace();
        	}
        }

    	/***************************************************************************
     	* We do process any events ourselves, we just need to forward them on.
     	*
     	* @see 	org.apache.zookeeper.Watcher#process(org.apache.zookeeper.proto.WatcherEvent)
     	*/
    	public void process(WatchedEvent event) {
        	dm.process(event);
    	}

    	public void run() {
        	try {
            	synchronized (this) {
                	while (!dm.dead) {
                   		wait();
                	}
            	}
        	} catch (InterruptedException e) {
        	}
    	}

    	public void closing(int rc) {
        	synchronized (this) {
            	notifyAll();
        	}
    	}

    	static class StreamWriter extends Thread {
        	OutputStream os;

        	InputStream is;

        	StreamWriter(InputStream is, OutputStream os) {
            	this.is = is;
            	this.os = os;
            	start();
        	}

        	public void run() {
            	byte b[] = new byte[80];
            	int rc;
            	try {
                	while ((rc = is.read(b)) > 0) {
                   		os.write(b, 0, rc);
                	}
            	} catch (IOException e) {
            	}

        	}
    	}

    	public void exists(byte[] data) {
        	if (data == null) {
            	if (child != null) {
                	System.out.println("Killing process");
                	child.destroy();
                	try {
                   		child.waitFor();
                	} catch (InterruptedException e) {
                	}
            	}
            	child = null;
        	} else {
            	if (child != null) {
                	System.out.println("Stopping child");
                	child.destroy();
                	try {
                   		child.waitFor();
                	} catch (InterruptedException e) {
                   		e.printStackTrace();
                	}
            	}
            	try {
                	FileOutputStream fos = new FileOutputStream(filename);
                	fos.write(data);
                	fos.close();
            	} catch (IOException e) {
                	e.printStackTrace();
            	}
            	try {
                	System.out.println("Starting child");
                	child = Runtime.getRuntime().exec(exec);
                	new StreamWriter(child.getInputStream(), System.out);
                	new StreamWriter(child.getErrorStream(), System.err);
            	} catch (IOException e) {
                	e.printStackTrace();
            	}
        	}
    	}
	}
{%endcodeblock%}

{%codeblock DataMonitor.java lang:java%}

	/**
     * A simple class that monitors the data and existence of a ZooKeeper
     * node. It uses asynchronous ZooKeeper APIs.
    */
	import java.util.Arrays;

	import org.apache.zookeeper.KeeperException;
	import org.apache.zookeeper.WatchedEvent;
	import org.apache.zookeeper.Watcher;	
	import org.apache.zookeeper.ZooKeeper;
	import org.apache.zookeeper.AsyncCallback.StatCallback;
	import org.apache.zookeeper.KeeperException.Code;
	import org.apache.zookeeper.data.Stat;

	public class DataMonitor implements Watcher, StatCallback {

    	ZooKeeper zk;

    	String znode;

    	Watcher chainedWatcher;

    	boolean dead;

    	DataMonitorListener listener;

    	byte prevData[];

    	public DataMonitor(ZooKeeper zk, String znode, Watcher chainedWatcher,
            DataMonitorListener listener) {
        	this.zk = zk;
        	this.znode = znode;
        	this.chainedWatcher = chainedWatcher;
        	this.listener = listener;
        	// Get things started by checking if the node exists. We are going
        	// to be completely event driven
        	zk.exists(znode, true, this, null);
    	}

    	/**
     	* Other classes use the DataMonitor by implementing this method
     	*/
    	public interface DataMonitorListener {
        	/**
         	* The existence status of the node has changed.
         	*/
        	void exists(byte data[]);

        	/**
         	* The ZooKeeper session is no longer valid.
         	*
         	* @param rc
         	*                the ZooKeeper reason code
         	*/
        	void closing(int rc);
    	}

    	public void process(WatchedEvent event) {
        	String path = event.getPath();
        	if (event.getType() == Event.EventType.None) {
            	// We are are being told that the state of the
            	// connection has changed
            	switch (event.getState()) {
            	case SyncConnected:
                	// In this particular example we don't need to do anything
                	// here - watches are automatically re-registered with 
                	// server and any watches triggered while the client was 
                	// disconnected will be delivered (in order of course)
                	break;
            	case Expired:
                	// It's all over
                	dead = true;
                	listener.closing(KeeperException.Code.SessionExpired);
                	break;
            	}
        	} else {
            	if (path != null && path.equals(znode)) {
                	// Something has changed on the node, let's find out
                	zk.exists(znode, true, this, null);
            	}
        	}
        	if (chainedWatcher != null) {
            	chainedWatcher.process(event);
        	}
    	}
    	
    	public void processResult(int rc, String path, Object ctx, Stat stat) {
        	boolean exists;
        	switch (rc) {
        	case Code.Ok:
            	exists = true;
            	break;
        	case Code.NoNode:
            	exists = false;
            	break;
        	case Code.SessionExpired:
        	case Code.NoAuth:
            	dead = true;
            	listener.closing(rc);
            	return;
        	default:
            	// Retry errors
            	zk.exists(znode, true, this, null);
            	return;
        	}

        	byte b[] = null;
        	if (exists) {
            	try {
                	b = zk.getData(znode, false, null);
            	} catch (KeeperException e) {
                	// We don't need to worry about recovering now. The watch
                	// callbacks will kick off any exception handling
                	e.printStackTrace();
            	} catch (InterruptedException e) {
                	return;
            	}
        	}
        	if ((b == null && b != prevData)
            	    || (b != null && !Arrays.equals(prevData, b))) {
            	listener.exists(b);
            	prevData = b;
        	}
    	}
 	}
{% endcodeblock%}