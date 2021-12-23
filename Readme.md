# Guardrails DevOps Take Home Challenges
## Questions for the team
About the architecture design:
- When a user triggers a scan, will Scan Job need the whole repo or just the diff? I'm assuming the whole repo is needed because the user trigger a scan via Dashboard.
- Can these services scale horizontally? If I understand it correctly, Scan Job is a short live process created by Worker service on demand, the other services are stateless services that can scale out horizontally.
- When multiple Scan Jobs work on a repo, do Data Processing service need to wait for all Scan Jobs for their results to start processing?
- If Scan Jobs send the result to stdout, how do Data Processing get this result? I guess it calls K8s API?
- What tools are currently used for monitoring, logging and tracing?
- What version of EKS is currently used? Do the team have any problem upgrading it?

## Architectural Challenge
### Question 1
How would you improve the current design to achieve better:
- High availability
- Resilience
- Performance
- Cost efficiency

Let's assume that all services are deployed on AWS EKS, including Dashboard, API, Storage, MQ, Worker, Scan, Data Processing, Notification.
With just the high level product workflow and without any further info, I can hardly make any improvement. Instead of changing the current architecture design, I'll list out a few operation related items that may help achieving a right balance of the above quality:

1. Infrastructure as Code
    - Prefer state definitive tools over ad-hoc API or CLI tools
    - Provision EKS and related AWS services with tools such as Terraform instead of AWS Console or ad-hoc API calls
    - Manage Kubernetes objects (deployment, services, ingresses, etc.) with helm and helmfile instead of kubectl
    - Manage this code with Git
    - Make frequent, small and reversible changes
    - Build the CI/CD pipeline to apply changes automatically
2. Learn from failures
    - Anticipate failures
    - Learn from failures that happened
3. Redundancy, self healing, auto scaling
    - Except for Scan, all other long live stateless services should be deployed as Deployment, which will manage Pods automatically. Never deploy Pod directly.
    - Number of replicas should be greater than 1
    - Use multiple AZs, and multiple regions if needed and possible
    - Auto scale the Pods horizontally if possible (with HPA)
    - Auto scale the Nodes horizontally if possible (with cluster autoscaler)
    - Deploy RabbitMQ as a cluster if possible, with HA policy for queues
4. Test recovery procedures
5. Measurement and continuous improvement
    - Collect logs and metrics and analyze this data to find bottlenecks and inefficient parts of the system (either use managed services such as Datadog, or self hosted services, a combination of Grafana, Prometheus, Jaeger, ELK)
6. Spend money and time on things that directly contribute to business value
    - Consider using Amazon MQ for RabbitMQ to replace self managed RabbitMQ
    - Analyze spending data to stop spending money on unnecessary resources and services
Following AWS Well Architected Framework may help a lot.

My suggestion for architectural design change is to separate business code from platform code:
    - Worker get the jobs from MQ, instead of calling K8s API to create Scan job, it do the scan itself
    - Data Processing service get the scan result via event streaming service like Kafka instead of calling K8s API to get Scan job stdout
    - If it makes sense, replace RabbitMQ with Kafka for Scan job MQ to simplify the tech stack
I think the service for running the business shouldn't know that it's being deployed on K8s. Avoid calling K8s API will make deploying the services on different distro of K8s and different versions of K8s much easier. These services can be even deployed on a bunch of VM or EC2 instances if needed.
Integration with K8s using K8s API or K8s webhook or CRD should be reserved for generic helper services that add value to K8s itself.


### Question 2
The number of scan requests can increase/decrease randomly in a day and on most weekends the system receives almost no requests at all.

What strategy would you suggest to save cost while still maintaining the possible performance and scan completion times?

I'd recommend autoscaling pods based on the number of jobs pending in MQ (autoscaling with custom metrics is available with K8s >= 1.23), and autoscaling nodes based on resource utilization as usual. This autoscaling strategy is reactive.
Another autoscaling strategy is proactive, setting different minimum number of idle pods in week days and weekend. By studying the usage pattern (from collected metrics), we can set the appropriate number of minimum pods to ensure best performance.
On the other hand, optimizing startup speed of the services to a certain point will benefit autoscaling greatly.
And if the architecture change with my suggestion above, i.e refactor Scan job to become long live stateless service that get the job from MQ, and send the result to Kafka; then autoscaling can be done with CPU alone, using HPA v1.

### Question 3
In step 6, each job needs to mount the source code folder into every engine that needs to run.

How would you store the source code and make sure that engines can run in a scalable way?

Using AWS EFS or equivalent shared file system, each source code repo can be pulled into a directory. Multiple Scan job can mount the same source code directory from EFS. Since EFS can scale indefinitely in term of storage and throughput, the heavy lifting is already done by AWS. To save storage cost, a cleanup jobs can check for inactive source code repos in a certain amount of time and delete those repos.
If my suggestion for source code storage is feasible, it'll make the problem in the latter technical challenge disappear without any code. I think the nature of that challenge is caching for the source code repos. Since we use shared file system, we create a mirror for each repo by pulling new code once. Scan job can be scheduled on any node without putting any more stress on the origin source code repos.

### Question 4
Propose a high level disaster recovery plan for the current architecture

My strategy for building a DR plan is to pick a service/function in the system, and ask the following questions:
- What happen if this fail?
- How would this fail?
- How to mitigate when this fail?
- What need to be backup?
- How do we recreate a working system with what we have (backup, source code, etc.)

Repeat asking these question until the answer to the first question become "business as usual, we can go back to sleep".
Usually, the most important thing to backup, with offsite backup or multi regional backup, are database, user generated data, source code and documentation. The source code should contain all the application code, infrastructure code, CI/CD pipeline code, and even the documentation.

## Technical Challenge
My solution for this challenge is to write a helm chart, please take a look at [batch-jobs-min-nodes](./batch-jobs-min-nodes). A values file for testing purpose is included as well: [here](./test_values.yaml).

To run the helm chart with test:

    helm install <release_name> batch-jobs-min-nodes --values test_values.yaml
