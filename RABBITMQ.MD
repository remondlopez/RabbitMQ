Here is a clear documentation of the steps we used to set up, verify, and secure your RabbitMQ Cluster across multiple namespaces. You fit share this one with your team or keep am for your records.
RabbitMQ Cluster Management Documentation
1. Architecture Overview
Infrastructure: Kubernetes Cluster.
Management Layer: RabbitMQ Cluster Operator (running in rabbitmq-system).
Message Broker: RabbitMQ Cluster (3-node HA setup) running in rabbitmq-system.
Client Access: Apps from 5 different namespaces (e.g., agent-captive) connect to this central cluster.
2. Cluster Health Verification
To confirm that the "motors" (RabbitMQ nodes) are healthy and talking to each other:
Command:
powershell
kubectl exec -it rabbitmq-server-0 -n rabbitmq-system -- rabbitmqctl cluster_status
Use code with caution.
Expected Result:
All 3 nodes (0, 1, 2) should be listed under Disk Nodes and Running Nodes.
Alarms should be (none).
3. Cross-Namespace Connectivity Test
Because the apps stay in different rooms (namespaces) than the RabbitMQ server, we verified the "road" was open.
Testing from App Namespace (agent-captive):
Using an existing pod (e.g., springboot-test-app...), we ran a network check:
powershell
kubectl exec -it <pod-name> -n agent-captive -- nc -zv rabbitmq.rabbitmq-system.svc.cluster.local 5672
Use code with caution.
Verification: Result showing "open" confirms that DNS and Network Policies allow traffic.
4. Multi-Tenant Configuration (Isolation)
To ensure teams don't mix their messages, we used Virtual Hosts (vhosts) and Dedicated Users.
A. Creating the "VIP Room" (vhost)
Each namespace gets its own isolated area.
powershell
kubectl exec -it rabbitmq-server-0 -n rabbitmq-system -- rabbitmqctl add_vhost agent_captive_vhost
Use code with caution.
B. Creating the Developer User
Avoid using the admin account for applications.
powershell
kubectl exec -it rabbitmq-server-0 -n rabbitmq-system -- rabbitmqctl add_user captive_dev_user <PASSWORD>
Use code with caution.
C. Setting Permissions
This locks the user to only their specific vhost.
powershell
kubectl exec -it rabbitmq-server-0 -n rabbitmq-system -- rabbitmqctl set_permissions -p agent_captive_vhost captive_dev_user ".*" ".*" ".*"
Use code with caution.
5. Developer Connection Cheat-Sheet
Provide these details to the developers in the agent-captive namespace:
Parameter	Value
Broker Host	rabbitmq.rabbitmq-system.svc.cluster.local
Port	5672 (AMQP) / 15672 (UI)
Username	captive_dev_user
Virtual Host	agent_captive_vhost
6. Useful Troubleshooting Commands
Check Operator Logs: kubectl logs -l app.kubernetes.io/name=rabbitmq-cluster-operator -n rabbitmq-system
List all VHosts: kubectl exec -it rabbitmq-server-0 -n rabbitmq-system -- rabbitmqctl list_vhosts
List all Users: kubectl exec -it rabbitmq-server-0 -n rabbitmq-system -- rabbitmqctl list_users
Access Dashboard: kubectl port-forward -n rabbitmq-system svc/rabbitmq 15672:15672 (Then visit http://localhost:15672)
You want make I add the Resource Limit (CPU/RAM) recommendations to this document so the cluster no go "choke" when load plenty?





yes
Adding Resource Limits is very important. Without them, one greedy app can "choke" the whole Kubernetes node and make other pods (or the RabbitMQ itself) crash.
Here is the section you should add to your documentation to keep the cluster stable:
7. Resource Limits & Quotas (Stability Guide)
To prevent the "Noisy Neighbor" effect, we must set boundaries for the RabbitMQ pods.
A. Recommended Settings per Node
For a standard production-ready cluster, these are the "Sweet Spot" settings:
Resource	Request (Guaranteed)	Limit (Max allowed)
CPU	500m (0.5 core)	2000m (2 cores)
Memory	2Gi	4Gi
B. Why Memory Limits are Critical
RabbitMQ has a Memory High Watermark (default 40%).
If the pod has 4GB RAM, RabbitMQ will stop accepting messages once it uses 1.6GB.
Warning: If you don't set a limit, RabbitMQ might think it has the whole node's memory, get killed by the Kubernetes OOM (Out Of Memory) Killer, and cause a "Reboot Loop."
C. How to apply these in your YAML
Update your RabbitmqCluster manifest with this block:
yaml
spec:
  resources:
    requests:
      cpu: "500m"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  rabbitmq:
    additionalConfig: |
      # This tells RabbitMQ exactly how much RAM it has
      # (Set this to 40-50% of your memory limit)
      vm_memory_high_watermark.relative = 0.4
Use code with caution.
D. Disk Space (Persistence)
Storage Class: Use a Fast SSD (like managed-csi or premium-ssd for Azure).
Size: Minimum 10Gi to 50Gi per node. If the disk fills up, RabbitMQ will stop accepting messages (Disk Alarm).
Pro-Tip for Monitoring
If you want to see if the cluster is "choking," run this:
powershell
kubectl top pods -n rabbitmq-system
Use code with caution.
(If this command no work, e mean say you never install Metrics Server for your cluster).
You want make I explain how to automatically scale the disk if e don full?
