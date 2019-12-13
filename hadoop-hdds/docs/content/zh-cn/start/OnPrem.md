---
title: 物理集群上 Ozone 的安装 
weight: 20

---
<!---
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

如果你乐于冒险，你可以在真实的集群上安装 ozone。搭建一个 Ozone 集群需要了解它的各个组件，Ozone 既可以和 HDFS 协同工作，也能够独立运行，但在这两种情况下 ozone 的组件是相同的。

## Ozone 组件 

1. Ozone Manager - 管理 Ozone 命名空间的服务，负责所有对卷、桶和键的操作。
2. Storage Container Manager - Acts as the block manager. Ozone Manager
requests blocks from SCM, to which clients can write data.
3. Datanodes - Ozone 的数据节点代码既可以运行在 HDFS 的数据节点内，也可以独立部署成单独的进程。

## 搭建一个纯 Ozone 集群

* 将 ozone-\<version\> 安装包解压到目标目录，Ozone 的 jar 包需要存在于集群的所有机器上，所以你需要在所有机器上进行此操作。

* Ozone 依赖名为 ```ozone-site.xml``` 的配置文件， 运行下面的命令可以在指定目录生成名为 ```ozone-site.xml``` 的配置文件模板，以便你可以将参数替换为合适的值。

{{< highlight bash >}}
ozone genconf <path>
{{< /highlight >}}

我们来看看生成的文件（ozone-site.xml）中都有哪些参数，以及它们是如何影响 ozone 的。当各个参数都配置了合适的值之后，需要把该文件拷贝到 ```ozone directory/etc/hadoop```。

* **ozone.metadata.dirs** 管理员通过此参数指定元数据的存储位置，通常应该选择最快的磁盘（比如 SSD，如果节点上有的话），OM、SCM 和数据节点会将元数据写入此路径。这是个必需的参数，如果不配置它，Ozone 会启动失败。
 
示例如下：

{{< highlight xml >}}
   <property>
      <name>ozone.metadata.dirs</name>
      <value>/data/disk1/meta</value>
   </property>
{{< /highlight >}}

*  **ozone.scm.names**  Storage container manager(SCM) 是 ozone 使用的分布式块服务，数据节点通过这个参数来连接 SCM 并向 SCM 发送心跳。在 HA 特性完成之前，我们给 ozone.scm.names 配置一台机器的地址即可。
  
  示例如下：
  
  {{< highlight xml >}}
    <property>
        <name>ozone.scm.names</name>
      <value>scm.hadoop.apache.org</value>
      </property>
  {{< /highlight >}}
  
 * **ozone.scm.datanode.id.dir** 每个数据节点会生成一个唯一 ID，叫做 Datanode ID。Datanode ID 会被写入此参数所指定路径下名为 datanode.id 的文件中，如果该路径不存在，数据节点会自动创建。

示例如下：
{{< highlight xml >}}
   <property>
      <name>ozone.scm.datanode.id.dir</name>
      <value>/data/disk1/meta/node</value>
   </property>
{{< /highlight >}}

* **ozone.om.address** OM 服务地址，OzoneClient 和 Ozone 文件系统需要使用此地址。

示例如下：
{{< highlight xml >}}
    <property>
       <name>ozone.om.address</name>
       <value>ozonemanager.hadoop.apache.org</value>
    </property>
{{< /highlight >}}


## Ozone 参数汇总

| Setting                        | Value                        | Comment |
|--------------------------------|------------------------------|------------------------------------------------------------------|
| ozone.metadata.dirs            | 文件路径                | 元数据存储位置                    |
| ozone.scm.names                | SCM 服务地址            | SCM的主机名:端口，或者IP:端口  |
| ozone.scm.block.client.address | SCM 服务地址和端口 | OM 等服务使用                                 |
| ozone.scm.client.address       | SCM 服务地址和端口 | 客户端使用                                        |
| ozone.scm.datanode.address     | SCM 服务地址和端口 | 数据节点使用                            |
| ozone.om.address               | OM 服务地址           | Ozone handler 和 Ozone 文件系统使用             |


## 启动集群

在启动 Ozone 集群之前，我们需要初始化 SCM 和 OM。

{{< highlight bash >}}
ozone scm --init
{{< /highlight >}}
这条命令会使 SCM 创建集群 ID 并初始化它的状态This allows SCM to create the cluster Identity and initialize its state.
```init``` 命令和 Namenode 的 ```format``` 命令类似，只需要执行一次，SCM 就可以command is similar to Namenode format. Init command is executed only once, that allows SCM to create all the required on-disk structures to work correctly.
{{< highlight bash >}}
ozone --daemon start scm
{{< /highlight >}}

Once we know SCM is up and running, we can create an Object Store for our use. This is done by running the following command.

{{< highlight bash >}}
ozone om --init
{{< /highlight >}}


Once Ozone manager is initialized, we are ready to run the name service.

{{< highlight bash >}}
ozone --daemon start om
{{< /highlight >}}

At this point Ozone's name services, the Ozone manager, and the block service  SCM is both running.\
**Please note**: If SCM is not running
```om --init``` command will fail. SCM start will fail if on-disk data structures are missing. So please make sure you have done both ```scm --init``` and ```om --init``` commands.

Now we need to start the data nodes. Please run the following command on each datanode.
{{< highlight bash >}}
ozone --daemon start datanode
{{< /highlight >}}

At this point SCM, Ozone Manager and data nodes are up and running.

***Congratulations!, You have set up a functional ozone cluster.***

## Shortcut

If you want to make your life simpler, you can just run
{{< highlight bash >}}
ozone scm --init
ozone om --init
start-ozone.sh
{{< /highlight >}}

This assumes that you have set up the slaves file correctly and ssh
configuration that allows ssh-ing to all data nodes. This is the same as the
HDFS configuration, so please refer to HDFS documentation on how to set this
up.
