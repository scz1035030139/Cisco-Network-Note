本文通过一套自拟的双 Spoke 与单 Hub 拓扑，说明 AutoVPN 使用相同 RD 或不同 RD 时，原始 VPNv4 路由、imported 副本、本地 label 和业务 VRF 上下文的区别。重点是：原始条目提供什么 label 转发动作，imported 条目又如何为业务 IP lookup 提供路由。

核心区别是：相同 RD 且 import 通过时，IOS BGP 可以复用收到的路由条目及其路径，让原始 VPNv4 路由同时归属本地业务 VRF；不同 RD 时，通常保留原始 VPNv4 条目，再在本地业务 VRF 中建立 imported 副本。两种方式都可以有正确的业务 IP 路由，但入站 label 的绑定对象可能不同。

# 示例拓扑与前提

| 设备或主机 | 角色 | 业务地址或前缀 | 业务 VRF |
| - | - | - | - |
| Hub | AutoVPN 中转节点，并将下一跳改为自己 | 不挂接本例的目的主机 | CLOUD_DEFAULT_VRF |
| Spoke1 | 目的站点 | 对外通告 10.1.100.11/32 | CLOUD_DEFAULT_VRF |
| Spoke2 | 源站点 | 对外通告 10.1.100.16/32 | CLOUD_DEFAULT_VRF |
| Host-A | Spoke1 后的目的主机 | 10.1.100.11 | 由 Spoke1 接入业务 VRF |
| Host-B | Spoke2 后的源主机 | 10.1.100.16 | 由 Spoke2 接入业务 VRF |

逻辑拓扑为：Host-B ↔ Spoke2 ↔ Hub ↔ Spoke1 ↔ Host-A。Spoke1 和 Spoke2 分别与 Hub 建立 AutoVPN 隧道，本例不建立 Spoke 直连隧道；底层 WAN 网络只负责承载外层 IPsec 报文。

测试方向为 Host-B 的 10.1.100.16 发往 Host-A 的 10.1.100.11。所有节点的业务 VRF 均 import 和 export RT 1:1。下面只改变 RD，保持目的前缀、RT 和流量方向不变。

| 比较方案 | Hub RD | Spoke1 RD | Spoke2 RD | 三者 import 和 export RT |
| - | - | - | - | - |
| 相同 RD | 1:1 | 1:1 | 1:1 | 1:1 |
| 不同 RD | 1:1 | 11:11 | 6:6 | 1:1 |

路由、label 和表项均为讲解用的示意值，不是设备实测输出。Spoke1 为目的前缀分配的 label 在两种方案中都假设为 19；Hub 在相同 RD 方案中使用本地 label 1229，在不同 RD 方案中使用本地 label 1663。这些数字不由 RD 数值直接计算。

本例按 per-prefix label 方式讲解，并假设 Hub 作为对外通告的下一跳，需要提供本地 label。若下一跳不变，BGP 可以继续传递原下一跳的 label，不能据此假定 Hub 一定另分本地 label。

# 关键概念

| 概念 | 作用与边界 |
| - | - |
| RD | 与 IP prefix 一起标识 VPNv4 路由。RD 相同不等于自动允许 import，仍需满足相应 import 条件。 |
| RT | 控制路由可以导入哪些 VRF。相同 RT 允许导入，不保证原始 label 自动绑定到某个本地 VRF。 |
| 本地 VRF ID | 本机业务路由上下文的标识，与 RD、RT、label 的数值不是同一个概念。 |
| BGP 前缀条目 net | 保存一个前缀的候选路径和该条目的 incoming label 等状态。一个条目可以有多条 path。 |
| BGP 路径 path | 保存一条候选路径，包括邻居、下一跳、远端 outgoing label 和属性等。 |
| RIB 和 FIB | 分别保存业务选路结果与实际转发表。它们是另外的对象，不与 BGP net/path 共用同一块内存。 |
| Incoming label | 本机分配、供其他设备交给本机使用的 label。BGP 输出中的 in 对应该条目的本地 label。 |
| Outgoing label | 向下一跳发送时使用的远端 label。BGP 输出中的 out 来自所选 path。 |
| AutoVPN metadata label | AutoVPN 将 VPN label 放在受 IPsec 保护的 metadata 中。名为 DST-VRF 的字段承载的是 label，不能直接当作接收方的本地 VRF ID。 |

