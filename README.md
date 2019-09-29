# Cluster best practices to build and manage applications on Azure Kubernetes Services (AKS)

These best practices and conceptual article is summarized from [these Azure Documentation](https://docs.microsoft.com/en-us/azure/aks/best-practices). These areas include multi-tenancy and scheduler features, cluster and pod security, business continuity and disaster recovery or storage and backups.

## Infrastructure best practices

### Multi-tenancy

- **[Best practices for cluster isolation](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation)**
    - Includes multi-tenancy core components and logical isolation with namespace.
    - Use logical isolation to separate teams, environment, workloads and projects. Try to minimize the number of physical AKS clusters you deploy to isolate teams or applications. 
        - Kubernetes [Namespaces](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads#namespaces) form the logical isolation boundary for workloads and resources.
        - Logical separation of clusters usually provides a higher pod density than physically isolated cluster. Less excess compute capacity that sits idle in the cluster.
        - With this best practices approach to autoscalling lets you run only the number of nodes required and minimizes cost.

            ![Cluster Isolation](https://drive.google.com/uc?export=view&id=1F7LHu7dvEdQxfRTPsncwij105FQ4Us6u "Kuberneter Cluster Isolation") 

- **[Best practices for basic scheduler](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-scheduler)**
    - Includes resource quotas and pod disruption budget.
    - Plan and apply the `resource quota` at the namespace level. 
        - Reject the deployment, if pods don't define resource requests and limits.
        - Basic example YAML manifest named `dev-team-resourcequota.yaml` sets a hard limit of a total of *20* CPUs, *32Gi* of memory, and *20* pods:
            ```yaml
            apiVersion: v1
            kind: ResourceQuota
            metadata:
            name: dev-team-resourcequota
            spec:
            hard:
                cpu: "20"
                memory: 32Gi
                pods: "20"
            ```

        - The `resource quota` above can be applied by specifying the namespace, such as `DevTeam1`:
            ```bash
            kubectl apply -f dev-team-resourcequota.yaml --namespace DevTeam1
            ```
    - Define `Pod Disruption Budgets (PDBs)` to make sure that a minimum number of pods are available in the cluster.
        - Let's look at an example of a replica set with 5 pods that run NGINX. The pods in the replica set as assigned the label `app: nginx-frontend`. During a cluster upgrade, you want to make sure at least *3* pods continue to run. The following YAML manifest for a `PodDisruptionBudget` object defines these requirements:

            ```yaml
            apiVersion: policy/v1beta1
            kind: PodDisruptionBudget
            metadata:
                name: nginx-pdb
            spec:
                minAvailable: 3
                selector:
                matchLabels:
                    app: nginx-frontend
            ```

        - You also can make sure a maximum number of unavailable instances. The following pod disruption budget YAML manifest defines that no more than *40%* pods in the replica set be unavailable:
        
            ```yaml
            apiVersion: policy/v1beta1
            kind: PodDisruptionBudget
            metadata:
                name: nginx-pdb
            spec:
                maxUnavailable: 40%
                selector:
                matchLabels:
                    app: nginx-frontend
            ```

        - Once your pod disruption budget is defined, you create it in your AKS cluster as with any other Kubernetes object:
            ```bash
            kubectl apply -f nginx-pdb.yaml
            ```
    - Monitor resource usage and adjust quotas as needed.
        - Regularly check for missing pod resource requests and limits using a diagnostic tool for Kubernetes clusters.
        - Run [`kube-advisor`](https://github.com/Azure/kube-advisor), it is an associated AKS open source project that scans a Kubernetes cluster and reports on issues that it finds.

- **[Best practices for advanced scheduler](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-advanced-scheduler)**
    - Includes using taints and tolerations, node selector and affinity, inter-pod affinity and anti-affinity.
    - Use `taints` (applied to a node that indicates only specific pods can be scheduled on them) and `tolerations` (applied to a pod that allows them to tolerate a node's taint) to limit what pods can be scheduled on nodes.
    - Give preference to pods to run on certain nodes with node selectors or node affinity.
    - Split apart or group together pods with inter-pod affinity or anti-affinity.

- **[Best practices for authentication and authorization](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-identity)**
    - Includes integration with Azure AD, using role-based access controls (RBAC) and pod identities.
    - Authenticate AKS cluster users with Azure Active Directory. Kubernetes doesn't provide an identity management solution to control which user can interact with resources. Typically integrate your cluster with an existing identiry solution. For example use [Azure Active Directory](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-identity#use-azure-active-directory):

        ![Cluster authentication and authorization](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-identity/cluster-level-authentication-flow.png "Cluster authentication and authorization")

    - Control access to resources with Kubernetes role-based access controls (RBAC). 
        - In Kubernetes, you can provide granular control of access to resources in the cluster. Permissions can be defined at the cluster level, or to specific namespaces. 
        - Kubernetes RBAC can be used in conjunction with Azure AD-integration, as define above.
        - As an example, you can create a `Role` that grants full access to resources in the namespace named `finance-app`, as shown in the following example YAML manifest:

            ```yaml
            kind: Role
            apiVersion: rbac.authorization.k8s.io/v1
            metadata:
                name: finance-app-full-access-role
                namespace: finance-app
            rules:
            - apiGroups: [""]
            resources: ["*"]
            verbs: ["*"]
            ```

        - A `RoleBinding` is then created that binds the Azure AD user `developer1@contoso.com` to the `RoleBinding`, as shown in the following YAML manifest:

            ```yaml
            kind: RoleBinding
            apiVersion: rbac.authorization.k8s.io/v1
            metadata:
                name: finance-app-full-access-role-binding
                namespace: finance-app
            subjects:
            - kind: User
                name: developer1@contoso.com
                apiGroup: rbac.authorization.k8s.io
            roleRef:
                kind: Role
                name: finance-app-full-access-role
                apiGroup: rbac.authorization.k8s.io
            ```
    
    - Use a managed identity to authenticate themselves with other services
        -  Don't use fixed credentials within pods or container images, as they are at risk of exposure or abuse.

### Security
- **[Best practices for security and updates](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-security)**
    - Including access to API server, limiting container access, and managing upgrades and node reboots.

    - Use Azure Active Directory and role-based access controls to secure API server access.
        - Securing access to the Kubernetes API-Server is one of the most important things you can do to secure your cluster.
        - Integrate Kubernetes RBAC with Azure AD to control access to the Kubernetes API server.

            ![Azure AD-integrated clusters in AKS](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-cluster-security/aad-integration.png "Azure AD-integrated clusters in AKS")

        - Use `groups` to provide access to resources versus individual identities, use Azure AD `group` membership to bind users to RBAC roles rather than individual users. As a user's group membership changes, their access permissions on the AKS cluster would change accordingly.

    - Secure container access to node resources.
        - Limit access to actions that containers can perform.
            - Provide the least number of permissions and avoid use of root / privileged escalation. For example, set `allowPrivilegeEscalation: false` in the pod manifest. 
        - To limit the actions that containers can perform, you can use the [`AppArmor`](https://kubernetes.io/docs/tutorials/clusters/apparmor/) Linux kernel security module. 
        - While `AppArmor` works for any Linux application, [`seccomp` (secure computing)](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#seccomp) works at the process level. 

    - Upgrade an AKS cluster to the latest Kubernetes version.
        - To stay current on new features and bug fixes, regularly upgrade to the Kubernetes version in your AKS cluster.

    - Keep nodes update to date and automatically apply security patches.
        - AKS automatically downloads and installs security fixes on each Linux nodes, but does not automatically reboot if necessary. Use [`kured`](https://github.com/weaveworks/kured) to watch for pending reboots, then safely cordon and drain the node to allow the node to reboot. `kured` can integrate with Prometheus to prevent reboots if there are other maintenance events or cluster issues in progress. 

- **[Best practices for container and image management security](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-container-image-management)**
    - Includes securing the image and runtime and automated builds on base image updates.
    - Scan for and remediate image and run time vulnerabilities.
        - Scan your container images for vulnerabilities, and only deploy images that have passed validation.
        - Regularly update the base images and application runtime, then redeploy workloads in the AKS cluster.
        - In a real-world example, you can use a continuous integration and continuous deployment (CI/CD) pipeline to automate the image scans, verification, and deployments. Azure Container Registry, [Twistlock](https://www.twistlock.com/) or [Aqua](https://www.aquasec.com/) include these vulnerabilities scanning capabilities. Then only allow verified images to be deployed.
    - Automatically trigger and redeploy container images when a base image is updated.
        - This build and update process should be integrated into validation and deployment pipelines such as Azure Pipelines or Jenkins.
        - These pipelines makes sure that your applications continue to run on the updated based images. 
        - Once your application container images are validated, the AKS deployments can then be updated to run the latest, secure images.

- **[Best practices for pod security](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security)**
    - Includes securing access to resources, limiting credential exposure, and using pod identities and digital key faults.

    - Use pod security context to limit access to processes and services or privilege escalation.
        - To run as a different user or group and limit access to the underlying node processes and services, define pod security context settings. Assign the least number of privileges required.
        - Pods should run as a defined user or group and not as `root`.
        - The following example pod YAML manifest sets security context settings to define: Pod runs as user ID *1000* and part of group ID *2000*. Can't escalate privileges to use `root`. Allows Linux capabilities to access network interfaces and the host's real-time (hardware) clock.

            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
                name: security-context-demo
            spec:
                containers:
                    - name: security-context-demo
                    image: nginx:1.15.5
                    securityContext:
                        runAsUser: 1000
                        fsGroup: 2000
                        allowPrivilegeEscalation: false
                        capabilities:
                            add: ["NET_ADMIN", "SYS_TIME"]
            ```

    - Authenticate with other Azure resources using pod managed identities.
        - Don't define credentials in youe application code.
        - Use managed identities for Azure resources to let your pod request access to other resources.

    - Request and retrieve credentials from a digital vault such as Azure Key Vault.
        - A digital vault, such as Azure Key Vault should also be used to store and retrieve digital keys and credentials.


### Network and storage

- **[Best practices for network connectivity](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-network)**
    - Includes different network models, network policies to control the flow of traffic in and out of pods, using ingress and web application firewall (WAF) and securing node SSH access.
    
    - Choose the appropriate network model, Azure Container Networking Interface (CNI) or Kubenet Networking.
        - `Kubenet` is suitable for small development or test workloads, as you don't have to create the virtual network and subnets separately from the AKS cluster. Simple websites with low traffic, or to lift and shift workloads into containers, can also benefit from the simplicity of AKS clusters deployed with `kubenet` networking. For most production deployments, you should plan for and use Azure CNI networking. Azure CNI lets you connect to existing Azure resources, on-premise resources, or other services directly via IP addresses assigned to each pod.

            ![Container Networking Interface (CNI)](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-network/advanced-networking-diagram.png "Container Networking Interface (CNI)")

    - Distribute Kubernetes ingress traffic.
        - To distribute HTTP or HTTPS traffic to your applications, use ingress resources and controllers.
        - Ingress controllers provide additional features over a regular Azure load balancer, and can be managed as native Kubernetes resources. 
            - A `load balancer` resource work at layer 4, and distributes traffic based on protocol or ports.
            - Most web applications that use HTTP or HTTPS should use `Kuberenetes ingress` resources and controllers, which work at layer 7. Ingress can distribute traffic based on the URL of the application and handle TLS/SSL termination. 

                ![Kubernetes Ingress](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-network/aks-ingress.png "Kubernetes Ingress")

        - Kubernets ingress has two components: ingress `resource` and `controller`. The ingress resource is a YAML manifest of `kind: Ingress`.
        - The following example YAML manifest would distribute traffic for myapp.com to one of two services, blogservice or storeservice:

            ```yaml
            kind: Ingress
            metadata:
            name: myapp-ingress
                annotations: kubernetes.io/ingress.class: "PublicIngress"
            spec:
            tls:
            - hosts:
                - myapp.com
                secretName: myapp-secret
            rules:
                - host: myapp.com
                http:
                    paths:
                    - path: /blog
                    backend:
                        serviceName: blogservice
                        servicePort: 80
                    - path: /store
                    backend:
                        serviceName: storeservice
                        servicePort: 80
            ```

        - The most common ingress controller is based on NGINX. AKS doesn't restrict you to a specific controller, so you can use other controllers such as Contour, HAProxy, or Traefik.

    - To scan incoming traffic for potential attacks, use a web application firewall (WAF) such as Barracuda WAF for Azure or Azure Application Gateway. These more advanced network resources can also route traffic beyond just HTTP and HTTPS connections or basic SSL termination.

        ![Application Gateway WAF](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-network/web-application-firewall-app-gateway.png "Application Gateway WAF")

    - Use network policies to allow or deny traffic to pods. By default, all traffic is allowed between pods within a cluster. For improved security, define rules that limit pod communication.

        - Network policy is a Kubernetes feature that lets you control the traffic flow between pods, this feature must be enabled when you create an AKS cluster. You can allow or deny traffic based on settings such as assigned label, namespace, or traffic port. Don't use Azure network security groups to control pod-to-pod traffic, use network policies.

        - The following example applies a network policy to pods with the `app: backend` label applied to them. The ingress rule then only allows traffic from pods with the `app: frontend` label:

            ```yaml
            kind: NetworkPolicy
            apiVersion: networking.k8s.io/v1
            metadata:
            name: backend-policy
            spec:
            podSelector:
                matchLabels:
                app: backend
            ingress:
            - from:
                - podSelector:
                    matchLabels:
                    app: frontend
            ```
    
    - Don't expose remote connectivity to your AKS cluster. Create a bastion host, or jump box, in a management virtual network. Use the bastion host to securely route traffic into your AKS cluster to remote management tasks.

        ![Bastion host connection](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-network/connect-using-bastion-host-simplified.png "Bastion host connection")

- **[Best practices for storage and backups](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-storage)**
    - Includes choosing the appropriate storage type and size, dynamically provisioning volumes and data backups.

    - Choose the appropriate storage type.
        - Use high performance, SSD storage for production workloads.
        - Plan for network-based storage when there is a need for multiple concurrent connections.

        | Use case       | Volume plugin           | Read/write once  | Read-only many       | Read/write many           |
        |-------------|-------------|--------------------------|-------------|-------------|
        |Shared configuration|Azure Files|Yes|Yes|Yes|
        |Structured app data|Azure Disks|Yes|No|No|
        |Unstructured data, file system operations|BlobFuse (preview)|Yes|Yes|Yes|

        - The type of storage you use is defined using Kubernetes `storage class`. The `storage class` is then referenced in the pod or deployment specification. The following example uses Premium Managed Disks and specifies that the underlying Azure Disk should be retained when the pod is deleted:

            ```yaml
            kind: StorageClass
            apiVersion: storage.k8s.io/v1
            metadata:
            name: managed-premium-retain
            provisioner: kubernetes.io/azure-disk
            reclaimPolicy: Retain
            parameters:
            storageaccounttype: Premium_LRS
            kind: Managed
            ```

    - Size the nodes for storage needs.
        - Each node size supports a maximum number of disks.
        - Different node sizes also provide different amounts of local storage and network bandwidth.
        - Plan for your application demands to deploy the appropriate size of nodes.

    - Dynamically provision volumes.
        - To reduce management overhead and let you scale, don't statically create and assign persistent volumes. Use dynamic provisioning.
        - A persistent volume claim (PVC) lets you dynamically create storage as needed. The following example YAML manifest shows a persistent volume claim that uses the managed-premium StorageClass and requests a Disk 5Gi in size:

            ```yaml
            apiVersion: v1
            apiVersion: v1
            kind: PersistentVolumeClaim
            metadata:
            name: azure-managed-disk
            spec:
            accessModes:
            - ReadWriteOnce
            storageClassName: managed-premium
            resources:
                requests:
                storage: 5Gi
            ```

        - In your `storage classes`, define the appropriate reclaim policy to minimize unneeded storage costs once pods are deleted. The `reclaimPolicy` can set to `retain` or `delete`. 

        ![Presistent Volume Claim](https://docs.microsoft.com/en-us/azure/aks/media/concepts-storage/persistent-volume-claims.png "Presistent Volume Claim")

    - Secure and back up your data.
        - Back up your data using an appropriate tool for your storage type, such as Velero or Azure Site Recovery. 
        - Verify the integrity, and security, of those backups.
            - Velero can back up persistent volumes along with additional cluster resources and configurations.
            - If you can't remove state from your applications, back up the data from persistent volumes and regularly test the restore operations to verify data integrity and the processes required.

### Running enterprise-ready workloads

- **[Best practices for business continuity and disaster recovery](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-multi-region)**
    - Includes using region pairs, multiple clusters with Azure Traffic Manager and geo-replication of container images registry.
    - When you deploy multiple AKS clusters, choose regions where AKS is available, and use paired regions.
        - When you deploy your application, add another step to your CI/CD pipeline to deploy to these multiple AKS clusters.
    - Azure Traffic Manager can direct customers to their closest AKS cluster and application instance. For the best performance and redundancy, direct all application traffic through Traffic Manager before it goes to your AKS cluster endpoint (*service load balancer IP*).

        ![AKS Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-bc-dr/aks-azure-traffic-manager.png "AKS Azure Traffic Manager")

        - Traffic Manager (uses DNS (layer 3) to shape traffic) performs DNS lookups and returns a user's most appropriate endpoint. Nested profiles can prioritize a primary location. 

            ![Azure Traffic Manager Geo Routing](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-bc-dr/traffic-manager-geographic-routing.png "AKS Azure Traffic Manager Geo Routing")

    - Store your container images in Azure Container Registry and geo-replicate the registry to each AKS region.
    - Where possible, don't store service state inside the container. Instead, use an Azure platform as a service (PaaS) that supports multiregion replication.
        - Containers and microservices are most resilient when the processes that run inside them don't retain state.
        - Because applications almost always contain some state, use a PaaS solution such as Azure Database for MySQL, Azure Database for PostgreSQL, or Azure SQL Database.
    - If you use Azure Storage, prepare and test how to migrate your storage from the primary region to the backup region.
        - Infrastructure-based asynchronous replication
            - In Kubernetes, you can use persistent volumes to persist data storage.
            - Persistent volumes are mounted to a node VM and then exposed to the pods.
            - Persistent volumes follow pods even if the pods are moved to a different node inside the same cluster.
            - Common storage solutions such as Gluster, Ceph, Rook, and Portworx.
                - The typical strategy is to provide a common storage point where applications can write their data. This data is then replicated across regions and then accessed locally.

                    ![AKS Infra Based Async Replication](https://docs.microsoft.com/en-us/azure/aks/media/operator-best-practices-bc-dr/aks-infra-based-async-repl.png "AKS Infra Based Async Replication")

        - Application-based asynchronous replication
            - Kubernetes currently provides no native implementation for application-based asynchronous replication.
            - Typically, the applications themselves replicate the storage requests, which are then written to each cluster's underlying data storage.

## Developer best practices

Simplify the development experience and define require application performance needs.

- **[Best practices for application developers to manage resources](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-resource-management)**
    - Includes defining pod resource requests and limits, configuring development tools and checking for application issues.
    - Set pod requests and limits on all pods in your YAML manifests. If the AKS cluster uses resource quotas, your deployment may be rejected if you don't define these values.
    - Development teams should deploy and debug against an AKS cluster using [Azure Dev Spaces](https://docs.microsoft.com/en-us/azure/dev-spaces/). This development model makes sure that role-based access controls, network, or storage needs are implemented before the app is deployed to production.
    - Install and use the [VS Code extension for Kubernetes](https://github.com/Azure/vscode-kubernetes-tools) when you write YAML manifests. You can also use the extension for integrated deployment solution, which may help application owners that infrequently interact with the AKS cluster.
    - Regularly run the latest version of `kube-advisor` open source tool to detect issues in your cluster. If you apply resource quotas on an existing AKS cluster, run `kube-advisor` first to find pods that don't have resource requests and limits defined.

- **[Best practices for pod security](https://docs.microsoft.com/en-us/azure/aks/developer-best-practices-pod-security)**
    - Includes securing access to resources, limiting credential exposure, and using pod identities and digital key faults.