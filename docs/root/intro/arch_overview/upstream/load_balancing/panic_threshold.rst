.. _arch_overview_load_balancing_panic_threshold:

紧急阈值（Panic Threshold）
-------------------------------

在负载均衡过程中，Envoy 通常只考虑上游集群中的可用（健康或降级）主机。但如果集群中可用主机的百分比过低，Envoy 将忽视所有主机中的健康状况和负载均衡。这就是所谓的 *紧急阈值*。默认的紧急阈值是 50%。可以通过 :ref:`可配置的 <config_cluster_manager_cluster_runtime>` 运行时参数以及 :ref:`集群配置 <envoy_v3_api_field_config.cluster.v3.Cluster.CommonLbConfig.healthy_panic_threshold>` 来调整这个阈值。紧急阈值用于避免随着负载的增加，部分主机的故障导致在整个集群中产生级联故障的情况。

当进入紧急状态时，Envoy 有两种模式可以选择：流量将可以向任意主机发送，或者不向任何主机发送（因此将总是失败）。这个模式可以在 :ref:`集群配置 <envoy_v3_api_field_config.cluster.v3.Cluster.CommonLbConfig.ZoneAwareLbConfig.fail_traffic_on_panic>` 中进行设置。选择在紧急状态中切断流量有助于避免压垮可能失败的上游服务，因为它将在所有主机被确定为不健康之前减少上游服务的负载。然而，它放弃了 _某些_ 请求成功的可能性，即使集群中的许多主机仍然是健康的。让某个服务的请求要么全部可以成功，要么全部失败，这也是一个不错的选择，因为它将更快地完成对集群的请求。相反的，如果一个集群即使是在被降级的情况下仍然需要为 _某些_ 请求提供服务，启用这个阈值可能是没有意义的。

紧急阈值可以与优先级一起工作。如果某个优先级的可用主机数量减少，Envoy 将尝试把一些流量转移到较低优先级的主机上。如果成功地在较低的优先级中找到了足够的可用主机，Envoy 将不考虑紧急阈值。用数学术语来说，如果所有优先级的归一化总可用性为 100%，Envoy 就会无视紧急阈值，并继续按照 :ref:`这里 <arch_overview_load_balancing_priority_levels>` 描述的算法在各优先级之间分配流量负载。然而，当归一化的总可用性降到 100% 以下时，Envoy 会假设所有优先级中没有足够的可用主机。它将继续在各个优先级之间分配流量负载，但如果某个优先级的可用性低于紧急阈值，则无论其可用性如何，流量都将流向该优先级的任意主机或不流向任何主机。

以下示例解释了归一化总可用性和紧急阈值之间的关系。假设紧急阈值使用默认值 50%。

假设一个简单的配置中，有 2 个优先级，P=1 100% 健康。在这种情况下，归一化的总健康度始终是 100%，P=0 永远不会进入紧急模式，Envoy 能够根据需要将尽可能多的流量转移到 P=1。

+--------------+---------------+--------------+---------------+--------------+----------------+
| P=0 健康端点 | 到 P=0 的流量 | P=0 是否紧急 | 到 P=1 的流量 | P=1 是否紧急 | 归一化总可用性 |
+==============+===============+==============+===============+==============+================+
| 72%          | 100%          | NO           | 0%            | NO           | 100%           |
+--------------+---------------+--------------+---------------+--------------+----------------+
| 71%          | 99%           | NO           | 1%            | NO           | 100%           |
+--------------+---------------+--------------+---------------+--------------+----------------+
| 50%          | 70%           | NO           | 30%           | NO           | 100%           |
+--------------+---------------+--------------+---------------+--------------+----------------+
| 25%          | 35%           | NO           | 65%           | NO           | 100%           |
+--------------+---------------+--------------+---------------+--------------+----------------+
| 0%           | 0%            | NO           | 100%          | NO           | 100%           |
+--------------+---------------+--------------+---------------+--------------+----------------+

如果 P=1 变得不健康，则继续不考虑紧急阈值，直到健康 P=0 + P=1 之和低于 100%。这时，Envoy 开始检查每个优先级的紧急阈值。

+--------------+--------------+---------------+--------------+---------------+--------------+----------------+
| P=0 健康端点 | P=1 健康端点 | 到 P=0 的流量 | P=0 是否紧急 | 到 P=0 的流量 | P=1 是否紧急 | 归一化总可用性 |
+==============+==============+===============+==============+===============+==============+================+
| 72%          | 72%          | 100%          | NO           | 0%            | NO           | 100%           |
+--------------+--------------+---------------+--------------+---------------+--------------+----------------+
| 71%          | 71%          | 99%           | NO           | 1%            | NO           | 100%           |
+--------------+--------------+---------------+--------------+---------------+--------------+----------------+
| 50%          | 60%          | 70%           | NO           | 30%           | NO           | 100%           |
+--------------+--------------+---------------+--------------+---------------+--------------+----------------+
| 25%          | 100%         | 35%           | NO           | 65%           | NO           | 100%           |
+--------------+--------------+---------------+--------------+---------------+--------------+----------------+
| 25%          | 25%          | 50%           | YES          | 50%           | YES          | 70%            |
+--------------+--------------+---------------+--------------+---------------+--------------+----------------+
| 5%           | 65%          | 7%            | YES          | 93%           | NO           | 98%            |
+--------------+--------------+---------------+--------------+---------------+--------------+----------------+

可以通过将紧急阈值设置为 0% 来禁用紧急模式。

只要有优先级没有进入紧急模式，就按上述方法计算负载分配。当所有优先级进入紧急模式时，负载计算算法会发生变化。在这种情况下，每个优先级收到的流量将等于该优先级的主机数量在所有优先级的主机数量中的占比。例如，如果有 2 个优先级 P=0 和 P=1，每个优先级由 5 台主机组成，那么每个级别将收到 50% 的流量。如果优先级 P=0 中有 2 台主机，优先级 P=1 中有 8 台主机，则优先级 P=0 将获得 20% 的流量，优先级 P=1 将获得 80% 的流量。

但是，如果任何优先级的紧急阈值为 0%，该优先级将永远不会进入紧急模式。在这种情况下，如果所有主机都不健康，Envoy将无法选择主机，而是立即返回 "503 - no healthy upstream" 的错误响应。

请注意，紧急阈值可以按 *每个优先级* 进行配置。
