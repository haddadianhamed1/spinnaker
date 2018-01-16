# spinnaker

The PipeLine

git repository  --> docker build --> docker push --> spinnaker Pipeline --> Kubernetes Deploy --> App Live

commit   ------->       trigger from github   ---------->    Trigger from docker Registery

# spinnaker build

* Make changes to spinnaker.yaml

* helm install --name test -f spinnaker.yaml stable/spinnaker

* Git and docker

* Setup dockerhub account

* Link dockerhub account with github

* Create new automated build

# spinnaker configuration
* Create a new application

* Create new loadbalancer

* Create new server group


```
step 0 ) create cluster and verify
# list kops clusters
AWS_PROFILE=personal kops get clusters --state=s3://kops-state-b4292

# delete Cluster
AWS_PROFILE=personal kops delete cluster spinnaker.hamedhaddadian.com --state=s3://kops-state-b4292 --yes

# create Cluster
AWS_PROFILE=personal kops create cluster --name=spinnaker.hamedhaddadian.com --state=s3://kops-state-b4292 --zones=us-west-1b,us-west-1c --node-count=2 --node-size=t2.large --master-size=t2.large --dns-zone=hamedhaddadian.com --master-zones=us-west-1b --ssh-public-key=/Users/hamedh/.ssh/id_rsa_kub.pub


AWS_PROFILE=personal kops update cluster spinnaker.hamedhaddadian.com --yes --state=s3://kops-state-b4292

* validate cluster: kops validate cluster
* list nodes: kubectl get nodes --show-labels
* ssh to the master: ssh -i ~/.ssh/id_rsa admin@api.spinnaker.hamedhaddadian.com

# check helm packages
helm ls
helm delete --purge hamed3

# verify kubernetes is setup
kubectl get nodes
kubectl get deployments


step 1 ) verify helm is configured
helm init

step 2) install spinnaker
helm install --name spinnaker -f spinnaker.yml stable/spinnaker

# verify it works
kubectl get pods
NAME                                               READY     STATUS    RESTARTS   AGE
spinnaker-jenkins-1300962005-k5d51                 1/1       Running   0          4m
spinnaker-minio-1433815553-qdtq2                   1/1       Running   0          4m
spinnaker-redis-849287136-jg7h4                    1/1       Running   0          4m
spinnaker-spinnaker-clouddriver-1402871711-wt18m   1/1       Running   0          4m
spinnaker-spinnaker-deck-1475044543-3v3r5          1/1       Running   0          4m
spinnaker-spinnaker-echo-4029422651-8fdw0          1/1       Running   0          4m
spinnaker-spinnaker-front50-2502198926-04jpv       1/1       Running   0          4m
spinnaker-spinnaker-gate-1546656221-6rk0l          1/1       Running   1          4m
spinnaker-spinnaker-igor-2117579405-6qd3j          1/1       Running   0          4m
spinnaker-spinnaker-orca-1279637590-xnhf6          1/1       Running   0          4m
spinnaker-spinnaker-rosco-2462873569-0qqq1         1/1       Running   0          4m


step 3) login to spinnaker
NOTES:
1. You will need to create 2 port forwarding tunnels in order to access the Spinnaker UI:
  export DECK_POD=$(kubectl get pods --namespace default -l "component=deck,app=spinnaker-spinnaker" -o jsonpath="{.items[0].metadata.name}")
  kubectl port-forward --namespace default $DECK_POD 9000

2. Visit the Spinnaker UI by opening your browser to: http://127.0.0.1:9000

step 4) configure github and dockerhub
I am going to link my dockerhub to my github repo with the app
lets use this repository
https://github.com/haddadianhamed1/docker

step A) link dockerhub and github
dockerhub > settings > linked accounts & services > link github > public and private
step B) create Automated bill from dockerhub
create > create automated build > from github > choose docker repo > new repository name: spinnaker-node-demo ( reminder this need to be the same as the yaml file we used) > create
step c) test the Trigger with git push or manual test
build settings > trigger  && check build details for job details


step 5) configure spinnaker
localhost:9000
step A) Create new application
Application > actions > create application > nodejs
from nodejs application in spinnaker
A) create a loadbalancer which is the services
stack: dev, targetport:3000, create
kubectl get svc
NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
nodejs-dev                        ClusterIP   100.64.158.13    <none>        80/TCP                        22s

B) after loadbalancer we have servergroup
clusters > create Server Group > stack: dev, containers: spinnaker-node-demo:latest(this is dockerhub build we created ), loadbalancer: nodejs-dev, container Port: 3000, add probes for verifying image works: probes: port:3000
note: containers are comming from dockerhub when we attached them to our github and ran a build

kubectl get pods
NAME                                               READY     STATUS    RESTARTS   AGE
nodejs-dev-v000-0j3w4                              1/1       Running   0          1m

kubectl get rs
NAME                                         DESIRED   CURRENT   READY     AGE
nodejs-dev-v000                              1         1         1         2m

# now to see if it is working
kubectl proxy # we can use loadbalancer also when we created our app but for testing we dont need a loadbalancer
#this will create a proxy to our API server
kubectl proxy
Starting to serve on 127.0.0.1:8001

# now if we go to this url
http://localhost:8001/api/v1/proxy/namespaces/default/services/nodejs-dev:80/

we see our HelloWorld.


C) Create a pipelines
pipelines > create > pipeline Name: trigger - deploy to dev
automated trigger > type :Docker Registery organization:hhaddadian image:spinnaker-node-demo
add stage > type: Deploy , stagename: deploy to dev, add server group> nodejs dev we created earlier , before adding change the image to tag resloved at runtime image > add

add one more stage > type destroy server group> check namespace > cluster nodejs-dev, target: Previous Server Group



D) Test
to test it we need to commit a dockerfile to github
git clone https://github.com/haddadianhamed1/docker.git
cd docker
git checkout -b v1.0.2
vi index.js and change something
Hello World!(v1.0.2)
git add, git commit, git push origin v1.0.2
# now github would let create a new branch, dokcerhub should do a dockerbuil, and after build is done spinnaker will deploy on kubernetes

kubectl get pods
NAME                                               READY     STATUS        RESTARTS   AGE
nodejs-dev-v000-0j3w4                              1/1       Terminating   0          28m
nodejs-dev-v001-h69gm                              1/1       Running       0          2m

# in dockerhub we should see a new trigger and if we go to proxy we should see the new deployment
http://localhost:8001/api/v1/proxy/namespaces/default/services/nodejs-dev:80/


```
