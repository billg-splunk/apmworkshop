## K8S Prep

You must have a ready Kubernetes cluster for this example.  
Hints on setting up a minimal cluster and local VM can be found in the [Appendix](../4-appendix.md).  

[Helm](helm.sh) must be installed.

Set `kubectl` context:    
`sudo kubectl config set-context --current --namespace=kube-system`

### All examples below take place in the `./apm/k8s/python` directory

#### K8S Step 1: set up the SignalFx SmartAgent as a sidecar pod  

```
helm repo add signalfx https://dl.signalfx.com/helm-repo
```
```
helm repo update
```
Build your Helm install script based on the following variables:

TOKENHERE: token from your account  
YOURREALMHERE: your realm from your account i.e. us1  
YOURK8SCLUSTERNAME: any name you pick to represent the cluster  
RELEASEVERSIONHERE: Use the current SignalFx SmartAgent version in the Helm script below from here: https://github.com/signalfx/signalfx-agent/releases i.e. 5.5.1

```
helm install --set signalFxAccessToken=TOKENHERE \
--set clusterName=YOURK8SCLUSTERNAME \
--set signalFxRealm=YOUREALMHERE \
--set agentVersion=RELEASEVERSIONHERE \
--set kubeletAPI.url=https://localhost:10250 signalfx-agent signalfx/signalfx-agent
```
Once you deploy the SmartAgent on your Kubernetes cluster, you will see the host appear within seconds in the Infrastructure Tab Kubnernetes Navigator in SignalFx.  
Check this and then move on to next step.

#### K8S Step 2: SmartAgent Update For Kubernetes     

If you are doing this workshop as part of a group:  
/apm/k8s/python (this directory) has an `agent.yaml` file with a default `environment` value.
Add a unique identifier to the `environment` name i.e. your initials `sfx-workshop-YOURINITIALSHERE`
For a group workshop The resulting stanza is:

```
monitors:
  - type: signalfx-forwarder
    listenAddress: 0.0.0.0:9080
    defaultSpanTags:
     environment: "sfx-workshop-YOURINITIALSHERE"
```

If you are NOT doing this workshop as part of a group, you can leave `agent.yaml` as is.

For K8S, use ```helm``` to reconfigure the agent pod with the enclosed `agent.yaml` additions to the `monitor` stanza:

`helm upgrade --reuse-values -f ./agent.yaml signalfx-agent signalfx/signalfx-agent`

#### K8S Step 3: Deploy the containerized instrumented Python examples 

Deploy the flask-server:

`source deploy-flask.sh`  
`source deploy-python-requests.sh`

#### K8S Step 4: Study the results

The APM Dashboard will show the instrumented Python-Requests Client posting to the Flask Server.  
Make sure you select the ENVIRONMENT to monitor on the selector next to `Troubleshooting` i.e. in image below you can see `sfx-workshop` is selected.

<img src="../../../assets/vlcsnap-00007.png"/>  

#### K8S Step 5: Study the `deployment.yaml` files

Spans need to be send to the SmartAgent which is running in its own pod- the `deployment.yaml` files will demonstrate this.

Normally we use `SIGNALFX_ENDPOINT_URL` pointing to `localhost` on a single host application where the SmartAgent is running.

In k8s we have separate pods in a cluster for apps and the SmartAgent.  
The SmartAgent pod is running with <ins>node wide visibility</ins>, so to tell each application pod where to send spans, we use this:

```
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
            - name: SIGNALFX_ENDPOINT_URL
              value: http://$(MY_NODE_NAME):9080/v1/trace
```


#### K8S Step 6: View trace spans flowing in SignalFx Agent pod
`kubectl get pods`

Note the pod name of the `SignalFx Agent` pod

`kubectl exec -it PODNAMEOFSIGNALFXAGENT -- bash -c "/bin/signalfx-agent status"`  

`signalfx-agent status` will show the metrics and spans being sent by the agent like this:

```
ubuntu@primary:~$ signalfx-agent status
SignalFx Agent version:           5.5.1
Agent uptime:                     1h31m32s
Observers active:                 host
Active Monitors:                  10
Configured Monitors:              10
Discovered Endpoint Count:        8
Bad Monitor Config:               None
Global Dimensions:                {host: primary}
GlobalSpanTags:                   map[]
Datapoints sent (last minute):    370
Datapoints failed (last minute):  0
Datapoints overwritten (total):   0
Events Sent (last minute):        6
Trace Spans Sent (last minute):   1083
Trace Spans overwritten (total):  0
```

Notice `Trace Spans Sent (last minute):   1083` 
This means spans are succssfully being sent to Splunk SignalFx.

#### K8S Step 7: Clean up deployments and services
`sh delete-all.sh`  
`helm delete signalfx-agent`  

This is the last lab of the [APM Instrumentation Workshop](../3-workshop-labs.md)