Label 是接收设备本地有效的标识。Hub 分配的 1663 和 Spoke 分配的 19 可以表示同一条端到端 VPN 路由在不同接收节点上的转发上下文。

# 两种方式的主要区别

| 比较项 | 相同 RD 且 import 通过 | 不同 RD 且通过 RT import |
| - | - | - |
| 原始路由与业务 VRF 的关系 | 原始 BGP 条目本身归入匹配的本地业务 VRF | 原始条目保留原 RD，本地 VRF 使用另外的 imported 条目 |
| 单个远端路径的对象关系 | 同一份 net/path 兼顾收到的 VPN 路由和本地业务 VRF 候选路由 | 原始 net/path 与 imported net/path 分离 |
| Imported 副本 | 对匹配的 VRF 可省去独立副本 | 在每个不同 RD 的目标 VRF 中查找或建立副本 |
| 业务 IP 路由 | 可以安装到业务 RIB/FIB | 同样可以安装到业务 RIB/FIB |
| 所讨论实现中的本地 label 绑定 | 可绑定业务 IP FIB，从中得到有效本地 VRF ID | 原始 VPNv4 label 绑定中转动作，但缺少有效本地业务 VRF ID |
| 所讨论实现中的 AutoVPN 接收结果 | 从 label 恢复业务 table，再查 inner IP | 需要支持 label 中转对象，不能将未知本地 VRF 当成 global IP table |

相同 RD 的对象复用原本是一种节省 VPN 表内存的优化。不同 RD 的对象分离则保留远端 VPN 路由的原始身份，并允许它被多个本地 VRF 分别导入和使用。

相同 RD 不会把所有路由合成一个对象。这里的复用针对同一个 VPNv4 前缀；不同前缀仍是不同条目，同一前缀从多个邻居学到时仍可有多条候选路径。

远端路由被标记为属于本地 VRF，也不等于它变成了本机起源或从本地 CE 学到的路由。

# 同一目标路由的两种表示

比较 Hub 如何保存和转发目的前缀 10.1.100.11/32。Hub 向 Spoke2 通告自己为下一跳，并提供 Hub 的本地 incoming label；Hub 向 Spoke1 发包时，使用 Spoke1 提供的 outgoing label 19。

## 相同 RD 的路由与转发

| BGP VPNv4 条目 | Incoming label | Outgoing label | 本地业务关联 |
| - | - | - | - |
| 1:1:10.1.100.11/32 | 1229 | 19 | CLOUD_DEFAULT_VRF |

Hub 的示意 LFIB 关系为：本地 label 1229，远端 label 19，目的为 10.1.100.11/32，输出路径通往 Spoke1，并具有 CLOUD_DEFAULT_VRF 的业务 IP FIB 绑定。

原始收到的路由与本地业务 VRF 使用的是同一份 BGP 条目及路径。本地 label 具有业务 IP FIB 绑定，AutoVPN 可以据此取得业务 VRF，再对 inner destination 10.1.100.11 做 IP lookup，选择通向 Spoke1 的 Virtual-Access，并重新封装加密发送。

Spoke2 发往 Hub 的 AutoVPN metadata 携带 Hub 的 label 1229。Hub 解密并解析 metadata 后，可以从该 label 恢复 CLOUD_DEFAULT_VRF，查询目的 IP，再向 Spoke1 重新封装加密；新报文的 metadata 使用 Spoke1 的 label 19。Spoke1 接收后使用自己本地的 label 19 完成相应处理。

## 不同 RD 的路由与转发

| 条目角色 | BGP VPNv4 条目 | 来源 | Incoming label | Outgoing label |
| - | - | - | - | - |
| 原始远端路由 | 11:11:10.1.100.11/32 | Spoke1 | 1663 | 19 |
| 本地业务 VRF 副本 | 1:1:10.1.100.11/32 | imported from 11:11:10.1.100.11/32 | nolabel | 19 |

### 原始条目负责保存 VPNv4 身份和 label 中转动作

