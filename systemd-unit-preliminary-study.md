title: systemd初探-服务的Unit文件结构
date: 2016-04-25
tags: [systemd,system,linux]
category: systemd
thumbnail: http://7xrkyd.com1.z0.glb.clouddn.com/systemd-unit-preliminary-study/systemd-1.jpg

---

服务的Unit文件可以分为三个配置区段，其中Unit段和Install段是所有Unit文件通用的，用于配置服务（或其他系统资源）的描述、依赖和随系统启动方式，而Service断则是服务类型的Unit文件（后缀为.service)特有的，用于定义服务的具体管理和操作方法。  
下面列出每个配置区段常用的参数有哪些。
1. Unit段
 - **Description**： 一段描述这个Unit文件的文字，通常只是简短的一句话。

 - **Documentation**：指定服务的文档，可以是一个或多个文档的URL路径。
 - **Requires**：依赖的其他Unit列表，列在其中的Unit模块会在这个服务启动的同时被启动。~~并且，如果其中有任意一个服务启动失败，这个服务也会被终止。~~（这一句话我验证时并没有出现这个情况，依赖服务启动失败了也是可以继续启动当前服务的）。依赖服务停止后，当前服务也将被停止。
 - **Wants**：与Requires相似，但只是在被配置的这个Unit启动时，触发启动列出的每个Unit模块，而不去考虑这些模块启动时候是否成功。
 - **After**：与Requires相似，但是在后面列出的所有模块启动完成以后，才会启动当前的服务。与Requires不同的是，After不会因为依赖程序在运行过程中停止运行，导致当前服务也停止。
 - **Before**：与After相反，在启动指定的任意一个模块之前，都会首先确保当前服务已经运行。
 - **BindsTo**：与Requires非常相似，但是一种更强的关联。启动这个服务时会同时启动列出的所有模块，当有模块启动失败时终止当前服务。反之，只要列出的模块全部启动以后，就会自动启动当前服务。并且，这些模块中有任意一个出现意外结束或重启，这个服务会跟着终止或重启。
 -  **PartOf**：这是一个BindsTo作用的子集，仅在列出的任何模块失败或重启时，终止或重启当前服务，而不会随列出模块的启动而启动。
 -  **OnFailure**：当这个模块启动失败时，就自动启动列出的每个模块。
 -  **Conflicts**：与这个模块有冲突的模块，如果列出的模块中有已经在运行的，则会将已启动的冲突模块停止，并启动当前模块；反过来，冲突模块启动时会把当前模块停止。  
 上面的这些配置，除了Description外，其他都可以被添加多次。比如After参数，可以使用多个After参数，也可以在一行内使用空格分割，写多个依赖模块。  
 其他参数可以参考：[systemd.unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)，或`man systemd.unit`。

2. Install段
 这部分配置的目标模块通常是特定运行目标的“.target”文件，用来使得服务在系统启动时自动运行。这个区段可以分成三种启动约束。
 - **WantedBy**：和前面**Wants**作用相似，但此处表示当前模块被依赖。
 - **RequiredBy**：和前面的Requires作用相似，但此处表示当前模块被依赖。
 - **Also**：当这个服务被enable/disable时，将自动enable/disable后面列出的每个模块。

