- [POC\_cloudflare\_tunnel\_to\_expose\_k8s](#poc_cloudflare_tunnel_to_expose_k8s)
  - [Concept](#concept)
  - [General instruction](#general-instruction)
  - [Detailed instructions](#detailed-instructions)
    - [Create CloudFlare resources \& K8S manifests (terraform)](#create-cloudflare-resources--k8s-manifests-terraform)
    - [Install ingress controller (helm)](#install-ingress-controller-helm)
    - [Deploy K8S config and secret (kubectl)](#deploy-k8s-config-and-secret-kubectl)
    - [Deploy cloudflared (kubectl)](#deploy-cloudflared-kubectl)
    - [Deploy App \& Ingress (kubectl)](#deploy-app--ingress-kubectl)
- [Check Cloudflare UI for status \& logs](#check-cloudflare-ui-for-status--logs)

# POC_cloudflare_tunnel_to_expose_k8s

POC how to expose any application running in Kubernetes without an external load balancer.  

Can be used for any Kubernetes cluster. Works great with Talos Linux.  

## Concept

[What is Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)

![ Alt Text](https://developers.cloudflare.com/_astro/handshake.eh3a-Ml1_1IcAgC.webp)

On Kubernetes:  

![ Alt Text](https://miro.medium.com/v2/resize:fit:720/format:webp/1*UrXZBLdPwsKkvbwjDlfevA.png).

ExternalDNS is optional, it is 'cheaper' to add the DNS record using terraform.  

## General instruction

Requirements:  
- Kubernetes Cluster / Talos Linux Cluster  

1] Create CloudFlare resources & K8S manifests (terraform)  
2] Install ingress controller (helm)  
3] Deploy K8S config and secret (kubectl)  
4] Deploy cloudflared (kubectl)  
5] Deploy App & Ingress (kubectl)  
6] Check Cloudflare UI for status & logs  

## Detailed instructions

### Create CloudFlare resources & K8S manifests (terraform)  

Pay attention to the static configuration:  
- `your_zone_id`
- `tunnel.poc`
- `https://ingress-nginx-controller.ingress-nginx.svc.cluster.local`

`main.tf`

    terraform {
      # OpenTofu version
      required_version = ">= 1.8.3"

     required_providers {
        cloudflare = {
          source  = "cloudflare/cloudflare"
          version = "5.1.0"
      }
    }

`variables.tf`

    variable "cloudflare_api_token" {
      type      = string
      nullable  = false
      sensitive = true
    }

    variable cloudflare_account_id {
      type      = string
      nullable  = false
    }

    # Must be at least 32 bytes
    variable "cloudflare_tunnel_secret" {
      type      = string
      nullable  = false
      sensitive = true
    }

    variable "project" {
      default = "POC_cloudflare_tunnel_to_expose_k8s"
    }

    variable "talos_control_plane_name" {
      default = "POC_cloudflare_tunnel_to_expose_k8s"
    }
    
    variable "dir_path_to_talos_files" {
      default = "../POC_cloudflare_tunnel_to_expose_k8s"
    }

`cloudflare.yaml`  

    data "cloudflare_zone" "tunnel_poc" {
      # name = "tunnel.poc"
      zone_id = "your_zone_id"
    }

    resource "cloudflare_zero_trust_tunnel_cloudflared" "tunnel_to_ingress" {
      account_id = var.cloudflare_account_id
      name = var.project
      config_src = "cloudflare"
      tunnel_secret = base64sha256(var.cloudflare_tunnel_secret)
    }

    resource "cloudflare_dns_record" "tunnel_to_ingress" {
      zone_id = data.cloudflare_zone.tunnel_poc.zone_id
      name    = "tunnel.poc"
      content = "${cloudflare_zero_trust_tunnel_cloudflared.tunnel_to_ingress.id}.cfargotunnel.com"
      type    = "CNAME"
      proxied = true
      ttl     = 1
      comment = "tunnel_to_ingress"
    }

    resource "cloudflare_zero_trust_tunnel_cloudflared_config" "tunnel_to_ingress" {
      tunnel_id = cloudflare_zero_trust_tunnel_cloudflared.tunnel_to_ingress.id
      account_id = var.cloudflare_account_id

      config = {
        ingress = [
          {
            hostname       = "tunnel.poc"
            service        = "https://ingress-nginx-controller.ingress-nginx.svc.cluster.local"
            origin_request = {
              no_tls_verify = true
            }
          },
          {
            service = "http_status:404"
          }
        ]
      }
    }

    # not used, because data not return token, fixed by Cloudflare, waiting for terraform provider release
    # https://github.com/cloudflare/terraform-provider-cloudflare/issues/5149
    data "cloudflare_zero_trust_tunnel_cloudflared_token" "tunnel_to_ingress" {
      account_id = var.cloudflare_account_id
      tunnel_id = cloudflare_zero_trust_tunnel_cloudflared.tunnel_to_ingress.id
    }

    locals {
      cloudflare_tunnel_secret = jsonencode({
        "AccountTag" : "${var.cloudflare_account_id}",
        "TunnelSecret" : "${base64sha256(var.cloudflare_tunnel_secret)}",
        "TunnelID" : "${cloudflare_zero_trust_tunnel_cloudflared.tunnel_to_ingress.id}"
      })
    }

    resource "local_file" "secret" {
      filename = "${var.dir_path_to_talos_files}/cloudflare_tunnel_token_${var.talos_control_plane_name}.yaml"
      content  = <<-EOT
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudflare-tunnel-credentials
      namespace: ingress-nginx
    type: Opaque
    data:
      credentials.json: ${base64encode(local.cloudflare_tunnel_secret)}
    EOT
    }

    resource "local_file" "cloudflared_config" {
      filename = "${var.dir_path_to_talos_files}/cloudflare_tunnel_config_${var.talos_control_plane_name}.yaml"
      content  = <<-EOT
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: cloudflared-tunnel-config
      namespace: ingress-nginx
    data:
      config.yml: |
        tunnel: "${cloudflare_zero_trust_tunnel_cloudflared.tunnel_to_ingress.id}"
        credentials-file: /etc/cloudflared/creds/credentials.json
    EOT
    }

### Install ingress controller (helm)  

    helm upgrade --install ingress-nginx ingress-nginx \
      --repo https://kubernetes.github.io/ingress-nginx \
      --namespace ingress-nginx --create-namespace \
      --set controller.service.type=ClusterIP \
      --set controller.kind=DaemonSet

### Deploy K8S config and secret (kubectl)  

Configuration needed for a cloudflared pod.  
It will be possible to improve it after the release: https://github.com/cloudflare/terraform-provider-cloudflare/issues/5149  

    kubectl apply -f cloudflare_tunnel_token_${var.talos_control_plane_name}.yaml
    kubectl apply -f cloudflare_tunnel_config_${var.talos_control_plane_name}.yaml

### Deploy cloudflared (kubectl)

Use more replicas OR daemonset in production environment.  

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: cloudflared
      name: cloudflared
      namespace: ingress-nginx
    spec:
      replicas: 1
      selector:
        matchLabels:
          pod: cloudflared
      template:
        metadata:
          labels:
            pod: cloudflared
        spec:
          securityContext:
            sysctls:
              - name: net.ipv4.ping_group_range
                value: "65532 65532"
          containers:
            - image: cloudflare/cloudflared:2025.2.0
              name: cloudflared
              args:
                - tunnel
                - --config
                - /etc/cloudflared/config.yml
                - --protocol # https://github.com/cloudflare/cloudflared/issues/1176#issuecomment-2404546711
                - http2  
                - --no-autoupdate
                - --metrics
                - 0.0.0.0:2000
                - run
              livenessProbe:
                httpGet:
                  path: /ready
                  port: 2000
                failureThreshold: 1
                initialDelaySeconds: 10
                periodSeconds: 10
              resources:
                requests:
                  cpu: "125m"
                  memory: "128Mi"
                limits:
                  cpu: "250m"
                  memory: "256Mi"
              volumeMounts:
                - name: creds
                  mountPath: /etc/cloudflared/creds
                  readOnly: true
                - name: config
                  mountPath: /etc/cloudflared/config.yml
                  subPath: config.yml
          volumes:
          - name: creds
            secret:
              secretName: cloudflare-tunnel-credentials
          - name: config
            configMap:
              name: cloudflared-tunnel-config

### Deploy App & Ingress (kubectl)  

Pay attention to the static configuration:  
- `tunnel.poc`
- `origin-cloudflare-cert`

Deployed in `namespace: ingress-nginx` for faster debug.

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: hello-world-ingress
      namespace: ingress-nginx
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      ingressClassName: nginx
      tls:
        - hosts:
            - tunnel.poc
          secretName: origin-cloudflare-cert
      rules:
        - host: tunnel.poc
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: hello-world-service
                    port:
                      number: 80

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: hello-world-service
      namespace: ingress-nginx
    spec:
      selector:
        app: hello-world
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80

    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-world-deployment
      labels:
        app: hello-world
      namespace: ingress-nginx
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: hello-world
      template:
        metadata:
          labels:
            app: hello-world
        spec:
          containers:
            - name: hello-world
              image: hashicorp/http-echo:latest
              args:
                - "-text=Hello, World!"
                - "-listen=:80"
              ports:
                - containerPort: 80
              resources:
                requests:
                  cpu: "125m"
                  memory: "128Mi"
                limits:
                  cpu: "250m"
                  memory: "256Mi"

# Check Cloudflare UI for status & logs  

You can check the logs directly in K8S and in Cloudflare. `Cloudflare -> Zero Trust -> Networks -> Tunnels`  
