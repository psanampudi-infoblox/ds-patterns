## Service Mesh
- platform level infra. Provides security, reliability, observability etc
- works at the os network stack level
- by very nature adds latency to your applications
- linkerd2 is kube native
    - expect to see kubernetes objects
    - control plane
    - data plane (mainly proxy) 
- control plane
    - central service driving data planes across the cluster
    - supports extensions (additional services)
- uses TLS for security    

### functions
- observability
- security
- reliability


## what you get
- call graph
- golden metrics
    - latency (p95 for ex. time taken by 95% of requests)
    - request rate
    - success rate/error rate


## more research needed
- is our deployment uses multi available zones ?
 - if so, is using linkerd os version going to bump up the cloud costs ?
- what is 'topology hints' in kubernetes
    - check 'troublw with topology aware routing'
- Read 'zero trust net sec in kubernetes'
- Read 'what is mtls'
- trust anchor ?

- bounce your pods once in a while. dont let them run for years.
- uddi and platform have added tracing coverage. TD , Data and devops apps do not have much tracing coverage.
- 