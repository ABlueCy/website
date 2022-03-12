---
title: TabletServer组件化
description:
weight: 1
---

# Introduction

As Vitess adoption expands, several feature requests have been popping up that will benefit from multiple instances of TabletServer (or its sub-components) co-existing within the same process.

The following features drive this refactor, in order of priority:

* Multi-schema
* VTShovel: multiple data import sources for vreplication
* VTDirect: allow VTGate to directly send queries to mysql

Beyond these features, componentizing TabletServer will make the vitess architecture more flexible. There are many places in the code where we instantiate a QueryService. All those places can now explore the benefit of instantiating a TabletServer or its sub-components.

# 介绍

随着Vitess应用范围不断增长，出现了多个功能请求，它们将受益于同一流程中共存的多个TabletServer实例（或其子组件）。

以下特性按优先级顺序驱动了此重构：

* Multi-schema
* VTShovel: vreplication的多数据源导入
* VTDirect: 从VTGate直接向mysql发送查询

除了这些特性之外，组件化的TabletServer也会使vitess体系结构更加灵活。代码中有很多地方可以让我们实例化一个QueryService。现在这些地方都可以探索实例化TabletServer或其子组件的收益。



# Features

This section describes the use cases and their features. An important prerequisite: In order to retain backward compatibility, the new features should not cause existing behavior to be affected.

# 特性

本节描述用例及其特性。一个重要的前提条件是：为了保持向下兼容性，新特性不应导致现有行为受到影响。

## Multi-schema

There has been a steady inflow of enquiries about use cases where a mysql instance has a large number of schemas (1000+). We currently support multi-schema by requiring the user to launch one vttablet process per schema. This, however, does not scale for the number of schemas we are beginning to see.

To enable this, we need the ability for a single vttablet to host multiple TabletServers. Requirements are:

* Grouped or consolidated Stats (/debug/vars).
* Segregated or consolidated HTTP endpoints, like (/debug/status), with sub-page links working.
* A better way to specify flags: the existing approach of command line flags may not scale.
* A tablet manager that can represent multiple tablet ids.

Other parts of TabletServer have already been modified to point at a shared mysql instance due to work done by @deepthi in #4727, and other related changes.

## Multi-schema

关于mysql实例有大量模式（1000+）的用例的查询源源不断。我们目前通过要求用户对每个模式启动一个进程来支持多模式。然而，这不会调节我们开始看到的模式数量。

为了实现这一点，我们需要一个vttablet能够承载多个TabletServer。要求如下：

* 分组或合并统计数据（/debug/vars）。
* 隔离或整合的HTTP端点（/debug/status）具有子页面链接。
* 指定标志的更好方法：现有的命令标识可能无法扩展。
* 一个tablet管理器可以代表多个tablet ID。

由于@deepthi在#4727中所做的改动，TabletServer的其他部分已经被修改为指向一个共享mysql实例。

## VTShovel

VTShovel is a data migration feature that extends VReplication to allow a user to specify an external mysql as the source. This can be used to import data and also keep the targets up-to-date as the source is written to.

VTShovel currently has two limitations:

* It supports only one external source per vttablet. The need for multiple sources was voiced as a requirement by one of the adopters.
* It does not support VDiff, which also requires multiple sources.

If the TabletServer refactor is architected correctly, VTShovel should inherit the multi-instance ability without any major impact. In particular:

* Leverage the flags refactor to support more than one external source.
* Observability features implemented for multi-schema like stats and HTTP endpoints should naturally extend to vtshovel.
* VDiff should work.

## VTShovel

VTShovel是一种数据迁移特性，它扩展了VReplication，允许用户指定外部mysql作为源。这可以用于导入数据，还可以让在目标端在数据源端写入情况下保持数据实现更新。

VTShovel目前有两个限制:

* 它只支持每个vttablet对应一个外部源。这个多源的需求被看作为其中一位应用者的需求。

* 它不支持VDiff，因为VDiff也需要多个源。

如果TabletServer重构的架构设计正确，VTShovel应该继承多实例功能，而不会产生任何重大影响。尤其：

* 利用标志重构来支持多个外部源。
* 就像stats和HTTP端点，多模式应用的可观察性特性应该自然扩展到vtshovel。
* VDiff应该有效。

