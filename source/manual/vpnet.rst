==========================
虚拟私人网络
==========================

虚拟私人网络通过保护公共网络连接来扩展私有网络至公共网络，如互联网。
通过 VPN，您可以创建大型的安全网络，
这些网络可以充当一个私有网络。

.. image:: images/Virtual_Private_Network_overview.png
    :width: 100%

（图片来自 `wikipedia <https://en.wikipedia.org/wiki/File:Virtual_Private_Network_overview.svg>`__）

公司使用这项技术连接分支机构和远程用户
（公路战士）。

OPNsense 支持为分支机构以及远程用户提供 VPN 连接。

通过图形用户界面，
可以轻松设置创建一个包含多个分支机构连接到单一站点的单一安全私有网络。
对于远程用户，可以创建和吊销证书，
且一个易于使用的导出工具使得客户端配置变得非常简单。

OPNsense 提供了广泛的 VPN 技术范围，从现代 SSL VPN 到众所周知的 IPsec，
以及通过插件使用的 WireGuard 和 Zerotier。

.. image:: images/vpn.png
    :width: 30%

--------------------------
IPsec
--------------------------

由于 IPsec 在许多不同的场景中被使用，有时可能会有点复杂，
我们将在本章描述不同的使用案例并提供一些示例。

.................................
一般背景
.................................

IPsec 模块结合了不同的功能，这些功能被分组到各种菜单项中。
从我们项目开始，我们就基于传统的 :code:`ipsec.conf` 格式提供 IPsec 功能，从 23.1 版本开始，
我们正在迁移到 `swantcl.conf <https://docs.strongswan.org/docs/5.9/swanctl/swanctlConf.html>`__。
在迁移现有功能集时，我们得出结论，世界已经发生了很大的变化，
为了提供更好的（API）访问可用的功能集，我们决定计划淘汰自项目开始以来就存在的传统“隧道设置”。
没有设定时间表，只是对使用“隧道设置”菜单项的隧道实施了功能冻结。

从长远来看，我们的主要目标之一是更好地对齐 GUI 组件，使其更真实地反映底层的现实，
因为我们使用 `strongswan <https://www.strongswan.org/>`__，我们的目标是比以前更紧密地遵循他们的术语。

以下功能在菜单中可用（截至 OPNsense 23.1）：

* 连接

  * 新的配置工具，提供对 :code:`swanctl` 配置的连接和池部分的访问

* 隧道设置

  * 传统 IPsec 配置工具

* 移动客户端

  * 提供对 `attr <https://docs.strongswan.org/docs/5.9/plugins/attr.html>`__ 插件和传统隧道的池配置的各种选项的访问

* 预共享密钥

  * 定义用于本地认证的 `secrets <https://docs.strongswan.org/docs/5.9/swanctl/swanctlConf.html#_secrets>`__

* 密钥对

  * 收集用于公钥认证的公钥和私钥。

* 高级设置

  * 定义穿透网络（从内核陷阱中排除），日志选项和一些通用选项

* 状态概览

  * 显示隧道状态

* 租约状态

  * 对于移动客户端，显示配置的各种池的地址租约

* 安全关联数据库

  * 显示安全关联，IPsec 描述两个或更多实体之间关系的基本概念

* 安全策略数据库

  * 安装的安全策略描述哪些流量被允许通过隧道

* 虚拟隧道接口

  * 编辑或创建新的 :code:`if_ipsec(4)` 接口并显示由传统隧道创建的接口

* 日志文件

  * 检查与 IPsec 相关的日志条目



将文档从隧道迁移到连接
-------------------------------------------

自OPNsense早期以来，已经使用隧道设置，
当转向提供的新选项时，一些术语可能会有些混淆。
本段旨在解释隧道部分的一些常见术语及它们在连接中的新位置。
要了解全部更改列表，上游迁移文档是一个有趣的阅读资源。


* 第1阶段 - 一般连接设置，如本地/远程地址和一般协议设置。
  选择使用的认证方式也是其中的一部分，它们可能涉及多轮。
* 第2阶段 - 现在Strongswan将这些称为**子项**，因为这些定义了正在使用的`:code:CHILD_SA`子部分。
  这是您可以定义两端网络的地方。
  当多个段被添加到同一个子项中时，这些被视为一个策略，其中所有的部分都能彼此通信。
