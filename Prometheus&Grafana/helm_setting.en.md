# 🛠️ Helm Configuration Guide (values.yaml)

[**English**](helm_setting.en.md) | [**한국어**](helm_setting.md)

When installing `kube-prometheus-stack` using Helm, you can customize configurations by modifying the `values.yaml` file.
Since the original `values.yaml` is very large, it is efficient to write only the changes in a separate file.

## 📝 Key Changes

The main configurations applied in this project are:

1.  **NodeSelector**: Pin major monitoring components to the **Master Node**.
2.  **Grafana Refresh Rate**: Reduce the minimum dashboard refresh interval from 5s to **1s**.
3.  **Additional Scrape Config**: Configure Prometheus to scrape metrics from `nvidia-exporter` on the Master Node.
4.  **Shorten Intervals**: Set scrape intervals for Prometheus and Node Exporter to **1s** for better real-time monitoring.

## 📄 values.yaml Example

Refer to [values.yaml](values.yaml) for the full configuration used.

> **Tip**: To extract the currently applied values:
> ```bash
> helm get values prometheus --namespace monitoring > current-values.yaml
> ```

## ⚙️ Applying Configuration

Upgrade the release using your custom `values.yaml` file.

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f values.yaml
```
