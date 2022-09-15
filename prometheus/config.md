```
// This block of configurations holds the alerts rules with a key called prometheus rules
  prometheus.rules: |-
// All the alerting rules are grouped when they belong to same set of instances
    groups:
// Name of the group
// this is a sample alert which is created for testing purpose of receiver configs
    - name: Sample alert
// A set of rules config which consists expression to be validated serverity and a jist of what the alert is about
// this is an example of alerting type of rule.There is one more of type of rule called recording rule which will be seen later
      rules:
// Name of the alert
      - alert: High Pod Memory
// promql expressions to be validated and alerted
        expr: sum(container_memory_usage_bytes) > 1
// The optional for clause causes Prometheus to wait for a certain duration.In this case, Prometheus will check that the alert continues to be active during each evaluation for 1 minutes before firing the alert. alerts that are active, but not firing yet, are in the pending state.
        for: 1m
// The labels clause allows specifying a set of additional labels to be attached to the alert. Any existing conflicting labels will be overwritten. The label values can be templated.
        labels:
          severity: high
// The annotations clause specifies a set of informational labels that can be used to store longer additional information such as alert descriptions or runbook links. The annotation values can be templated.
        annotations:
          summary: High Memory Usage
		  
// This group has the examples of recording rules.allows us to precompute frequently needed or computationally expensive expressions and save their result as a new set of time series. 
    - name: k8s.rules
      rules:
// rule to calculate the cpu usage of containers which belong to the same namespace and are saved in a new metric called namespace_container_cpu_usage_seconds_total_sum_rate.We can query using the same metric in prometheus once this rule is defined
      - expr: |
          sum(rate(container_cpu_usage_seconds_total{job="kubernetes-cadvisor", image!="", container!=""}[5m])) by (namespace)
// record is used to save the above expression output in a different metric name and this metric can further be used for  any alerting/dashboard purposes
        record: namespace_container_cpu_usage_seconds_total_sum_rate
// rule to calculate the memory usage of containers which belong to the same namespace and are saved in a new metric called namespace_container_memory_usage_bytes_sum.We can query using the same metric in prometheus once this rule is defined
      - expr: |
          sum(container_memory_usage_bytes{job="kubernetes-cadvisor", image!="", container!=""}) by (namespace)
        record: namespace_container_memory_usage_bytes_sum
// Group of rules which alert when any of the prometheus targets are undiscoverable
// Note the timframe here is 1m and all the groups starting here are the type of alerting rules(not recoding rules)		
    - name: Target Down
      rules:
      - alert: TargetDown
        annotations:
// this meaningful message in the alert rule along with template values will keep the message dynamic in nature
// The $labels variable holds the label key/value pairs of an alert instance. The $value variable holds the evaluated value of an alert instance.
// For ex: <50%> of the <node-exporter> targets are down. A message like this will be shown when we expand the alerts in the prom UI
// All the expressions used in the below rules will be displayed in the alerts -> message block upon clicking takes us to prometheus graph Ui and shows the reult in case if you want to dig further
          message: '{{ $value }}% of the {{ $labels.job }} targets are down.'
        expr: 100 * (count(up == 0) BY (job) / count(up) BY (job)) > 10
        for: 1m
        labels:
          severity: high
		  
// the above group gives us an overall pictuer of targets but the below one in case if we want to monitor individual targets and categorize based on severity these can be used
// Make sure we are only enabling one of it based on requirement
// Note the change in timeframe here I customizd is 15 mins
// you can bring one of the node down in the test cluster to see these alerts fire after 15 mins
    - name: kubernetes-individual-targets
      rules:
// rule for Alertmanager status
      - alert: AlertmanagerDown
        annotations:
          message: Alertmanager has disappeared from Prometheus target discovery.
        expr: |
          absent(up{job="kubernetes-service-endpoints", kubernetes_name="alertmanager",kubernetes_namespace="monitoring"} == 1)
        for: 15m
        labels:
          severity: critical
		  
// rule for CoreDNS status
      - alert: CoreDNSDown
        annotations:
          message: CoreDNS has disappeared from Prometheus target discovery.
        expr: |
          absent(up{job="kubernetes-service-endpoints", kubernetes_name="kube-dns"} == 1)
        for: 15m
        labels:
          severity: critical
		  
// rule for KubeAPI status
      - alert: KubeAPIDown
        annotations:
          message: KubeAPI has disappeared from Prometheus target discovery.
        expr: |
          absent(up{ job="kubernetes-apiservers"} == 1)
        for: 15m
        labels:
          severity: critical

// rule for KubeStateMetrics status		  
      - alert: KubeStateMetricsDown
        annotations:
          message: KubeStateMetrics has disappeared from Prometheus target discovery.
        expr: |
          absent(up{job="kube-state-metrics"} == 1)
        for: 15m
        labels:
          severity: critical

// rule for Kubelet status		
      - alert: KubernetesNodesDown
        annotations:
          message: Kubelet has disappeared from Prometheus target discovery.
        expr: |
          absent(up{job="kubernetes-nodes"} == 1)
        for: 15m
        labels:
          severity: critical
		  
// rule for NodeExporter status	
      - alert: NodeExporterDown
        annotations:
          message: NodeExporter has disappeared from Prometheus target discovery.
        expr: |
          absent(up{job="node-exporter"} == 1)
        for: 15m
        labels:
          severity: critical
		  
// rule for Prometheus status	

      - alert: PrometheusDown
        annotations:
          message: Prometheus has disappeared from Prometheus target discovery.
        expr: |
          absent(up{job="kubernetes-service-endpoints", kubernetes_name="prometheus-service", kubernetes_namespace="monitoring"} == 1)
        for: 15m
        labels:
          severity: critical    

// This group of alerts belong to various kubernetes workloads and their status		  
    - name: kubernetes-apps
      rules:
// Alert for pod which is continuously restarting..Note that the alert will give the data of namespace and the container details along with the count of restarts in the message
      - alert: KubePodCrashLooping
        annotations:
          message: Pod {{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.container
            }}) is restarting {{ printf "%.2f" $value }} times / 5 minutes.
        expr: |
          rate(kube_pod_container_status_restarts_total{job="kube-state-metrics"}[15m]) * 60 * 5 > 0
        for: 1h
        labels:
          severity: critical
		  
// Alert for the pod which is in Unknown or pending state for more than an hour

      - alert: KubePodNotReady
        annotations:
          message: Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a non-ready
            state for longer than an hour.
        expr: |
          sum by (namespace, pod) (kube_pod_status_phase{job="kube-state-metrics", phase=~"Pending|Unknown"}) > 0
        for: 1h
        labels:
          severity: critical
		  
// Alert for Deployment failure which hasnt been rolled back yet 
      - alert: KubeDeploymentGenerationMismatch
        annotations:
          message: Deployment generation for {{ $labels.namespace }}/{{ $labels.deployment
            }} does not match, this indicates that the Deployment has failed but has
            not been rolled back.
        expr: |
          kube_deployment_status_observed_generation{job="kube-state-metrics"}
            !=
          kube_deployment_metadata_generation{job="kube-state-metrics"}
        for: 15m
        labels:
          severity: critical
		  
// Alert for deployment replica mismatch that is read and desired replicas are not equal
      - alert: KubeDeploymentReplicasMismatch
        annotations:
          message: Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has not
            matched the expected number of replicas for longer than an hour.
        expr: |
          kube_deployment_spec_replicas{job="kube-state-metrics"}
            !=
          kube_deployment_status_replicas_available{job="kube-state-metrics"}
        for: 1h
        labels:
          severity: critical
		  
// Alert for StatefulSet replica mismatch that is read and desired replicas are not equal
      - alert: KubeStatefulSetReplicasMismatch
        annotations:
          message: StatefulSet {{ $labels.namespace }}/{{ $labels.statefulset }} has
            not matched the expected number of replicas for longer than 15 minutes.
        expr: |
          kube_statefulset_status_replicas_ready{job="kube-state-metrics"}
            !=
          kube_statefulset_status_replicas{job="kube-state-metrics"}
        for: 15m
        labels:
          severity: critical
		  
// Alert for StatefulSet(new version) failure which hasnt been rolled back yet 
      - alert: KubeStatefulSetGenerationMismatch
        annotations:
          message: StatefulSet generation for {{ $labels.namespace }}/{{ $labels.statefulset
            }} does not match, this indicates that the StatefulSet has failed but has
            not been rolled back.
        expr: |
          kube_statefulset_status_observed_generation{job="kube-state-metrics"}
            !=
          kube_statefulset_metadata_generation{job="kube-state-metrics"}
        for: 15m
        labels:
          severity: critical

// Alert for notifying bad daemonset status..You can observer this by bringing a worker node down in the cluster..Node exporter should see this issue
      - alert: KubeDaemonSetRolloutStuck
        annotations:
          message: Only {{ $value }}% of the desired Pods of DaemonSet {{ $labels.namespace
            }}/{{ $labels.daemonset }} are scheduled and ready.
        expr: |
          kube_daemonset_status_number_ready{job="kube-state-metrics"}
            /
          kube_daemonset_status_desired_number_scheduled{job="kube-state-metrics"} * 100 < 100
        for: 15m
        labels:
          severity: critical
		  
// Alert for notifying bad daemonset count is matching the desired number..You can observer this by bringing a worker node down in the cluster..Node exporter should see this issue
      - alert: KubeDaemonSetNotScheduled
        annotations:
          message: '{{ $value }} Pods of DaemonSet {{ $labels.namespace }}/{{ $labels.daemonset
            }} are not scheduled.'
        expr: |
          kube_daemonset_status_desired_number_scheduled{job="kube-state-metrics"}
            -
          kube_daemonset_status_current_number_scheduled{job="kube-state-metrics"} > 0
        for: 10m
        labels:
          severity: warning
		  
// Alert for notifying the ds pods if they are not supposed to run on a node
// For ex: Pods of DaemonSet monitoring/node-exporter are running where they are not supposed to run.
      - alert: KubeDaemonSetMisScheduled
        annotations:
          message: '{{ $value }} Pods of DaemonSet {{ $labels.namespace }}/{{ $labels.daemonset
            }} are running where they are not supposed to run.'
        expr: |
          kube_daemonset_status_number_misscheduled{job="kube-state-metrics"} > 0
        for: 10m
        labels:
          severity: warning
		  
// This group of alerts depicts infrastructure alerts for memory and CPU usage
    - name: Infrastructure alerts
      rules:
// Alert which notifies when actual memory usage of pods is greater than 90 % of cluster memory capacity
      - alert: Cluster Memory Usage
        expr: sum(container_memory_working_set_bytes{id="/"})/sum(machine_memory_bytes{}) * 100 > 90
        for: 5m
        labels:
          severity: high
        annotations:
          summary: Cluster Memory Usage > 90%
		  
// Alert which notifies when actual CPU usage of pods is greater than 90 % of cluster CPU capacity
      - alert: Cluster CPU Usage 
        expr: sum (rate (container_cpu_usage_seconds_total{id="/"}[5m])) / sum (machine_cpu_cores{}) * 100 > 90
        for: 5m
        labels:
          severity: high
        annotations:
          summary: "Cluster CPU Usage  > 90%"
          description: "Cluster CPU Usage on host {{$labels.instance}} : (current value: {{ $value }})." 
		  
// Below group of alerts has alerts related to kubernetes nodes,apiserver,certificates

    - name: kubernetes-system
      rules:
// Alert to notify when the node is down
      - alert: KubeNodeNotReady
        annotations:
          message: '{{ $labels.node }} has been unready for more than an hour.'
        expr: |
          kube_node_status_condition{job="kube-state-metrics",condition="Ready",status="true"} == 0
        for: 30m
        labels:
          severity: warning
		  
// Alert to notify when the components of kubernetes are running on different versions
      - alert: KubeVersionMismatch
        annotations:
          message: There are {{ $value }} different semantic versions of Kubernetes
            components running.
        expr: |
          count(count by (gitVersion) (label_replace(kubernetes_build_info{job!="kube-dns"},"gitVersion","$1","gitVersion","(v[0-9]*.[0-9]*.[0-9]*).*"))) > 1
        for: 1h
        labels:
          severity: warning

// Alert to notify when there are too many 500 errors when kubectl command is issued		  
      - alert: KubeClientErrors
        annotations:
          message: Kubernetes API server client '{{ $labels.job }}/{{ $labels.instance
            }}' is experiencing {{ printf "%0.0f" $value }}% errors.'
        expr: |
          (sum(rate(rest_client_requests_total{code=~"5.."}[5m])) by (instance, job)
            /
          sum(rate(rest_client_requests_total[5m])) by (instance, job))
          * 100 > 1
        for: 15m
        labels:
          severity: warning
		  
// ALert to notify when close to 110 pods are scheduled or running on a node..As per kube sandards it has be less than or equal to 110
      - alert: KubeletTooManyPods
        annotations:
          message: Kubelet {{ $labels.instance }} is running {{ $value }} Pods, close
            to the limit of 110.
        expr: |
          kubelet_running_pods{job="kubernetes-nodes"} > 110 * 0.9
        for: 15m
        labels:
          severity: warning
		  
// Alert to notify internal API server communication failures
// For ex: if one of the node is down this would be the message you see 
// API server is returning errors for 33.33333333333333% of requests for GET nodes proxy.
      - alert: KubeAPIErrorsHigh
        annotations:
          message: API server is returning errors for {{ $value }}% of requests for
            {{ $labels.verb }} {{ $labels.resource }} {{ $labels.subresource }}.
        expr: |
          sum(rate(apiserver_request_total{job="kubernetes-apiservers",code=~"^(?:5..)$"}[5m])) by (resource,subresource,verb)
            /
          sum(rate(apiserver_request_total{job="kubernetes-apiservers"}[5m])) by (resource,subresource,verb) * 100 > 10
        for: 10m
        labels:
          severity: critical
		  
// Alert to notify the certificate expiry within in next 7 days..These certs are used internally by kubernetes to authenticate to api server
    
      - alert: KubeClientCertificateExpiration
        annotations:
          message: A client certificate used to authenticate to the apiserver is expiring
            in less than 7.0 days.
        expr: |
          apiserver_client_certificate_expiration_seconds_count{job="kubernetes-apiservers"} > 0 and histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="kubernetes-apiservers"}[5m]))) < 604800
        labels:
          severity: warning
		  
// Alert to notify the certificate expiry within in next 24 hours..These certs are used internally by kubernetes to authenticate to api server..Note the change in the serverity
      - alert: KubeClientCertificateExpiration
        annotations:
          message: A client certificate used to authenticate to the apiserver is expiring
            in less than 24.0 hours.
        expr: |
          apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0 and histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))) < 86400
        labels:
          severity: critical   
		  
// From this block prometheus scrapping config starts
  prometheus.yml: |-
// The global configuration specifies parameters that are valid in all other configuration contexts. They also serve as defaults for other configuration sections.  
    global:  

// How frequently to scrape targets by default.Targets are the services which we wish to monitor.These can be also overwritten in individual scrape configs.This has to be tuned according to the size of clusters we dont want to overboard the data  
      scrape_interval: 5s  

// A list of scrape configurations. This is typically a section where we ask prometheus server to monitor particular URL's or metrics endpoints.In the general case, one scrape configuration specifies a single job. In advanced configurations, this may change. Targets can be static or dynamic.  

    scrape_configs:  

// The job name assigned to scraped metrics by default. This will be visible in the prometheus targets page  
// configuration responsible to scrape node exporter related data  
      - job_name: 'node-exporter'   

// service discovery: Service discovery (SD) enables you to provide that information to Prometheus from whichever database you store it in. Prometheus supports many common sources of service information, such as Consul, Amazon’s EC2, and Kubernetes out of the box  

// Kubernetes SD configurations allow retrieving scrape targets from Kubernetes' REST API and always staying synchronized with the cluster state.  

// List of Kubernetes service discovery configurations.  
        kubernetes_sd_configs:  

// Role: There can be multiple roles in this configs depending on what metrics you wish to scrape The endpoints role discovers targets from endpoints of a service.All the Service endpoints are scrapped if the service metadata is annotated with prometheus.io/scrape and prometheus.io/port annotations.  
          - role: endpoints  

//  relabel_configs: These relabeling steps are applied before the scrape occurs and only have access to labels added by Prometheus’ Service Discovery. They allow us to filter the targets returned by our SD mechanism, as well as manipulate the labels it sets.This way it gives us a granular control on what mertrics we wish to scrape using prometheus rather collecting all the data from metrics endpoints..All the relabelled clabelsa nd configs can be seen under service_discovery page from UI under each individual block  
  
// source_labels: It expects an array of one or more label names, which are used to select the respective label values  
  
// regex: A regular expression and is used to match the extracted value from source_labels  
  
// action:  There are multiple actions which are allowed in this configuration. Out of which the keep and drop actions allow us to filter out targets and metrics based on whether our label values match the provided regex.  
  
// In this particular block its recognizing the node exporter endpoint and scraping he metrics from node-exporter and keeping those metrics.  
        relabel_configs:  
        - source_labels: [__meta_kubernetes_endpoints_name] ## represents the name of the endpoints object.  
          regex: 'node-exporter' ## matches the regex  
          action: keep ## scrapes the metrics  
  
// configuration responsible to scrape kubernetes api server related data       
      - job_name: 'kubernetes-apiservers'  
        kubernetes_sd_configs:  
        - role: endpoints  
// Configures the protocol scheme used for requests. Prometheus talking to kubernetes api in a secured communication rather than http  
        scheme: https  
// In order to make theinternal secure tls communication it needs certs and token which are provided in below config  
        tls_config:  
// CA certificate to validate API server certificate with. This is available in each pod in the cluster by default   
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt  
// token for authenticating to API server  
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token  
        relabel_configs:  
// so the below block is responsible for scraping metrics from default namespace of service called kubernetes of endpoint https.Typically the defaulr kubernetes service endpoint when it resolves which consists of all api server related metrics  
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name] ## labels representing name of namespace,service and port  
          action: keep  
          regex: default;kubernetes;https   
		    
// configuration responsible to scrape kubernetes node related metrics like node ready status etc..    
      - job_name: 'kubernetes-nodes'  
        scheme: https  
        tls_config:  
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt  
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token  
        kubernetes_sd_configs:  
// The node role discovers one target per cluster node( as in 3 nodes 3 targets) with the address defaulting to the Kubelet's HTTP port.  
  
// replacement: If the extracted value matches the given regex, then replacement gets populated by performing a regex replace  
  
// target_label: If the relabel action results in a value being written to some label, target_label defines to which label the replacement should be written  
  
        - role: node  
        relabel_configs:  
// the below block means all __meta_kubernetes_node_label_(.+) will be changed to the (.+)  
// e.g. __meta_kubernetes_node_label_kubernetes_io_arch='amd64' to kubernetes_io_arch='amd64' i.e when you query in prometheus the field will be replaced as kubernetes_io_arch . Try executing kubelet_node_name in prometheus  
        - action: labelmap  
          regex: __meta_kubernetes_node_label_(.+)  
  
//  the below block means Endpoint adress within targets is by default replaced with kubernetes default service.It can be seen in targets page under kubernetes-nodes job..typically assiginign adress from where to get node metrics		    
        - target_label: __address__  
          replacement: kubernetes.default.svc:443  
		    
// the below block means that get the node name from the label __meta_kubernetes_node_name and replace it in __metrics_path__ with the given replacement  
eg : nodename is worker1 __metrics_path__ would be replaced by /api/v1/nodes/worker1/proxy/metrics    
  
        - source_labels: [__meta_kubernetes_node_name]  
          regex: (.+)  
          target_label: __metrics_path__  
          replacement: /api/v1/nodes/\{1}/proxy/metrics       
  
// configuration responsible to scrape kubernetes pod related metrics       
      - job_name: 'kubernetes-pods'  
        kubernetes_sd_configs:  
// The pod role discovers all pods and exposes their containers as targets. For each declared port of a container, a single target is generated.   
        - role: pod  
        relabel_configs:  
		  
// This block means that scrape the metrics from  targets with label __meta_kubernetes_pod_annotation_prometheus_io_scrape equals 'true',which means the user added prometheus.io/scrape: true in the pods' annotation.  
  
// In my case nothing will be scrapped from this job cause I dont have any annotations on pods. This would be 0 on my service discovery tab  
  
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]  
          action: keep  
          regex: true  
		    
// This block means that if the user overrides the scrape path via pod annotations,its value will be put in __metrics_path__ and the metrics will be read from that path of the pod  
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]  
          action: replace  
          target_label: __metrics_path__  
          regex: (.+)  
// This block means that if the user overrides the port via pod annotations(application listening port to access metrics),if the user added prometheus.io/port in the pod annotation, use this port to replace the port in __address__.By default the __address__ label is set to the <host>:<port> address of the target.  
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]  
          action: replace  
          regex: ([^:]+)(?::\d+)?;(\d+)  
          replacement: \1:\2  
          target_label: __address__  
		    
// labelmap : The labelmap action is used to map one or more label pairs to different label names.  
  
// the below block means all __meta_kubernetes_pod_label_(.+) will be changed to the (.+)  
// e.g. __meta_kubernetes_pod_label_app='armada-api' to app='armada-api' i.e when you query in prometheus the field will be replaced as app  
// this is done for easy understanding  
  
        - action: labelmap  
          regex: __meta_kubernetes_pod_label_(.+)  
		    
// This means if __meta_kubernetes_namespace matches .*(anything not empty), put its value in label kubernetes_namespace i.e when you query in prometheus the field will be replaced as kubernetes_namespace  
  
        - source_labels: [__meta_kubernetes_namespace]  
          action: replace  
          target_label: kubernetes_namespace  
		    
// This means if __meta_kubernetes_pod_name matches .*(anything not empty), put its value in label kubernetes_pod_name, i.e when you query in prometheus the field will be replaced as kubernetes_pod_name  
        - source_labels: [__meta_kubernetes_pod_name]  
          action: replace  
          target_label: kubernetes_pod_name  
  
// configuration responsible to scrape metrics from kube-state-metrics deployment.Metrics include data about kubernetes APi objects.Kube state metrics service exposes all the metrics on /metrics URI and the default metrics path is /metrics hence we dont explicitly mention it here  
  
// static configs: target to monitor.Here in this example, its listening to kubestate metrics  end point hence it can scrape  metrics. The default metrics path  is /metircs,It can be a comma seperated value in targets if there are multiple targets within same job  
        
      - job_name: 'kube-state-metrics'  
        static_configs:  
          - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']  
		    
// configuration responsible to scrape metrics from ca advisor..Typically the usage metrics  
      - job_name: 'kubernetes-cadvisor'  
        scheme: https  
        tls_config:  
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt  
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token  
        kubernetes_sd_configs:  
        - role: node  
        relabel_configs:  
		  
// the below block means all __meta_kubernetes_node_label_(.+) will be changed to the (.+)  
// e.g. __meta_kubernetes_node_label_kubernetes_io_arch='amd64' to kubernetes_io_arch='amd64' i.e when you query in prometheus the field will be replaced as kubernetes_io_arch..These renamed and replaced labels can be seen under service_discovery page from UI under each individual block  
  
        - action: labelmap  
          regex: __meta_kubernetes_node_label_(.+)  
		    
//  the below block means Endpoint adress within targets is by default replaced with kubernetes default service.It can be seen in targets page under ca advisor job..typically assiginign adress from where to get caadvisor metrics  
        - target_label: __address__  
          replacement: kubernetes.default.svc:443  
		    
// the below block means that get the node name from the label __meta_kubernetes_node_name and replace it in __metrics_path__ with the given replacement  
eg : nodename is worker1 __metrics_path__ would be replaced by /api/v1/nodes/worker1/proxy/metrics/cadvisor  
        - source_labels: [__meta_kubernetes_node_name]  
          regex: (.+)  
          target_label: __metrics_path__  
          replacement: /api/v1/nodes/\{1}/proxy/metrics/cadvisor  
  
  
// configuration responsible to scrape metrics from kubernetes service endpoints when we use internal services directly  
  
      - job_name: 'kubernetes-service-endpoints'  
        kubernetes_sd_configs:  
        - role: endpoints  
        relabel_configs:  
  
// This block means that scrape the metrics from  targets with label __meta_kubernetes_service_annotation_prometheus_io_scrape equals 'true',which means the user added prometheus.io/scrape: true in the service' annotation.   
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]  
          action: keep  
          regex: true  
// This block means that replace the __scheme__(default http) accordingly if user mentioned anything in the service annotation prometheus.io/scheme in the service' annotation.   
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]  
          action: replace  
          target_label: __scheme__  
          regex: (https?)  
		  
// This is also for service annotation of prometheus, if the user overrides the scrape path, its value will be put in __metrics_path__  
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]  
          action: replace  
          target_label: __metrics_path__  
          regex: (.+)  
// This block means that if the user overrides the port via service annotations(application listening port to access metrics),if the user added prometheus.io/port in the service annotation, use this port to replace the port in __address__.By default the __address__ label is set to the <host>:<port> address of the target.  
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]  
          action: replace  
          target_label: __address__  
          regex: ([^:]+)(?::\d+)?;(\d+)  
          replacement: \1:\2  
  
// the below block means all __meta_kubernetes_service_label_(.+) will be changed to the (.+)  
// e.g. __meta_kubernetes_service_label_k8s_app='kube-dns' to k8s_app='kube-dns' i.e when you query in prometheus the field will be replaced as k8s_app..Try executing coredns_build_info metric in prometheus	  
  
        - action: labelmap  
          regex: __meta_kubernetes_service_label_(.+)  
		    
// the below block means all __meta_kubernetes_namespace will be changed to kubernetes_namespace  
// e.g. __meta_kubernetes_namespace='kube-system' to kubernetes_namespace='kube-system' i.e when you query in prometheus the field will be replaced as kubernetes_namespace	..Try executing coredns_build_info metric in prometheus	  
  
        - source_labels: [__meta_kubernetes_namespace]  
          action: replace  
          target_label: kubernetes_namespace  
		    
// the below block means all __meta_kubernetes_service_name will be changed to kubernetes_name  
// e.g. __meta_kubernetes_service_name='kube-dns' to kubernetes_name='kube-dns' i.e when you query in prometheus the field will be replaced as kubernetes_name	..Try executing coredns_build_info metric in prometheus	  
        - source_labels: [__meta_kubernetes_service_name]  
          action: replace  
          target_label: kubernetes_name  
	  
```
