---
title: 部署Prometheus并监控Caddy
url : 部署Prometheus并监控Caddy
date: 2024-11-05 14:58:26
categories:
- 运维
tags:
- 运维
- prometheus
- caddy
---

[普罗米修斯（Prometheus）](https://prometheus.io/) 是一套开源的监控系统，可以用来监控并存储软件程序运行中的各项指标。

<!-- more -->

# 部署

[下载](https://prometheus.io/download/)、解压。

修改文件夹中的 `prometheus.yml`，配置对自身指标的监控：

```yml prometheus.yml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

运行 `prometheus`，默认服务端口号即为 9090，已经可以通过 WEB 查看服务。

# 查看指标

通过[表达式（expression）](https://prometheus.io/docs/prometheus/latest/querying/basics/) 筛选想要查询的内容，例如 `prometheus_target_interval_length_seconds` 用来查询普罗米修斯对系统每次指标采集的间隔。

## 条件筛选

`表达式{key="value"}` 的方式筛选，例如 `{job="prometheus", quantile="0.99"}`，表示采集任务是 prometheus（监控自己），获得其中 0.99 分位数的指标（即 99% 的数据延迟都低于该值）。

# 监控Caddy

Caddy 中的 admin 服务已经按照[普罗米修斯指标标准](https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format)实现了一套监控指标，只需在全局选项中打开：

```caddy
{
    servers {
        metrics
    }
}
```

然后在普罗米修斯中添加任务：

```yml prometheus.yml
scrape_configs:
  - job_name: caddy
    static_configs:
      - targets: ['localhost:2019']
```

之后就能用表达式查看 caddy 提供的数据，例如这样的：`go_gc_duration_seconds{job="caddy"}`。

# 简单总结

指标监控是各种系统中都非常常见的一项需求，而普罗米修斯提供了一套标准化的解决方案。

至于能否在生产环境中满足具体项目的指标要求，则还是需要在项目开始前对具体情况进行考察和测试。