* 第1阶段/隧道隔离 - 此选项确保在第2阶段定义的每个网络都被视为其自己的子项（例如，两个第2阶段会变成两个子项）
* 第2阶段/手动SPD条目 - 手动SPD条目，这已经被替换为它自己的菜单选项（安全策略数据库），
  提供更多的灵活性和可见性。

.. 注意::

  使用DNS作为端点是可能的，但与之前有些不同，因为在大多数情况下，
  防火墙尝试解析名称并没有使用Strongswan提供的功能。
  然而，由于`if_ipsec(4)`的限制，目前无法为VTI隧道使用DNS条目，因为这种类型的接口不能以可靠的方式动态更改。

.. 注意::

  当将预共享密钥类型隧道迁移到连接时，也确保在“预共享密钥”模块中添加一个条目。
  如果两端应使用各自的标识符，请填写本地和远程值。
  旧模块在第1阶段页面请求此信息，并将相同的信息写入到秘密中。

.. 提示::

  由于OPNsense也为传统隧道使用新的Strongswan格式，
  因此当从机器下载:code:`swanctl.conf`文件时，手动转换一个隧道相对容易。你可以在:code:`/usr/local/etc/swanctl/swanctl.conf`中找到它，
  其格式几乎与OPNsense中可用的连接GUI相同。


结合传统隧道和连接
-------------------------------------------

结合使用隧道和连接是可能的，但存在一些限制。
由于我们的传统隧道为每个配置的子项（第2阶段）强制一个
:code:`reqid`，新连接子项的自动编号有重叠的风险。为了防止这些重叠，需要在连接子项中设置一个未使用的:code:`reqid`。



.................................
安全策略和路由
.................................

为了通过IPsec隧道传输流量，我们需要一个匹配流量的策略。
默认情况下，添加第2阶段（或子项）策略时，也会安装一个“内核路由”，它在正常路由发生之前捕获流量。

.. 注意::

  如果没有为隧道设置策略，流量将不会被接受，
  如果带有内核路由的策略与本地或本地路由网络重叠，那么流量将不会被有问题的主机接收。

.. 提示::

  在策略中匹配重叠网络（VTI或重叠网络）时，确保在:menuselection:`VPN -> IPsec -> 高级设置`中
  的:code:`Passthrough networks`选项中排除自己的网络段，以防止流量被黑洞吞噬。


.................................
防火墙规则
.................................

当使用传统隧道且在:menuselection:`VPN --> IPsec --> 高级设置`中未选中`:code:Disable Auto-added VPN rules`时，
会为连接到此主机的远程主机自动创建一些防火墙规则。
新的连接功能不提供此项，必须手动指定（WAN）规则以便能够连接到此主机上的IPsec。

IPsec相关的协议和端口包括以下内容：