原始条目为 11:11:10.1.100.11/32，incoming label 为 1663，outgoing label 为 19。它保存从 Spoke1 收到的原始 VPNv4 路由，以及“收到 Hub 的 1663 → 改用 Spoke1 的 19 → 发往 Spoke1”的 label 中转动作，没有通过 same-RD 分支归入 Hub 的业务 VRF。

这是该条目在 LFIB 和下游 OCE 中提供的转发职责，不等于任意 AutoVPN 接收实现都会直接执行该中转动作。若接收路径要求先恢复本地业务 VRF，还需要单独处理这个上下文问题。

### Imported 条目负责业务 VRF 内的 IP lookup

Imported 条目为 1:1:10.1.100.11/32，来源为 11:11:10.1.100.11/32，incoming label 为 nolabel，outgoing label 仍为 19。它为 CLOUD_DEFAULT_VRF 的选路与 RIB/FIB 安装提供路由：packet 已经处于这个业务 VRF 时，可以按目的 IP 查到通往 Spoke1 的路径，并在发送时使用远端 label 19。

nolabel 仅表示这个 imported 条目没有独立的 incoming label，不表示业务 IP 路由不可用，也不表示不能加密发送。多一个 imported 副本，不等于必然多分配一个 incoming label。

### 两条路由的查表入口不同

Spoke2 按 Hub 通告的本地 label 1663 发送报文。Hub 解密后首先按 metadata 中的 1663 查 label 表，命中原始 VPNv4 转发对象，而不是直接命中 CLOUD_DEFAULT_VRF 内的 imported IP 路由。反过来，若 packet 已经带有正确业务 VRF，它可以直接进行业务 IP lookup，使用 imported 条目安装的 FIB 路由。

本例的原始 label 中转对象没有有效的本地业务 VRF ID。如果某条 AutoVPN 接收路径必须先从 label 恢复业务 IP table，却没有处理这种对象，就可能错误进入 global IP table，或无法继续正确转发。不能把“未知本地业务 VRF”当成“应使用 table 0”。不同 RD 本身并不必然导致丢包，关键在于接收实现能否正确处理该 label 的类型和上下文。

Hub 本地产生的业务 VRF packet 从一开始就有正确业务 VRF，可以直接查 imported 副本安装的业务 IP FIB；来自 Spoke 的入站 packet 则从 metadata label 进入，需要接收路径正确处理 label 中转或业务上下文恢复。这解释了为什么“Hub 的业务 VRF 中有路由”不能单独证明 Spoke 间中转一定正确。

# 多个 VRF 导入同一路由

下面将同一自拟拓扑扩展为 Hub 上有两个业务 VRF 的情况。继续使用 1663/19 作为示意 label；它们不是双 VRF 实测输出，也不表示新增 VRF 后一定保留相同的分配值。

## 两个本地 RD 都不同于 Spoke

假设 Spoke RD 为 11:11、export RT 为 1:1；Hub VRF-A 的 RD 为 1:1，VRF-B 的 RD 为 2:2，两者都 import RT 1:1。

在普通 RT import、没有要求两个副本独立重新通告本地 label 的条件下，预期关键字段为：

| 条目 | 本地归属 | Incoming label | Outgoing label |
| - | - | - | - |
| 原始 11:11:10.1.100.11/32 | VPN 原始条目 | 1663 | 19 |
| Imported 1:1:10.1.100.11/32 | VRF-A | nolabel | 19 |
| Imported 2:2:10.1.100.11/32 | VRF-B | nolabel | 19 |

两个 imported path 各自保留来源关系，并在各自 VRF 中参与选路和 RIB/FIB 安装。它们保留相同的远端 outgoing label 19，因为目的仍是 Spoke 上同一个 VPN/FEC。

普通 RT import 通常只增加副本，不会因为多导入一个 VRF，就自动多分配一个 incoming label。若重起源、对外通告或其他机制要求每个副本提供自己的本地 label，则可能另外分配或使用相应 label；具体粒度还受 per-prefix、per-VRF 等模式影响。VRF 本来已有 aggregate label，也不代表此次 import 新增了一个 label。

原始 label 1663 仍只标识自己的转发上下文。VRF-A 和 VRF-B 同时 import，并不能让它自动成为两个 VRF 各自的入站标识。这正是根据所有 import 关系，为一个原始 switching label 任意填写某个 VRF ID 会产生歧义的原因。

