# Task 18 - Kubernetes Monitoring with CloudWatch, Prometheus and Grafana

## Objective

Deploy and monitor a Kubernetes application running on an AWS EC2 instance using:

- AWS CloudWatch
- Amazon SNS
- Prometheus
- Grafana
- Kubernetes
- Kind

## Monitoring Architecture

CloudWatch monitoring flow:

Kubernetes Application → CloudWatch → CloudWatch Alarms → SNS → Email Notification

Prometheus monitoring flow:

Kubernetes Application → Prometheus → Grafana → Monitoring Dashboard

## 1. Kubernetes Application Deployment

A simple Nginx monitoring application was deployed to the Kind Kubernetes cluster.

The application contains two replicas and is exposed using a NodePort service.

Verification commands:

```bash
kubectl get nodes
kubectl get pods
kubectl get deployments
kubectl get services
```

Application service:

- Service: `monitoring-app-service`
- Type: `NodePort`
- NodePort: `30080`

## 2. AWS CloudWatch Monitoring

Amazon CloudWatch Agent was installed and configured on the EC2 instance.

The following infrastructure metrics were monitored:

- EC2 CPU utilization
- Memory utilization
- Disk utilization
- Network-related metrics

Custom CloudWatch namespace:

`MonitoringTask`

Important custom memory metric:

`mem_used_percent`

## 3. CloudWatch Logs

CloudWatch Agent was configured to collect system and application logs.

Log groups:

- `/monitoring-task/system`
- `/monitoring-task/application`

Application errors can therefore be detected from application logs.

## 4. Amazon SNS

An SNS topic was created:

`monitoring-alerts`

An email subscription was configured and confirmed.

SNS was integrated with CloudWatch alarms to send email notifications.

## 5. CPU Monitoring and Alert

A CloudWatch alarm was created for high EC2 CPU utilization.

Alarm:

`High-CPU-Monitoring`

CPU load was generated using the stress utility.

Example:

```bash
stress --cpu 2 --timeout 300
```

The CloudWatch alarm changed from OK to ALARM and an SNS email notification was successfully received.

## 6. Memory Monitoring and Alert

CloudWatch Agent collected the custom memory metric:

`mem_used_percent`

Alarm:

`High-Memory-Monitoring`

Memory load was generated using stress.

Example:

```bash
stress --vm 1 --vm-bytes 1500M --vm-keep --timeout 600
```

The memory threshold was exceeded, the CloudWatch alarm entered ALARM state, and an SNS email notification was successfully received.

## 7. Application Error Monitoring

Application logs were sent to:

`/monitoring-task/application`

A CloudWatch metric filter was created for:

`ERROR`

Metric:

`ApplicationErrorCount`

Alarm:

`Application-Error-Alarm`

Test errors were generated using:

```bash
echo "$(date) ERROR Application failed - test alert" | sudo tee -a /var/log/monitoring-app/application.log
```

This configuration allows CloudWatch to detect application errors and trigger an alarm.

## 8. Prometheus Monitoring

Prometheus was installed in the Kubernetes `monitoring` namespace using Helm.

Verification command:

```bash
kubectl get pods -n monitoring
```

Prometheus components included:

- Prometheus Server
- kube-state-metrics
- node-exporter
- Alertmanager
- Pushgateway

Prometheus successfully collected Kubernetes metrics.

Example verified PromQL metric:

```promql
kube_pod_status_phase
```

The query returned Kubernetes pod status data, confirming that Prometheus was collecting Kubernetes metrics.

Prometheus targets were also checked to verify metric collection.

## 9. Grafana

Grafana was installed using Helm in the `monitoring` namespace.

Prometheus was configured as the Grafana data source using:

`http://prometheus-server.monitoring.svc.cluster.local`

Grafana was then used to visualize metrics collected by Prometheus.

## 10. Grafana Dashboard

A dashboard named `Kubernetes Monitoring Dashboard` was created.

The dashboard included:

- CPU Utilization
- Memory Utilization
- Pod CPU Usage
- Pod Memory Usage
- Pod Status
- Network Receive

CPU Utilization query:

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Memory Utilization query:

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

Pod CPU Usage query:

```promql
sum by (pod) (rate(container_cpu_usage_seconds_total{pod!=""}[5m]))
```

Pod Memory Usage query:

```promql
sum by (pod) (container_memory_working_set_bytes{pod!=""})
```

Pod Status query:

```promql
sum by (phase) (kube_pod_status_phase)
```

Network Receive query:

```promql
sum(rate(container_network_receive_bytes_total[5m]))
```

## 11. Application Load Testing

Application load was generated using a temporary BusyBox pod.

```bash
kubectl run load-generator \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c 'while true; do wget -q -O- http://monitoring-app-service > /dev/null; done'
```

During the test, Kubernetes resource metrics were observed through Prometheus and Grafana.

After testing, the load generator was removed:

```bash
kubectl delete pod load-generator
```

## 12. Final Verification

The following components were verified:

- Kubernetes node Ready
- Application pods Running
- Application service available
- CloudWatch infrastructure metrics
- CloudWatch system logs
- CloudWatch application logs
- CPU alarm
- Memory alarm
- Application error monitoring
- SNS email notifications for tested alarms
- Prometheus Kubernetes metrics
- Prometheus targets
- Grafana Prometheus data source
- Grafana monitoring dashboard
- Application load monitoring

## Troubleshooting

### Issue 1 - Grafana Port Already in Use

The following error was encountered:

```text
bind: address already in use
```

An old kubectl port-forward process was already listening on port 3000.

The process was identified using:

```bash
sudo lsof -i :3000
```

The old process was stopped:

```bash
kill -9 <PID>
```

Grafana port forwarding was then started again:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80 --address=0.0.0.0
```

### Issue 2 - Grafana Frontend Failed to Load

Grafana displayed:

```text
Grafana has failed to load its application files
```

The Grafana pod, service and logs were checked:

```bash
kubectl get pods -n monitoring
kubectl get svc grafana -n monitoring
kubectl logs -n monitoring deployment/grafana --tail=30
```

The Grafana pod was healthy. The existing port-forward process was restarted.

### Issue 3 - CloudWatch Memory Metric

The required memory metric was verified in the CloudWatch Agent configuration:

```text
mem_used_percent
```

The CloudWatch Agent configuration was located under:

```text
/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/
```

### Issue 4 - Stress Memory Percentage Not Supported

The installed version of `stress` did not accept a percentage value for `--vm-bytes`.

Instead of using a percentage, a fixed memory value was used:

```bash
stress --vm 1 --vm-bytes 1500M --vm-keep --timeout 600
```

### Issue 5 - Application Error Alarm Insufficient Data

The application error alarm initially displayed `Insufficient Data`.

Missing data was configured as not breaching so that periods without application errors would not be considered alarm conditions.

A new ERROR log entry was then generated to test application error monitoring.

## Result

The Kubernetes application was successfully monitored using AWS CloudWatch and Prometheus.

CloudWatch provided infrastructure metrics and log monitoring. CloudWatch alarms and Amazon SNS provided alerting and email notifications for the tested alarm conditions.

Prometheus successfully collected Kubernetes metrics, while Grafana provided dashboards for visualizing Kubernetes and infrastructure performance.

Application load testing was also performed to observe monitoring data under load.
