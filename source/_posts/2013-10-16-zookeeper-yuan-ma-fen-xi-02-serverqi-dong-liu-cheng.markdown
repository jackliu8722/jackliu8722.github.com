---
layout: post
title: "Zookeeper-源码分析(二)——Server启动流程"
date: 2013-10-16 08:23
comments: true
categories: 分布式{distributed}
---
Zookeeper中，server的启动入口为类org.apache.zookeeper.server.quorum.QuorumPeerMain的main方法，由zkServer.sh脚本调用。其main方法的内容如下：

{% codeblock lang:java%}
     
    public static void main(String[] args) {

        QuorumPeerMain main = new QuorumPeerMain();
        try {
            main.initializeAndRun(args);
        } catch (IllegalArgumentException e) {
            LOG.error("Invalid arguments, exiting abnormally", e);
            LOG.info(USAGE);
            System.err.println(USAGE);
            System.exit(2);
        } catch (ConfigException e) {
            LOG.error("Invalid config, exiting abnormally", e);
            System.err.println("Invalid config, exiting abnormally");
            System.exit(2);
        } catch (Exception e) {
            LOG.error("Unexpected exception, exiting abnormally", e);
            System.exit(1);
        }
        LOG.info("Exiting normally");
        System.exit(0);
    }
{% endcodeblock%}

在main方法，调用了类QuorumPeerMain的initializeAndRun方法做后续的处理，其源码如下：

{% codeblock lang:java%}
     
    protected void initializeAndRun(String[] args)
        throws ConfigException, IOException
    {
        //创建配置类
        QuorumPeerConfig config = new QuorumPeerConfig();
        //如果只有一个参数，则调用配置类的parse方法解析配置文件
        if (args.length == 1) {
            config.parse(args[0]);
        }

        // 启动并调度清理任务，定时清理数据目录下的数据
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();
        //运行quorum模式
        if (args.length == 1 && config.servers.size() > 0) {
            runFromConfig(config);
        } else {
            //无配置文件，或者服务的数量为0，运行standalone模式
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            ZooKeeperServerMain.main(args);
        }
    }
{% endcodeblock%}

在initializedAndRun方法中，zookeeper首先创建QuorumPeerConfig类解析配置文件，然后根据配置是否存在和server的数量选择运行的模式。如果无配置文件或者server的数量为1(即只有当前server)，则运行standalone模式，即代码中执行ZookeeperServerMain.main(args),否则以quorum模式运行，即以QuorumPeerConfig的实例为参数运行runFromConfig函数，其代码如下：

{% codeblock lang:java%}
     
    public void runFromConfig(QuorumPeerConfig config) throws IOException {
      try {
          ManagedUtil.registerLog4jMBeans();
      } catch (JMException e) {
          LOG.warn("Unable to register log4j JMX control", e);
      }

      LOG.info("Starting quorum peer");
      try {
          ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
          //配置是客户端连接Zookeeper服务器的端口，以及最大连接数
          cnxnFactory.configure(config.getClientPortAddress(),
                                config.getMaxClientCnxns());
          //创建管理quorum协议的QuorumPeer类
          quorumPeer = new QuorumPeer();
          //设置客户端连接Zookeeper服务器的端口，Zookeeper会监听这个端口，接受客户端的访问请求
          quorumPeer.setClientPortAddress(config.getClientPortAddress());
          quorumPeer.setTxnFactory(new FileTxnSnapLog(
                      new File(config.getDataLogDir()),
                      new File(config.getDataDir())));
          //设置集群中的其它zookeeper
          quorumPeer.setQuorumPeers(config.getServers());
          //设置选举算法
          quorumPeer.setElectionType(config.getElectionAlg());
          //设置server ID
          quorumPeer.setMyid(config.getServerId());
          //这个时间是作为Zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
          quorumPeer.setTickTime(config.getTickTime());
          quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
          quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
          //initLimit配置项是用来配置 Zookeeper 接受客户端（这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过 10 个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒
          quorumPeer.setInitLimit(config.getInitLimit());
          //syncLimit配置项标识 Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 2*2000=4 秒
          quorumPeer.setSyncLimit(config.getSyncLimit());
          quorumPeer.setQuorumVerifier(config.getQuorumVerifier());
          quorumPeer.setCnxnFactory(cnxnFactory);
          quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
          quorumPeer.setLearnerType(config.getPeerType());
          //实现了Runable接口的start方法，启动线程。
          quorumPeer.start();
          quorumPeer.join();
      } catch (InterruptedException e) {
          // warn, but generally this is ok
          LOG.warn("Quorum Peer interrupted", e);
      }
    }
{% endcodeblock%}

runFromConfig的执行流程中，主要创建了QuorumPeer对象，该类定义了Server类型。首先根据配置对象中的配置信息设置相关的配置，然后调用start函数启动服务，QuorumPeer类将在后面的文章中进行详细地分析。