3. Service段
 这个区段是“.service”文件独有的，也是对于服务配置最重要的部分。这部分的配置选项非常多，主要分为服务生命周期控制和服务上下文配置两个方面。下面是比较常用的一些参数。  
 （1）服务生命周期控制相关的参数
  - **Type**：服务的类型，常用的有simple（默认类型）和forking，默认的simple类型可以适用于绝大多数场景，因此一般可以忽略者这个参数的配置。对于服务进程启动后通过fork系统调用创建子进程，然后关闭应用程序本身进程的情况，则应该将Type的值设置为forking；否则Systemd将不会跟踪子进程的行为，而认为服务已经退出。

  - **RemainAfterExit**：指为true或false（也可以写yes或no），默认为false。当配置为true时，Systemd只会负责启动服务进程，之后即便服务进程退出了，Systemd也仍然会认为这个服务还在运行中。这个配置主要是提供给一些并非常驻内存，而是启动注册后立即退出，然后等待消息按需启动的特殊类型服务使用的。
  - **ExecStart**：这个参数是几乎每个“.service”文件都会有的，指定服务启动的主要命令，在每个配置文件中只能使用一次。需要注意的时：包括下面的所有Exec，如果使用了linux命令，需要给出完整的执行程序的路径：  
  ```shell
  ExecStart=python3 /var/lib/test/main.py # 会报`Executable path is not absolute...`错误

  # 正确的应该为  
  ExecStart=/usr/bin/python3 /var/lib/test/main.py
  ```

  - **ExecStartPre**：指定在启动执行ExecStart命令前的准备工作，在同一个配置文件中可以有多个，所有命令会按照文件中书写的顺序依次被执行。
  - **ExecStartPost**：指定在启动执行ExecStart命令后的收尾工作，在同一个配置文件中也可有多个。
  - **TimeoutSec**：快速设置**TimeoutStartSec**和**TimeoutStopSec**参数成指定值。（另外，关于默认时间设定都在systemd配置文件中的**DefaultTimeoutStartSec**、**DefaultTimeoutStopSec**和**DefaultRestartSec**字段进行配置，如果这些字段缺省，**DefaultTimeoutStartSec**和**DefaultTimeoutStopSec**的默认指为90s，**DefaultRestartSec**默认为100ms）
  - **TimeoutStartSec**：启动服务时的等待秒数，如果超出这个时间服务仍然没有执行完所有的启动命令，则Systemd会认为服务自动失败。这一配置对于使用Docker容器托管的应用十分重要。由于Docker第一次运行时可能会需要从网络上下载服务的镜像文件，因此造成比较严重的延时，容易被Systemd误判断为启动失败而杀死。通常，对于这种服务，需要将**TimeoutStartSec**设置为0，关闭超时检测。
  - **ExecStop**：停止服务所需要执行的主要命令，在每个配置文件中只能够有一个。
  - **ExecStopPost**：指定在ExecStop命令执行后的收尾工作，在同一配置文件中可以有多个。
  - **TimeoutStopSec**：停止服务时的等待秒数，如果超过这个时间服务仍然没有停止，Systemd会使用SIGKILL信号强行干掉服务进程。
  - **Restart**：这个值用于指定在什么情况下需要重启服务进程。常用的值有：no、no-success、on-failure、on-abnormal、on-abort和always。默认值为no，即不会自动重启服务。这些不同的值分别表示在哪些情况下，服务会重新启动。  

|退出原因\Restart设置| no |always|on-success|on-failure|on-abnormal|on-abort|
|:------------------:|:--:|:----:|:--------:|:--------:|:---------:|:------:|
|正常退出||o|o||||
|异常退出||o||o||
|启动/停止超时||o||o|o||
|被异常杀死||o||o|o|o|

  - **RestartSec**：如果服务需要被重启，这个参数的值为服务被重启前的等待秒数。默认为100ms。
  - **ExecReload**：重新加载服务所需执行的主要命令。
  （2）服务上下文配置相关的参数
  - **Environment**：为服务添加环境变量，格式直接为Environment=“foo=bar”（看了一下Systemd的手册，这个参数所接受的格式有些奇葩，建议是直接“foo=bar”，取的时候使用${foo}进行获取）
  - **EnvironmentFile**：指定加载一个包含服务所需的环境变量列表的文件，文件中的每一行都是一个环境变量的定义。顺便提一下，建议使用的时候将`=`换成`=-`，如`EnvironmentFile=-/etc/my.env`，和`=`的区别是，使用`=-`时，假如`/etc/my.env`文件不在也不会报错。
  - **Nice**：服务的进程优先级，指越小优先级越高，默认为0，。其中-20为最高优先级，19为最低优先级。
  - **WorkingDirectory**：指定当前服务的工作目录。
  - **RootDirectory**：指定当前服务进程的根目录（/目录）。如果配置了这个参数，服务将无法访问指定目录外的任何文件。
  - **User**：指定运行服务的用户，会影响服务对本地文件系统的访问权限。
  - **Group**：指定运行服务的用户组，会影响服务对本地文件系统的访问权限。
  - **MountFlags**：这个值其实是服务的Mount Namespace的配置，会影响服务进程上下文中挂载点的信息，即服务是否会继承主机上已有的挂载点，以及如果服务运行时执行了挂载或卸载设备的操作，是否会真实地在主机上产生效果。可选值为shared、slave和private，具体作用如下表所示：

|MountFlags|效果|
| ---------- | :----: |
|shared|服务与主机公用一个MountNamespace，继承主机挂载点，且服务挂载或卸载设备会真实地反映到主机上|
|slave|服务使用独立的Mount Namespace，它会继承主机挂载点，但服务对挂载点的操作只在自己的Namespace内生效，不会反映到主机上。|
| private|服务使用独立的Mount Namespace，它在启动时没有任何挂载点，服务对挂载点的操作也不会反映到主机上。|

  - **LimitCPU/LimitSTACK/LimitNOFILE/LimitNPROC**等：限定服务可用的系统资源量，CPU、程序堆栈、文件句柄数量、子进程数量等。


#### 参考：
- 《CoreOS实践之路》
- [systemd.unit — Unit configuration](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)
