---
title: "Hello Read You: 迎来 Google Reader API"
tags:
  - ReadYou
  - Android
date: 2024-02-21
---

|[English](https://Ash7.io/blog/hello-read-you-welcome-to-google-reader-api-en-us/)|中文|

在两周前，我们发布了 Read You 的 [0.9.12](https://github.com/Ashinch/ReadYou/releases/tag/0.9.12) 版本，该版本包含了 Google Reader API 集成、用户界面改进等内容。借此机会分享一下 Read You 在集成 Google Reader API 上的一些心路历程，让用户能够了解一些 Read You 的内部机制，也期望可以帮助到以后诞生的其他 RSS Reader。

## 多租户

简要描述针对 RSS Reader 的多租户概念：允许 Reader 同时拥有多个（甚至不同终端类型的）账户，并保持一致的交互体验。这解决了 Reader 本地数据备份、隔离、云同步等问题，同时也适应了更广泛的 RSS 用户群体。

不同终端类型的账户：

- **本地**：所有数据仅存储在当前设备上，该 Reader 自身满足基本的 RSS 使用需求。
- **自托管**：用户自行部署、托管的 RSS 工具，可以与多个支持的 Reader 接入实现云同步。这些工具大多是开源项目，接受来自社区的代码审计，用户完全掌控自己的数据和隐私，不依赖第三方服务。代表性的 RSS 自托管工具有 FreshRSS、Miniflux、Tiny Tiny RSS 等。
- **第三方服务提供商**：成熟的商业产品整合了各类 RSS 功能，通常需要联网使用，数据和隐私由服务提供商控制和保护，用户通过其云计算技术获得更成熟的功能体验，如实时推送的新文章通知或商业广告。代表性的 RSS 服务提供商有 Inoreader、Feedly、Feedbin 等。

Read You 的开发初衷就是让 Android 也拥有自己的 “Reeder”，因此许多设计理念都来自于这位 iOS 上的 “艺术品”，多租户架构的设计也是如此。

这些是在分析了 Reeder 的多账户设计后，对 Read You 的期望：

- **数据层面**：以每个租户的唯一标识字段作为垂直切割点，用户行为模式下的任何 Query 和 Modify 都会附带上其自身的租户标识，以从底层开始隔离租户间的数据，防止租户数据越界。
- **逻辑层面**：在应用行为模式下，除了 Application 层的逻辑都需要考虑到多租户场景下存在的问题。
- **UI 层面**：根据租户类型设计不同的特征，共同特征抽象为公共组件，差异特征抽象为单独组件，需要注意提供针对性的状态管理。

Read You 在立项起就开始为多账户设计做着准备，目前 Read You 在 UI/UX 层面对各账户类型的共性把握得不错，在 Key Vision 的基础上根据不同账户类型支持的 API 做了一些差异化，例如不支持写操作的 Fever 账户。

但在数据库建模方面也存在一些难以调和的问题（曾经的草率）。就目前的表结构而言，更好的设计应该是：Article、Feed、Group 的 ID 都应为 UUID，仅供本地 CRUD 使用。另外，单独提供 External ID 字段用于幂等控制，对于本地账户类型，该值可以是 RSS 1.0、RSS 2.0、Atom 中的 ID、GUID 等值。

在支持第三方服务集成时，这个坑显得尤其棘手，待支持了文章数据的 导入/导出 功能后一定要尽早填上，虽说届时又少不了一顿数据库的迁移了。

## 16 年前的 Google Reader

Google 从未公开发表过 Google Reader 的 API 文档，目前市面上流传的各式文档都是当年对其逆向而来的结果。

Google Reader 中的 Article 是其 Item 概念中的一种，在向服务器传递 Item 的 ID 时，为了减小报文体积， ID 一般为长整型：

```
150177826473082
```

而当服务器向客户端返回 Item 详情时，其中的 Item ID 一般为特定格式：

```
tag:google.com,2005:reader/item/000088960000047a
```

前面这段 `tag:google.com,2005:reader/item/` 为固定值，后面这段 `000088960000047a` 是由 `150177826473082` 转换为 64 位无符号填充 0 的 16 进制数得来，对应的 Kotlin 代码：

```kotlin
fun String.ofItemIdToHexId() = String.format("%016x", toLong())
```

在 Google Reader 中，Tag 概念是一种较为通用的、对信息进行过滤和组织的操作，例如：

- 为 Article 添加 Tag 可以将其设置为已加星标、已读。
- 为 Feed 添加 Tag，被添加了相同 Tag 的 Feed 就组成了 Folder（或称 Group、Category）。

很显然一个 Feed 是可以被同时添加多个 Tag 的，Feed 与 Folder 在 Google Reader 中是多对多的关系。与其集成的 Reader 如果不支持多 Folder 的关系，就需要做一些针对性的改变，比如只取 Feed 所对应的第一个 Folder。

在 Google Reader 中，对信息进行组织、过滤后就形成了各式各样的 Stream：

- **所有 Item**：`user/-/state/com.google/reading-list`
- **已加星标的 Item**：`user/-/state/com.google/starred`
- **已读的 Item**：`user/-/state/com.google/read`
- **Folder、Tag 或 Smart**：`user/-/label/...`
- **Feed**：`feed/...`
- **Broadcast**（侧重于 Article 的第三方服务商一般会返回空结果）：`user/-/state/com.google/broadcast`

通过 Google Reader API 来查询 Stream 时，可以配合灵活的差集操作来进一步缩小查询范围，降低服务器压力和减小报文体积，例如获取 `2024-02-20 12:00:00` 之后才被加入到服务器上的 100 个未读 Item：

```
GET /stream/items/ids
?s=user/-/state/com.google/reading-list
&xt=user/-/state/com.google/read
&ot=1708401600
&n=100
```

## 各式各样的 “Google Reader”

按照 Google 模式实现了 Google Reader API 的第三方服务商有很多，支持度比较完善的有 FreshRSS、BazQux 等，而其他第三方服务商的实现可能会有些差异：

- Inoreader：现在需要走 OAuth 2.0 认证。
- Miniflux：认证接口传 `output=json` 时，Response 为 JSON 结构，这无可厚非。但目前一些供应商在实现此认证接口时，为了兼容性是会选择与 Google Reader 一样忽略认证接口传过来的 `output` 参数，所以它们的 Response 是 `text/plain` 的，其各字段值以换行符进行分割。

现今支持 Google Reader API 的服务商只是实现了其部分的接口，这些接口一般都是较常用的、在整个同步过程中不可或缺的，在做这些接口的集成时，就需要注意各个服务商对这些接口的支持程度，有时候就需要在兼容性上对接口的选择做出妥协。

以 `/mark-all-as-read` 为例，该接口允许一次性将 Stream 中多达 50k 个 Item 标记为已读，但很可惜不是所有的实现服务商都支持了该接口，所以只能选择最普通、兼容性最好的 `/edit-tag` 接口来将 Item 按批次逐量标记为已读。

## 同步策略

相比其他 API，Google Reader API 的自由度更高，同步过程中有多种实现选择。

同步过程主要包括：

1. **同步 Tag**：`/tag/list`
2. **同步 Folder**：`/subscription/list`
3. **同步 Feed**：`/subscription/list`
4. **同步文章及状态**

各 Reader 对前三点的实现方式基本相同，但在第四点上因复杂度、条件、侧重点不同，实现可能会有较大差异。

在同步文章及状态时所要考虑的一些问题：

> 同步多久以前的文章？

请设想一下，如果服务器上有 3 年至今多达 100k 的文章，那么完全同步时对 Reader 来说是何种的灾难场景？

> 未读文章全部都要吗？

这一点看上去毋庸置疑，我们就是要同步未读文章来让用户阅读才做出的 Reader，但那些 3 年前的未读文章还有必要同步吗？

> 已加星标文章全都要吗？

已加星标文章对用户来说那是至关重要的 “资产”，我们当然希望 Reader 能把服务器上所有已加星标的文章都同步下来，供用户随时浏览。但如果服务器上有 50k 的已加星标文章时又是另一回事了。
  
以上的种种问题，每个 Reader 心中都有一份自己的答卷，这个 [Issue](https://github.com/FreshRSS/FreshRSS/issues/2566#issuecomment-541317776) 中论述了一些 Reader 的同步策略，开发者们也在寻找着那个心目中的 “最佳实践”。

Read You 参考了一些 Reader 集成 Google Reader API 的 “答卷”，在综合了自身的领域建模、数据库设计之后，因自身情况与 Reeder 相似，所以也是采用了与之一样的[同步策略](https://github.com/Ashinch/ReadYou/pulls?q=is:pr+is:closed+greader)。

Reeder 的同步策略：同步了完整的已加星标文章、完整的未读文章和过去长达一月内的部分已读文章，这在多设备间同步时提供了良好的阅读体验，用户总是能够获取到他们想要同步的内容。

Read You 在初始同步周期时，会获取过去一个月的所有的已读文章，为用户提供更好的多设备体验。但在以后的同步周期（第 `n` 次）里，它只获取本地设备和服务器之间存在差异的 Item，从而减少不必要的网络请求。

目前 Read You 的同步时间主要耗费在了替换已有的订阅源数据库记录上，以后可以通过修改模型来解决。同步时主要依赖于内存来操作数据交换，这对低内存设备来说可能不是很友好，如果利用数据库中间表进行数据交换的话可以解决这个问题。但目前 Read You 的其他逻辑已经消耗了过多的连接池资源，再增加连接池消耗的话会导致高内存设备的也出现滞后的体验。我们也期待在未来的迭代中通过重构模型来解决这些问题。

Read You 的详细同步过程：

1. 获取服务器上的 Tag（由于 Read You 的 Tag 功能尚未实现，所以此步骤暂且跳过）。
2. 获取服务器上的订阅源列表和其所属的 Folder（如果本地数据有缺漏，就用来自服务器的数据补上，与服务器保持幂等）。
3. 获取服务器上所有未读 Item 的 ID 集合。
4. 获取服务器上所有已加星标 Item 的 ID 集合。
5. 用服务器上未读 Item 的 ID 集合与本地未读 Item 的 ID 集合求差集，然后通过差集中的 ID 来进一步获取所对应的文章内容（每次最多可处理 10k 个 Item）。
6. 用服务器上已加星标 Item 的 ID 集合与本地已加星标 Item 的 ID 集合求差集，然后通过差集中的 ID 来进一步获取所对应的文章内容（每次最多可处理 10k 个 Item）。
7. 用服务器一个月以内的已读 Item 的 ID 集合与本地已读 Item 的 ID 集合求差集，然后通过差集中的 ID 来进一步获取所对应的文章内容（每次最多可处理 10k 个 Item）。

如果服务器上有大量的未读文章、已加星标文章或一个月内的已读文章，那么肯定会大大增加初始同步期间的耗时，但一旦同步过一次之后，本地与服务器之间的数据差集会变小，那么下次同步的速度就很快了。

可见该同步策略会比较依赖 Reader 的 “定时同步” 功能，通过定期同步来保持本地与服务器之间的数据差集不会太大，使得每次自动、手动同步时都只需要极小的耗时。

## 再会

Read You 在集成 Google Reader API 的过程中，面临了许多挑战，也取得了重要的改进，期待下一次的技术分享。

Best regards,
Ash 👋