## VTDirect

The excessive use of CPU due to gRPC continues to be a concern among some adopters. Additionally, Vitess is now being deployed against externally managed databases like RDS and CloudSQL. Such users are reluctant to pay the latency cost of the extra hop.

VTDirect is the ability of VTGate to directly send queries to the mysql instances.

This feature adds the following requirements over the previous ones:

* Some features of TabletServer (like sequences) should be disabled or redirected to an actual vttablet.
* TabletServers can have a life-cycle as tablets are added and removed from the topo. The variables and end-points need to reflect these changes.

In previous discussions, alternate approaches that did not require a TabletServer refactor were suggested. Given that the TabletServer refactor brings us much closer to this feature, we’ll need to re-evaluate our options for the best approach. This will be a separate RFC.

## VTDirect

gRPC导致的CPU过度使用仍然是一些使用者关注的问题。此外，Vitess现在正在针对外部管理的数据库（如RDS和CloudSQL）进行部署。如此用户们不愿意支付额外的延迟成本。

VTDirect是VTGate直接将请求发送到mysql实例的能力。

与之前相比，这个特性增加了以下要求：

* TabletServer的某些功能（如序列）应该被禁用或重定向到实际的vttablet中。

* 就如tablet在topo中的添加和删除那样，TabletServer会拥有一个生命周期。变量和端点需要映射这些变化。

在之前的讨论中，提出了不需要TabletServer重构的替代方法。鉴于TabletServer重构让我们更接近这个特性，我们需要重新评估我们的选项，以获得最佳方法。这将是一个单独的RFC。

# Requirements

This section describes the requirements dictated by the features.

# 需求

本节描述了功能规定的要求。

## Stats

Stats (/debug/vars) should be reported in such a way that the variables from each TabletServer can be differentiated. Idiomatic usage of vitess expects the monitoring tool to add the tablet id as a dimension when combining variables coming from different vttablets. Therefore, every TabletServer should be changed to add this dimension to its exported variables.

On the flip side, this may result in an extremely large number of variables to be exported. If so, it may be better to consolidate them. There is no right answer; We have to support both options.

## Stats

Stats（/debug/vars）应该像每个TabletServer中的变量被区分那样展示。vitess的惯用用法是，当组合来自不同vttablets的变量时，监控工具会将vttablets id添加为一个维度。因此，应该更改每个TabletServer，将此维度添加到其输出的变量中。

另一方面，这可能会导致导出大量变量。如果是这样的话，最好是整合它们。没有正确的答案；我们不得不支持这两种选择。

### Other options considered

We could have each TabletServer export a brand new set of variables by appending the tablet id to the root variable name. However, this would make it very hard for monitoring tools because they are not very good at dealing with dynamic variable names.

### 其他考虑到的方案

我们可以通过将tablet id附加到根变量名的方式让每个TabletServer导出一组全新的变量。但是，这将使监控工具变得非常艰难，因为它们不太擅长处理动态变量名。

## HTTP endpoints

A TabletServer exports a variety of http endpoints. In general, it makes sense to have each TabletServer export a separate set of endpoints within the current process. However, in cases where the performance of the underlying mysql is concerned, it may be beneficial to consolidate certain pages.

We’ll start with a separate set of pages, with each set prefixed by the tablet id. For example, what was previously `/debug/consolidations` will now become `/cell-100/debug/consolidations`.

## HTTP endpoints

TabletServer 导出各种http端点。一般来说，每个 TabletServer 在当前进程中导出一组单独的端点才有意义。但是，在考虑底层 mysql 的性能的情况下，合并某些页面可能是有益的。

我们将从一组单独的页面开始，每个页面都以tablet ID 为前缀。 例如，以前的 `/debug/consolidations` 现在将变成 `/cell-100/debug/consolidations`。

### Other options considered

We could keep the existing set of endpoints unchanged, and have each TabletServer add its section. But this would make it hard to troubleshoot problems related to a specific TabletServer.

The best-case scenario would be the “why not both” option: the original set of pages continue to exist and provide a summary from all the tablet servers. This can still be implemented as an enhancement.

### 其他考虑到的选项

我们可以保持现有的端点不变，并让每个 TabletServer 添加它的部分。 但这会使解决与特定 TabletServer 相关的问题变得困难。

