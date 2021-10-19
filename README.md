### ksniff — all the goodness of Wireshark, running in Kubernetes.

1) **Install and setup krew on your machine**
    
    `https://krew.sigs.k8s.io/docs/user-guide/setup/install/`
    
2) **Install ksniff**
     
    Install ksniff using the kubectl plugin package manager, krew:
    
    `$ kubectl krew install sniff`
    
3) **You will also need to install Wireshark on your local machine.**

    `https://www.wireshark.org/download.html`
    
4) **Accessing application in the cluster from outside**
    
    I've used an example application `https://github.com/kubernetes/examples/tree/master/guestbook-go` with frontend exposed on port 3000 via a LoadBalancer service.
    I can make an external request to my guestbook application that is exposed via port 3000
    
    ```
    
    $ curl -v a24******************************.eu-central-1.elb.amazonaws.com:3000
    *   Trying 35.***.**.20:3000...
    * TCP_NODELAY set
    * Connected to a24******************************.eu-central-1.elb.amazonaws.com (35.***.**.20) port 3000 (#0)
    > GET / HTTP/1.1
    > Host: a24******************************.eu-central-1.elb.amazonaws.com:3000
    > User-Agent: curl/7.68.0
    > Accept: */*
    > 
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 200 OK
    < accept-ranges: bytes
    < content-length: 922
    < content-type: text/html; charset=utf-8
    < last-modified: Wed, 16 Dec 2015 18:26:32 GMT
    < date: Fri, 15 Oct 2021 10:22:18 GMT
    < x-envoy-upstream-service-time: 0
    < server: istio-envoy
    < x-envoy-decorator-operation: guestbook.default.svc.cluster.local:3000/*
    < 
    <!DOCTYPE html>
    <html lang="en">
     <head>
      <meta content="text/html; charset=utf-8" http-equiv="Content-Type">
      <meta charset="utf-8">
      <meta content="width=device-width" name="viewport">
      <link href="style.css" rel="stylesheet">
      <title>Guestbook</title>
      </head>
      <body>
       <div id="header">
         <h1>Guestbook</h1>
       </div>

       <div id="guestbook-entries">
        <p>Waiting for database connection...</p>
       </div>

       <div>
        <form id="guestbook-form">
          <input autocomplete="off" id="guestbook-entry-content" type="text">
          <a href="#" id="guestbook-submit">Submit</a>
        </form>
       </div>

       <div>
        <p><h2 id="guestbook-host-address"></h2></p>
        <p><a href="env">/env</a>
        <a href="info">/info</a></p>
      </div>
      <script src="//ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min.js"></script>
       <script src="script.js"></script>
     </body>
    </html>
    * Connection #0 to host a24******************************.eu-central-1.elb.amazonaws.com left intact
    ```
    
    Let’s make another request, but view the internal inter-cluster network traffic via ksniff. First I need to get the name of the `guestbook service’s Pod`, as this is where I’ll be attaching ksniff
    
    ```
    $ kubectl get pods
    NAME                 READY   STATUS    RESTARTS   AGE
    guestbook-sxt2z      2/2     Running   0          4h42m
    redis-master-fmqzg   2/2     Running   0          4h42m
    redis-slave-7qtrd    2/2     Running   0          4h42m
    redis-slave-vqk9f    2/2     Running   0          4h42m

    ```
    
    I can attach ksniff to this Pod with a simple command via my local machine:
    
    ```
    $ kubectl sniff guestbook-sxt2z -n default
    INFO[0000] using tcpdump path at: '/home/rumen/.krew/store/sniff/v1.6.1/static-tcpdump'    
    INFO[0000] no container specified, taking first container we found in pod. 
    INFO[0000] selected container: 'guestbook'              
    INFO[0000] sniffing method: upload static tcpdump       
    INFO[0000] sniffing on pod: 'guestbook-sxt2z' [namespace: 'default', container: 'guestbook', filter: '', interface: 'any'] 
    INFO[0000] uploading static tcpdump binary from: '/home/rumen/.krew/store/sniff/v1.6.1/static-tcpdump' to: '/tmp/static-tcpdump' 
    INFO[0000] uploading file: '/home/rumen/.krew/store/sniff/v1.6.1/static-tcpdump' to '/tmp/static-tcpdump' on container: 'guestbook' 
    INFO[0000] executing command: '[/bin/sh -c test -f /tmp/static-tcpdump]' on container: 'guestbook', pod: 'guestbook-sxt2z', namespace: 'default' 
    INFO[0001] command: '[/bin/sh -c test -f /tmp/static-tcpdump]' executing successfully exitCode: '1', stdErr :'' 
    INFO[0001] file not found on: '/tmp/static-tcpdump', starting to upload 
    INFO[0004] verifying file uploaded successfully         
    INFO[0004] executing command: '[/bin/sh -c test -f /tmp/static-tcpdump]' on container: 'guestbook', pod: 'guestbook-sxt2z', namespace: 'default' 
    INFO[0004] command: '[/bin/sh -c test -f /tmp/static-tcpdump]' executing successfully exitCode: '0', stdErr :'' 
    INFO[0004] file found: ''                               
    INFO[0004] file uploaded successfully                   
    INFO[0004] tcpdump uploaded successfully                
    INFO[0004] spawning wireshark!                          
    INFO[0004] start sniffing on remote container           
    INFO[0004] executing command: '[/tmp/static-tcpdump -i any -U -w - ]' on container: 'guestbook', pod: 'guestbook-sxt2z', namespace: 'default' 

    ```
    
    In the above CLI output, you should see all of the tcpdump configuration, and if all goes well, Wireshark should launch
    I can see all traffic going in and out of the pod.
    
    
    Inspecting the HTTP traffic after I make a request in the guestbook go I can see that the source IP of my requests is the internal node IP I currently have (10.250.11.207) and the destination is 100.96.0.19 (The container's eth0 IP address). We can see that the source port is 3000 (request coming from the browoser, that's the open port to external http requests)
    
    ![screen1](https://github.com/rdimitrov4/Inspecting-Service-Mesh-TLS-in-Kubernetes/blob/main/screen1.png?raw=true)
    `Node IP`
    
    ![screen2](https://github.com/rdimitrov4/Inspecting-Service-Mesh-TLS-in-Kubernetes/blob/main/screen2.png?raw=true)
    `Container IP`
    
    ![screen3](https://github.com/rdimitrov4/Inspecting-Service-Mesh-TLS-in-Kubernetes/blob/main/screen3.png?raw=true)
    `Wireshark on the Pod`
    
    
5) **Installing Istio Service Mesh on the Kubernetes cluster to enable TLS encryption**

    After meeting the prerequisites for Istio and making the necessary platform setup we can Install Istio on our K8S cluster following the official documentation
    `https://istio.io/latest/docs/setup/`
    
   After doing so and labeling our default namespace with the `istio-injection=enabled` lable we are ready to go.
   
   
