start k3s cluster in Docker
```bash
K3S_TOKEN=${RANDOM}${RANDOM}${RANDOM} docker compose -p k3s up -d
```
take kubeconfig to interact with k3s cluster and apply httpbin deployment
```bash
docker exec k3s-server-1 cat /etc/rancher/k3s/k3s.yaml > kubeconfig
export KUBECONFIG=$PWD/kubeconfig

kubectl apply -f httpbin/
```
to check that service is working need folow next link http://localhost/

## Description:
a cluster was needed for the test, it would be possible to use terraform and some cloud, but for the reviewer it is extra work and cost, i decided that k3s in docker is the best solution, it can be brought up anywhere and the result can be checked. i took Docker compouse from the official repo and added +1 agent for the test https://github.com/k3s-io/k3s/blob/main/docker-compose.yml

i chose kubernetes version 1.33, because in 1.34 full link for docker image is required and can add extra work and problems.

i chose a regular Deployment set of files without Helm, in prod i would use a gitops approach
i split the files into different files for readability, in reality it is better to use Kustomize, but the task is not about that.

to start i created a separate namespace, because it is a level of isolation, it would be possible to add resource limitation for the namespace, but the task says it should be simple and i decided not to add it
after that i went to the docker regestry to look at the container versioning for httpbin, so as not to use latest, but there are only test and latest

for Service i use 8080, so as not to use the default 80. and i map it to port 80 in the container because gunicorn starts there on 80.

i add hpa for horizontal scaling with a simple rule avg 70% cpu, but it will not work in this example because metrics server is not installed in k3s, but the task is not about that

it is necessary to expose the application outside, since the traefic ingress controller exists, i add kind ingress and map it to the Service.

i create Deployment,
in metadate nothing extra, only the application name and namespace. i specify 3 replicas(3 nodes on which pods can be assigned, in k3с the taint is removed from controll plane), i match the replicaset to the deployment,
i add resources for the pod, i specify requests cpu and ram and limit only ram, because in best practices it is better not to limit cpu if it is not needed, cpu trotling can happen, it is better to leave cpu queue planning to the linux core, i add probe.
then i will do security, we will run the process not as root, but as a specific user 1000, we will make our filesystem only readble and forbid privilege escalation
the task has a requirement that it be evenly distributed across the kubernetes cluster, which means it is necessary to use topologySpreadConstraints,
i write a rule that for each nodes the difference in pods is no more than 1 and a soft rule that if we did not meet the criteria above still scale the pod.
after apply pods are in state crash because the application lacks a readonly filesystem, after googling i understand that gunicorn needs worker_tmp_dir, i will add emptyDir /tmp and limit it in size.
after successful apply, i check kubectl -n playson-task get po -o wide, i see 3 pods in state running on different nodes

the task is completed, the solution is minimal and clean, security practices are added and the pods moved out to different nodes