- 协议：ESP (https://en.wikipedia.org/wiki/IPsec#Encapsulating_Security_Payload)
- 端口：500/UDP (https://en.wikipedia.org/wiki/Internet_Security_Association_and_Key_Management_Protocol)
- 端口：4500/UDP (https://en.wikipedia.org/wiki/NAT_traversal#IPsec)

.. 注意::

  我们不提供自动规则的主要原因之一是，这些规则要么比预期的更开放（允许来自任何地方的IPsec），
  要么因为规则引擎会“猜测”远程端点（如果是fqdn）而过于封闭。


我们防火墙的默认行为是阻止入站流量，这也意味着使用隧道的流量应该被明确允许，
:menuselection:`防火墙 --> 规则 --> IPsec` 菜单项提供了访问IPsec流量策略的途径。


.................................
死对等检测（DPD）
.................................

死对等检测（DPD）是一种通过向远程发送周期性的R-U-THERE消息并期待返回R-U-THERE-ACK消息
来检测一个IKE对等体是否失效的方法，如`RFC 3706 <https://www.ietf.org/rfc/rfc3706.txt>`__所指定。

当一个对等体被认为已失效时，可以指定一个动作，比如关闭CHILD_SA或在新的IKE_SA下重新协商CHILD_SA。

.. 注意::

  默认情况下DPD是禁用的，使用连接时，确保指定一个大于0的
  :code:`DPD delay (s)`来启用此功能。可以在其子项上指定动作。


.................................
实施方案
.................................

在设置IPsec VPN时，有两种主要的场景类型，它们各自有优缺点。

基于策略的
--------------------------

第一种是标准的基于策略的隧道，它通过策略保护隧道的安全，
并安装内核陷阱，
在流量匹配这些策略的情况下通过隧道发送流量。
例如一个本地网络:code:`192.168.1.0/24`向负责:code:`192.168.2.0/24`的远程位置发送流量。这种场景的优点是设置简单，无需配置路由，在此例中:code:`192.168.1.10`联系:code:`192.168.2.10`时，
数据包会无缝地通过隧道转发到远程位置。

当本地流量由于隧道需要网络地址转换而不匹配有关策略时，
只要手动添加策略到安全策略数据库，也是可能的，
这也被称为“IPsec之前的NAT”。


基于路由的（VTI）
--------------------------

基于路由的，也称为VTI，隧道使用称为:code:`if_ipsec(4)`的虚拟接口，可以在:menuselection:`VPN -> IPsec -> 虚拟隧道接口`下找到。
这将通信的两端链接起来用于路由目的，之后应用正常的路由。
在这种情况下，需要禁用“(安装) 策略”复选标记，对于子项（传统隧道配置中的第1阶段）定义。
通常，通信策略（第2阶段或子项）设置为匹配所有流量（对于IPv4是:code:`0.0.0.0/0`，对于IPv6是:code:`::/0`）。

所以与基于策略的选项相同的示例需要为有关目的地配置（静态）路由
（:code:`192.168.1.0/24`需要到:code:`192.168.2.0/24`的路由，反之亦然），
对等连接发生在绑定到隧道接口的另一个子网中的小型网络上（例如:code:`10.0.0.1` <-> :code:`10.0.0.2`）。

这种设置的优点是可以使用标准或高级路由技术来转发隧道周围的流量。

.. 注意::

为了在`:code:if_ipsec(4)`设备上过滤流量，需要设置一些可调参数。
`:code:net.inet.ipsec.filtertunnel`和`:code:net.inet6.ipsec6.filtertunnel`需要被设置为`:code:1`，
而`:code:net.enc.in.ipsec_filter_mask`和`:code:net.enc.out.ipsec_filter_mask`需要被设置为`:code:0`，以允许在设备上应用规则。
缺点是基于策略的隧道(`:code:enc0`)无法再被过滤，因为这改变了从在`:code:enc0`设备上过滤的行为到`:code:if_ipsec(4)`设备的过滤。

.. 警告::

    目前似乎不可能为`:code:if_ipsec(4)`设备添加NAT规则。

.. 警告::

    为了可靠地设置VTI隧道，两端都应使用静态IP地址。
    虽然在传统配置中可以解析主机名，但这永远不会导致稳定的配置，
    因为`:code:if_ipsec(4)`设备在接受流量之前会匹配源和目的地，
    对任何外部变化都没有知识。


.................................
移动用户 / 远程工作者
.................................

IPsec也可用于服务连接到OPNsense的远程工作者，这些远程工作者使用各种客户端，
如Windows、MacOS、iOS和Android。客户端类型通常决定了正在使用的认证方案。

如果应为客户端提供默认设置，这些可以从`:menuselection:VPN -> IPsec -> 移动客户端`配置。
此页面上的池选项（虚拟IPvX地址池）仅由传统隧道配置使用，
使用新的连接模块时，可以为每个连接配置不同的池。

示例部分包含OPNsense中可用的各种选项。
使用OPNsense 23.1起提供的新“连接”选项时，
通常很容易实现不同的`Strongswan示例 <https://docs.strongswan.org/docs/5.9/interop/windowsClients.html>`__，
因为我们在新模块中非常紧密地遵循`swanctl.conf <https://docs.strongswan.org/docs/5.9/swanctl/swanctlConf.html>`__格式。

.................................
示例
.................................

本段落提供了一些常用实施方案的示例。

新版 > 23.1 (`VPN -> IPsec -> 连接`)
------------------------------------------------------------------------------

.. toctree::
   :maxdepth: 2
   :titlesonly:

   how-tos/ipsec-s2s-conn
   how-tos/ipsec-s2s-conn-route
   how-tos/ipsec-s2s-conn-binat
   how-tos/ipsec-swanctl-rw-ikev2-eap-mschapv2


.. 提示::

    我们这边为新模块提供的示例数量有限，但为了获得灵感，
    通常浏览 `Strongswan <https://wiki.strongswan.org/projects/strongswan/wiki/UserDocumentation#Configuration-Examples>`__ 提供的示例是个好主意。
    相当多的swanctl.conf示例在我们的新模块中容易实现，因为我们遵循相同的术语。

传统 (`VPN -> IPsec -> 隧道设置`)
------------------------------------------------------------------------------


.. toctree::
   :maxdepth: 2
   :titlesonly:

   how-tos/ipsec-s2s
   how-tos/ipsec-s2s-binat
   how-tos/ipsec-s2s-route
   how-tos/ipsec-s2s-route-azure
   how-tos/ipsec-rw


我们文档中提供了以下客户端设置示例：

.. toctree::
  :maxdepth: 2
  :titlesonly:

  how-tos/ipsec-rw-android
  how-tos/ipsec-rw-linux
  how-tos/ipsec-rw-w7


.. 注意::

 在基于策略的隧道中使用网络地址转换是不同的，因为安装的IPsec策略应该接受流量以便进行封装。
 `IPSec BINAT`文档将解释如何应用转换。

.................................
调优考虑
.................................

根据工作负载（许多不同的IPsec流或单一流），
启用`:code:ipsec`的多线程加密模式可能会有所帮助，在这种情况下，
加密包被分派到多个处理器（特别是当只使用单个隧道时）。

为此，在`:menuselection:系统 --> 设置 --> 可调参数`中添加或更改以下可调参数：

.. 注意::

    :code:`net.inet.ipsec.async_crypto` = **1**

为了在系统中可用的核心上更好地分配负载，启用`:doc:接收侧扩展 </troubleshooting/performance>`可能会有所帮助。
在这种情况下，需要更改以下可调参数：

.. 注意::

    * :code:`net.isr.bindthreads` = **1**
    * :code:`net.isr.maxthreads` = **-1**   <-- 等于机器中的核心数量
    * :code:`net.inet.rss.enabled` = **1**
    * :code:`net.inet.rss.bits` = **X** <-- 见`:doc:rss </troubleshooting/performance>`文档。


.................................
杂项变量
.................................

路径MTU探测
--------------------------

当尝试强制执行路径MTU探测（`PMTU <https://en.wikipedia.org/wiki/Path_MTU_Discovery>`__）时，
您需要确保数据包带有`:code:DF`位离开网络。内核提供了一个可调参数`:code:net.inet.ipsec.dfbit`，
它提供3个选项，`:code:0`，
清除离开防火墙的数据包上的位（默认），`:code:1`，设置DF位，或`:code:2`从内部头复制该位。


.................................
诊断
.................................

为了跟踪连接的隧道，
您可以使用`:menuselection:VPN -> IPsec -> 状态概览`浏览配置的隧道。

`:menuselection:VPN -> IPsec -> 安全策略数据库`也很实用，可以获得注册策略的洞察，
当使用NAT时，这里也应该可见额外的SPD条目。


在对防火墙进行故障排除时，检查系统上可用的日志很可能是必需的。
在OPNsense的UI中，日志文件通常与它们所属的组件的设置分组。
可以在“日志文件”菜单项中找到日志文件。

.. 提示::

    当尝试调试各种问题时，
    可以通过`:menuselection:VPN -> IPsec -> 高级设置`中的设置配置收集的日志信息量。


.................................
Custom configurations
.................................

In some (rare) cases one might want to add custom configuration options not available in the user interface, for this reason we
do support standard includes.

While the :code:`swanctl.conf` and the legacy :code:`ipsec.conf` configuration files are well suited to define IPsec-related configuration parameters,
it is not useful for other strongSwan applications to read options from these files.
To configure these other components, it is possible to manually append options to our default template, in which case files
may be placed in the directory :code:`/usr/local/etc/strongswan.opnsense.d/` using the file extention :code:`.conf`

IPsec configurations are managed in `swantcl.conf <https://docs.strongswan.org/docs/5.9/swanctl/swanctlConf.html>`__ format (as of 23.1), merging your own additions is possible by
placing files with a :code:`.conf` extension in the directory :code:`/usr/local/etc/swanctl/conf.d/`.

.. Warning::

    Files added to these directories will not be mainted by the user interface, if you're unsure if you need this, it's likely
    a good idea to skip adding files here as it might lead to errors difficult to debug.

.. Note::

    Prior to version 23.1 it was also possible to add secrets and ipsec configurations in :code:`/usr/local/etc/ipsec.secrets.opnsense.d/`
    and :code:`/usr/local/etc/ipsec.opnsense.d/`, with the switch to 23.1 these files are deprecated and should be manually migrated into swanctl.conf
    format.



--------------------------
OpenVPN (SSL VPN)
--------------------------

One of the main advantages of OpenVPN in comparison to IPsec is the ease of configuration, there are fewer settings involved
and it's quite simple to export settings for clients.

.................................
General context
.................................

The OpenVPN module incorporates different functions to setup secured networks for roadwarriors and side to side connections.
Since the start of our project we organized the openvpn menu section into servers and clients, which actually is a role
for the same OpenVPN process. As our legacy system has some disadvantages which are difficult to fix in a migration, we have chosen
to add a new component named :code:`Instances` in version 23.7 which offers access to OpenVPN's configuration in a similar way as
the upstream `documentation <https://openvpn.net/community-resources/reference-manual-for-openvpn-2-6/>`__ describes it.
This new component will eventually replace the existing client and server options in a future version of OPNsense, leaving
enough time to migrate older setups.

.. Tip::

  When upgrading into a new major version of OPNsense, always make sure to read the release notes to check if your setup
  requires changes.

.. Note::

  OpenVPN on OPNsense can also be used to create a tunnel between two locations, similar to what IPsec offers. Generally
  the performance of IPsec is higher which usually makes this a less common choice.
  Mobile usage is really where OpenVPN excells, with various (multifactor) authentication options and
  a high flexibility in available network options.


The following functions are available in the menu (as of OPNsense 23.7):

* Instances

  * New instances tool offering access to server and client setups

* Servers

  * Legacy server configuration tool

* Clients

  * Legacy client configuration tool

* Client Specific Overrides

  * Set client specific configurations based on the client’s X509 common name.

* Client Export

  * Export tool for client configurations, used for server type instances

* Connection Status

  * Show tunnel statusses

* Log File

  * Inspect log entries related to OpenVPN


....................................
Public Key Infrastructure  (X.509)
....................................

OpenVPN is most commonly used in combination with a public key infrastructure, where we use a certificate autority which
signs certificates for both server and clients (Also know as TLS Mode).
More information about this topic is available in our  :doc:`Trust section <certificates>`.

.. Tip::

  As of version 24.1 OPNsense is able to use OCSP to validate client certificates when using the new Instances. Make sure :code:`Use OCSP (when available)`
  is enabled in the trust section of the server instance and the CA used contains a proper :code:`AuthorityInfoAccess` extension
  as described in our  :doc:`Trust section <certificates>`.

.................................
Firewall rules
.................................

To allow traffic to the tunnel on any interface, a firewall rule is needed to allow the tunnel being established.
The default port for OpenVPN is :code:`1194` using protocol :code:`UDP`.

After communication has been established, it's time to allow traffic inside the tunnel. All OpenVPN interfaces defined in
OPNsense are  :doc:`grouped <firewall_groups>` as `OpenVPN`.

.. Tip::

    In order to use features as policy based routing or manual routes, you can :doc:`assign <interfaces>` the underlying
    devices and use them in a similar fashion as physical interfaces.


.................................
High availability [CARP]
.................................

When operating an OpenVPN server, there's not much needed to allow an active/passive setup for your environment other then
using a virtual (CARP) address. As the server will stop receiving traffic when the virtual address doesn't it,
the backup will eventually become out of service automatically.

In client mode, the OpenVPN instance needs to stop trying to reconnect when it's not in :code:`MASTER` mode, the legacy
client module shutsdown all instances directly attached to the interface. Our new instances module allows to select
the :code:`vhid` to track. In most cases an explicit bind isn't needed for a client, the default for a client is to
use the :code:`nobind` option.

.. Note::
  It's not possible to move between machines fully seamless  as the client will have to reconnect in order to reach a
  valid state again.



.................................
Examples
.................................

This paragraph offers examples for some commonly used implementation scenarios.

.. Note::

    When using a site to site example with :code:`SSL/TLS` instead of a shared key, make sure to configure "client specific overrides"
    as well to correctly bind the remote networks to the correct client.


Legacy (:menuselection:`VPN -> OpenVPN -> Client|Server`)
------------------------------------------------------------------------------

.. toctree::
   :maxdepth: 2
   :titlesonly:

   how-tos/sslvpn_s2s
   how-tos/sslvpn_client


New (:menuselection:`VPN -> OpenVPN -> Instances`)
------------------------------------------------------------------------------

.. toctree::
   :maxdepth: 2
   :titlesonly:

   how-tos/sslvpn_instance_s2s
   how-tos/sslvpn_instance_roadwarrior



.................................
Client Specific Overrides
.................................

The mechanism of client overrides utilises OpenVPN :code:`client-config-dir` option, which offer the ability to use
specific client configurations based on the client's X509 common name.

It is possible to specify the contents of these configurations in the gui under :menuselection:`VPN -> OpenVPN -> Client Specific Overrides`.
Apart from that, an authentication server (:menuselection:`System -> Access -> Servers`) can also provide client details in special cases when returning
:code:`Framed-IP-Address`, :code:`Framed-IP-Netmask` and :code:`Framed-Route` properties.

.. Tip::

      Radius can be used to provisioning tunnel and local networks.

A selection of the most relevant settings can be found in the table below.

.. csv-table:: Client Specific Overrides
   :header: "Parameter", "Purpose"
   :widths: 30, 40

   "Disabled", "Set this option to disable this client-specific override without removing it from the list"
   "Servers", "Select the OpenVPN servers where this override applies to, leave empty for all"
   "Common name", "The client's X.509 common name, which is where this override matches on"
   "IPv[4|6] Tunnel Network", "The tunnel network to use for this client per protocol family, when empty the servers will be used"
   "IPv[4|6] Local Network", "The networks that will be accessible from this particular client per protocol family."
   "IPv[4|6] Remote Network", "These are the networks that will be routed to this client specifically using iroute, so that a site-to-site VPN can be established."
   "Redirect Gateway", "Force the clients default gateway to this tunnel"

.. Note::

      When configuring tunnel networks, make sure they fit in the network defined on the server tunnel itself to allow the server to send data back to the client.
      For example in a :code:`10.0.0.0/24` network you are able to define a client specific one like :code:`10.0.0.100/30`.

      To reduce the chances of a collision, also make sure to reserve enough space at the server as the address might already be assigned to a dynamic client otherwise.


.. Tip::

      When using topology "subnet" the netmask usually equals the one defined in the instance itself as the gateway
      being pushed to the client is the first adress in the network and otherwise unreachable.

--------------------------
Wireguard
--------------------------

.................................
General context
.................................

WireGuard® is a simple yet fast and modern VPN solution, which in some cases is more convenient than IPsec or OpenVPN, certainly
in terms of options you need to configure. In our experience IPsec is the fastest solution for site-to-site connections, but Wireguard is the simplest
option to setup.

A wireguard setup on our end exists of the following main components:

* Instances: in the wireguard configuration these are called "interfaces" and they describe how the virtual :code:`wgX` device on our end is configured in terms of addressing and cryptography.
* Peers: these are the clients that are allowed to connect to us, described by their optional remote address including the networks that are allowed to pass through the tunnel. Peers belong to one or more instances.

.................................
Instances
.................................


In order to configure an instance, we start by adding one in the gui and generate a keypair. The public key is usually required for the
other end of the tunnel (peer). An unused port to listen on is required as well. The tunnel addresses are configured on the :code:`wgX` device
(which is always visible in :menuselection:`Interfaces --> Overview`).

By default, when "*Disable routes*" is not set, routes are created for each connected peer to the networks selected in "*Allowed IPs*", optionally
only a single gateway route might be configured as well.

.. Note::

   When choosing tunnel addresses, make sure the network defined includes the addresses being used by the peers. For
   example when choosing :code:`10.10.0.1/24` the :code:`wgX` interface has this address configured and is able to accept
   a peer using :code:`10.10.0.2/32`.


.. Note::

   Make sure to enable Wireguard in the general tab before adding instances.


.. Tip::

  Remember to create a firewall rule to allow traffic to the configured port and inside the tunnel.

.................................
Peers
.................................

Peers define the hosts that we exchange information with, which might be a road-warrior type or a static destination, in which case
you either provide or omit an "*Endpoint Address and Port*". At minimum you need the public key of the other party, optionally you may offer a pre-shared key
as additional security measure. The "*Allowed IPs*" define the networks that are allowed to pass the tunnel.

.. Note::

  In most cases the "*Allowed IPs*" list contains the networks used on the remote host and the peer ip address
  (instance/tunnel address) configured on the other end.

.. Tip::

    When NAT and firewall traversal persistence is required, the :code:` Keepalive interval` can be used to exchange packets every defined
    interval ensuring states will not expire.


.................................
High availability (using CARP)
.................................

When using wireguard on active/passive high availability clusters, only one instance at a time is allowed to communicate to the
other party. In OPNsense this can be reached by selecting a :code:`vhid` to track as instance dependancy {Depend on (CARP)}.

If an instance depends on a CARP vhid, it will query the current status and determine if the interface should be usable (when MASTER), the
interface status (up/down) will be toggled accordingly.


.. Note::

  As the interface itself will not change, all of its addresses and routes remain when not being active. This ensures a relatively
  quick switch between roles.


.. Tip::

  Because the carp dependancy is managed per instance, you are able to keep tunnels available selectively, for example to manage the machines
  remotely.


.................................
Diagnostics and debugging
.................................

In :menuselection:`VPN --> WireGuard --> Diagnostics` you can find the configured instances and peers including their last known
handshake and the amount of data being exchanged. For Instances you are also able to see if the device underneath (:code:`wgX`) is
up or down, depending on the carp status described in the previous chapter.

.. Tip::

  Althought wireguard itself offers very limit logging, our setup process will make a note of errors and signal about certain events.
  When having issues configuring an instance or peer, always make sure to check the logs in  :menuselection:`VPN --> WireGuard --> Log File` first.


.. Warning::

  When having issues exchanging packets between both ends of the tunnel, always make sure to check if the "*Allowed IPs*"
  in the peer configurations contain the proper networks.
  In case traffic is not allowed when traveling **in**, its dropped silently (a capture will not show it),
  roughly the same happens when traveling **out**, a capture will show it, but nothing will be send out.


.. Note::

  Runtime debugging from the console is possible using the :code:`ifconfig` command,
  for more information see the upstream `manual page <https://man.freebsd.org/cgi/man.cgi?wg(4)>`__


.................................
Examples
.................................

This paragraph offers examples for some commonly used implementation scenarios.

.. toctree::
   :maxdepth: 2
   :titlesonly:

   how-tos/wireguard-s2s
   how-tos/wireguard-client
   how-tos/wireguard-client-azire
   how-tos/wireguard-client-mullvad
   how-tos/wireguard-client-proton
   how-tos/wireguard-selective-routing

--------------------------
Plugin VPN options
--------------------------

Via plugins additional VPN technologies are offered, including:

* **OpenConnect** - SSL VPN client, initially build to connect to commercial vendor appliances like Cisco ASA or Juniper.
* **Stunnel** - Provides an easy to setup universal TLS/SSL tunneling service, often used to secure unencrypted protocols.
* **Tinc** - Automatic Full Mesh Routing
* **WireGuard** - Simple and fast VPN protocol working with public and private keys.
* **Zerotier** - seamlessly connect everything, requires account from zerotier.com, free for up to 100 devices.


.. toctree::
   :maxdepth: 2
   :titlesonly:

   how-tos/openconnect
   how-tos/stunnel
   how-tos/zerotier