最好的情况是"全都要"选项：原来的页面继续存在并提供来自所有tablet务器的摘要。 这仍然可以实现加强。

## Command line flags

Extending the command line flags to be able to specify parameters for a thousand tablet servers is not going to be practical.

Using config files is a better option. To prevent verbosity, we can use a hierarchy where the root config specifies initial values for the flags, and the TabletServer specific configs can inherit and override the original ones.

The input file format could be yaml. Also, this is a good opportunity for us to come up with better names.

Since the config option is more powerful and flexible, specifying that file in the command line will supersede all legacy flags.

## 命令行标记

扩展命令行标志为一千tablet服务器指定参数是不切实际的。

使用配置文件是一个很好的选择。为了防止冗长，我们可以使用根配置能根据标识初始化的层次结构，这样 TabletServer 具体的配置可以继承和覆盖原始配置。

输入文件格式可以是yaml格式，当然，这也是我们想出一个更好名字的好机会。

由于配置选项更加强大和灵活，在命令行中指定的文件将取代所有旧标志。

### Other options considered

These configs could be hosted in the topo. This is actually viable. There are two reasons why this option takes a backseat:

* We currently don’t have good tooling for managing data in the topo. VTCtld is currently the only way, and people have found it inadequate sometimes.
* There are mechanisms to secure config files, which will allow it to contain secrets like the mysql passwords. This will not be possible in the case of topos.

### 其他考虑到的选项

这些配置可以在 topo 中管理。实际是可行的。这个选择放在次要位置有两个原因。

* 我们目前没有很好的工具来管理 topo 中的数据。 VTCtld 是目前唯一的方法，有时，人们会发现它功能有所欠缺。
* 这里需要一些机制保护配置文件，从而使文件能存放像mysql密码那样私密的东西。这是 topo 所不具备的。

# Design

We propose to address the above requirements with the following design elements.

# 设计

我们提倡使用下列设计元素来解决上述需求。

## Dimension Dropper

The dimension dropper will be a new feature of the stats package. Its purpose is to remove specific dimensions from any multi-dimensional variable.

We’ll introduce a new command-line flag that takes a list of labels as input, like `-drop_dimensions='Keyspace,ShardName'`. The stats package will then remove that dimension from any variable that refers to it.

In the case of the TabletServer, specifying `TabletID` in the list of dropped dimensions will have the effect of all TabletServers incrementing a common counter instead of different ones under their own tablet id.

The reason for this approach is that there are already other use cases where the number of exported variables is excessive. This allows us to address those use cases also.

It’s possible that this feature is too broad. For example, one may not want to drop the `Keyspace` dimension from all variables. If we encounter such use cases, it should be relatively easy to extend this feature to accommodate more specific rules.

## Dimension Dropper

dimension dropper 将会是stats包的一个新特性. 其目的是从任何多维变量中删除指定的维度。

我们会引入一个以列表作为输入的新的命令行，例如：`-drop_dimensions='Keyspace,ShardName'`。随后，stats 包将删除该维度所涉及到的所有变量。

在 TabletServer 的情况下，在删除的维度列表中指定“TabletID”将具有所有 TabletServers 增加一个公共计数器的效果，而不是在它们自己的平板电脑 ID 下增加不同的计数器。

这个方案提出的原因是已经有一些扩展变量的案例在使用了。这样也允许我们来处理这些案例。

这个特性可能太宽泛了。例如，有个场景可能不希望从所有变量中删除'Keyspace'维度。如果我们遇到这样的场景，这个特性应该很相对比较容易的扩展这个特性，来满足一些具体的规则。

## Exporter

The exporter will be a new feature that will layer between TabletServer and the singleton APIs: stats and http. It will allow you to create exporters that are either anonymous or named.

An anonymous exporter will behave as if you invoked the stats and http directly. Open issue: we’ll need to see if we want to protect from panics due to duplicate registrations.

A named exporter will perform consolidations or segregations depending on the situation:

* In the case of a stats variable, it will create a common underlying variable, and will update the dimension that matches the name of the exporter.
* In the case of http, it will export the end point under a new URL rooted from the name of the exporter.

