+++
title = 'Kubernetes Service External Traffic Policy and AWS ALB'
date = 2024-04-11T16:09:02+02:00
draft = false
toc = true

+++

## Lead-up

Let's say you have an oddly specific use case of running [the Nginx Ingress Controller](https://github.com/nginxinc/kubernetes-ingress) behind the AWS Application Load Balancer. Maybe you don't want to lose the comfort and practicality of AWS Certificate Manager, yet you long for some features only Nginx can provide (e.g. setting custom headers). It sounds odd to have yet another hop of reverse proxy, but you decided it's worth it.

To not bother with configuring the AWS ALB manually you've configured [AWS ALB Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.7/). You've used the official Nginx chart to deploy the Nginx along with necessary resources. You've decided 3 pods will be just right. You read the documentation of ALB Controller, Service Type must be set to NodePort for configuration between ALB and EKS nodes. You've configured the Ingress resource just like that.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress-controller
  namespace: nginx
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /nginx-health
    alb.ingress.kubernetes.io/ip-address-type: ipv4
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/load-balancer-name: nginx
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-nginx-ingress-controller
                port:
                  number: 80
```

It exposes the service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nginx-ingress-controller
  namespace: nginx
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30851
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
      nodePort: 32350
  selector:
    app.kubernetes.io/instance: nginx
    app.kubernetes.io/name: nginx-ingress
  type: NodePort
  externalTrafficPolicy: Local
```

ALB Controller picks up the annotations from the Service resource, Load Balancer is reconciled, up & running. But what's that? Some nodes are not passing a healthcheck. A quick check reveals these are the ones that are not running any of Nginx pods with error saying that the `Request timed out`. Why is that? 


## Debugging 

You debug the thing. You SSH into one of the nodes which are not passing the healthcheck. You do the thing:
```shell
curl -I localhost:30851
```

And you get the response. You double-check - Nginx is not running on this node. The traffic is local, you're using a loopback interface. Then why ALB can't reach it?

## Root cause

Well, the problem sits in the chair. You've copy-pasted some bollocks from the internet without understanding what does it mean.

You see, when you put the `.spec.externalTrafficPolicy = 'Local'` in your Service definition, you ordered kubeProxy to handle all the traffic originating outside of the cluster by the node that received that traffic, omitting completely the Service abstraction.

What does it mean in practice is if your node doesn't run the pod which is registered as an endpoint in the Service it will drop (not reject) the connection, which will result in timeout.

But why it doesn't happen when testing it from a node? Because it's not considered an external traffic, hence if you'd like to mimic the same behaviour within a cluster you'd need to specify `.spec.internalTrafficPolicy = 'Local'`. Both fields default to `'Cluster'`. 

## Tl;dr

If some of your nodes are not passing healtchecks for NodePort services remove `.spec.externalTrafficPolicy = 'Local'`.

## References

- [Kubernetes docs on Traffic Policies](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)
- [Kubernetes docs on handling external traffic](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip)
- [Kubernetes docs on Service NodePort type](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
- [Piece of code responsible for described behaviour](https://github.com/kubernetes/kubernetes/blob/be4b7176dc131ea842cab6882cd4a06dbfeed12a/pkg/proxy/nftables/proxier.go#L1178)