## 一个本地 RD 与 Spoke 相同

若 VRF-A 的 RD 改为 11:11，VRF-B 仍为 2:2，则原始 11:11:10.1.100.11/32 可以通过 same-RD import 直接归入 VRF-A，只有 VRF-B 需要另外的 imported 副本。

此时通常是两个 BGP net，而不是一个原始条目加两个副本。在本讨论的 label 分配条件下，原始条目的本地 label 可具有 VRF-A 的业务 FIB 绑定。收到这个 label 恢复的是 VRF-A；VRF-B 也 import 了该路由，并不会让同一个 label 自动选择 VRF-B。

# Label 中转的用途与故障边界

没有有效本地业务 VRF 的 VPNv4 switching label，仍然可以具有完整转发动作。例如 Hub 的 LFIB 已经知道：收到 1663，改用远端 label 19，选择通往 Spoke1 的路径。

这种模式用于只做 VPN label 中转的节点，包括没有对应本地业务 IP 路由的 Route Reflector next-hop-self 或 Inter-AS 中转场景。它可以按 LFIB/OCE 转发，不必先选择一个本地业务 VRF 再查 inner IP。

Hub 把下一跳改为自己时，需要提供在 Hub 本地有意义的 label。其他设备把 1663 交给 Hub，Hub 再改用 Spoke1 的 19；不能把 Spoke1 本地有效的 19 无条件当成 Hub 本地的同一个标识。

未知业务 VRF 表示这个 label 没有明确的本地 IP VRF 归属，不等于应该进入 global IP table。BGP 中原始条目的全局 VPN topology、本地 label 对象的未知 VRF，以及 packet 的 global IPv4 table 是不同概念。

依赖 label 恢复业务 VRF 的 AutoVPN 接收路径，需要明确处理没有本地业务 VRF 绑定的 label 中转对象。正确的业务 IP 路由和有效的下游 OCE 即使同时存在，也不能替代接收阶段缺失的上下文处理。

一种处理思路是直接执行 LFIB 已有的中转动作；另一种思路是明确建立业务 VRF 上下文，再进入相应 IP 转发处理。两者对业务 VRF 输入处理与 per-VPN policy 的执行方式可能不同，需要按设计选择，不能随意为中转 label 填入某个 imported VRF。

# 配置与排查要点

1. 若目标 AutoVPN 实现要求从入站 label 恢复本地业务 VRF，应确认其对不同 RD 的支持。本例的相同 RD 方案指同一业务 VRF 在 Hub、Spoke1、Spoke2 上都使用 1:1，并非仅让两个 Spoke 相同。

2. 共同 RD 讨论的是同一业务 VPN 跨站点的一致性，不要求 Hub 上不同的 VRF-A、VRF-B 使用相同 RD。

3. MPLS L3VPN 协议本身不要求所有 PE 使用相同 RD。不同 RD 加共同 RT 是正常的路由组织方式；这里的问题是特定 AutoVPN 实现对接收 label 上下文的处理。

4. 相同 RD 仍需要正确 RT 和 import 条件；匹配 RD 不能替代 import 策略。

5. 排查时分别检查原始 VPNv4 条目、业务 VRF imported 条目、incoming/outgoing label、LFIB/QFP OCE，以及解密后 packet 的业务 table。仅看业务 VRF 中有路由，无法证明入站 label 能恢复该 VRF。

6. 对同一组 label，确认它是绑定业务 IP FIB，还是只具有 VPNv4 中转动作。不要直接把 metadata label、RD 或本地 VRF ID 当作同一个数值。

7. 多 VRF 推导示例用于解释对象与 label 的关系，具体 label 数量还受下一跳是否变化、是否重新通告、label mode 和已有分配状态影响。

# 总结

相同 RD 且 import 通过时，原始 VPNv4 条目可直接归入匹配的本地业务 VRF，并可能获得业务 IP FIB 绑定的 label。不同 RD 时，原始条目保留 VPNv4 身份与 label 中转动作，imported 条目则为本地业务 VRF 提供 IP 路由。判断 packet 能否正确中转，必须分别检查“入站 label 命中哪个对象”和“业务 IP lookup 使用哪个 VRF”，不能只看业务 VRF 中是否已经有目的路由。