Currently, the connection pools have a different name based mechanism to export different stats. The exporter functionality should support this prefixing, which will eliminate the boiler-plate code in those components.

A prototype of this implementation (aka embedder) is present in the vtdirect branch. This needs to be cleaned up and ported to the latest code.

There is no need for the exporter to provide the option to consolidate without the name because the dimension dropper can cover that functionality.

It’s possible to achieve backward compatibility for stats by creating an exporter with a name (tablet id), and then dropping that dimension in stats. However, it’ll work only for stats and not for the http endpoints. For this reason, we need to support explicit code paths for the anonymous exporters. Plus, it makes things more explicit.

## Exporter

Exporter将是一个新特性，它将在TabletServer和singleton API之间分层：stats和http。这个特性允许你创建匿名或者定义的exporters

匿名Exporter表现就像你直接调用stats和http一样。存在的问题：我们需要考虑是否希望避免因重复注册而引发的panic。

命名的Exporter将根据情况进行合并或分离：

* 对于stats变量，它将创建一个公共基础变量，并更新与命名的Exporter匹配的维度。
* 对于http，它会将导出一个根据命名exporter产生的新URL下的终端。

目前，连接池基于不同机制有着不同的名字来导出不同的stat。Exporter特性应该支持这种前缀，这将消除这些组件中的公式化的代码。

vtdirect分支中提供了该实现的原型（也称为嵌入式程序）。这需要清理并移植到最新的代码。

Exporter不需要提供不带名称的合并选项，因为dimension dropper可以包含该功能。

可以通过创建一个带有名称（tablet id）的Exporter，然后在stats中删除该维度，这样的方式实现统计数据的向后兼容性。然而，它只适用于统计数据，而不适用于http端点。因此，我们需要支持匿名Exporter的明确的代码路径。此外，它使事情更加清晰。

## Config loader

The TabletServer already has most, if not all, of its input flags consolidated into a `Config` struct under tabletenv. The existing flags initialize a `DefaultConfig` global variable. If the command line specifies a newly defined flag, like `-tablet_config='filename.yaml'`, then we can branch off into code that reads the yaml file and initializes the configs from there.

The code will load the global part of the yaml into a “global” Config. For each tablet specific config, the global config will be copied first, and then the tablet specific overrides will be overwritten into the copied values.

This is an opportunity for us to rename the members of the Config struct to use better names and data types. The yaml tags will have to match these new names.

The most popular yaml reader seems to be https://github.com/go-yaml/yaml. We’ll start with that and iterate forward.

The dbconfigs data structure will also be folded into the `Config`. This is because each tablet could potentially have different credentials.

### Bonus points

Given that vitess uses protos everywhere, we could look at standardizing on a generic way to convert yaml to and from protos. This will allow us to look at converting all formats to yaml. If this sounds viable, we can convert the `Config` struct to be generated from a proto, and then have yaml tags that can convert into it. This will future-proof us in case we decide to go this route.

On the initial search, there is no standard way to do this conversion. It would be nice if protos supported this natively as they do for json. We do have the option of using this code to build our own yaml to proto converter: https://github.com/golang/protobuf/blob/master/jsonpb/encode.go.

## TabletManager

The TabletManager change will put everything together for the multi-schema feature.

The ActionAgent data structure will be changed to support multiple tablet servers:

* QueryServiceControl will become a list (or map)
* UpdateStream will be deleted (deprecated)
* TabletAlias, VREngine, _tablet and _blacklistedTables will be added to the QueryServiceControl list

All other members of TabletManager seem unaffected.

The tablet manager API will be extended for cases where requests are specific to a tablet id. For example, `GetSchema` will now require the tablet id as an additional parameter. For legacy support: if tablet id is empty, then we redirect the request to the only tablet.

Note: VREngine’s queries are actually tablet agnostic. The user is expected to restrict their queries to the dbname of the tablet. This is not a good user experience. We should tighten up the query analyzer of vrengine to add dbname as an additional constraint or fill in the correct value as needed.

## VReplication

VReplication should have a relatively easy change. We already have a field named external mysql. This can be a key into the tablet id Config, which can then be used to pull the mysql credentials needed to connect to the external mysql.

The multi-instance capabilities of VStreamer will naturally extend to support all the observability features we’ll add to it.
