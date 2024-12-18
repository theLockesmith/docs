***This guide assumes that you are using a shared filesystem for your volumes. Consider converting all volumes not shared between pods to block storage.***

## Use btcpayserver-docker to generate the compose file

Generate your docker compose file:
`. ./btcpay-setup.sh --install-only`

## Use Kompose to generate manifests

Ensure environment variables used in the compose file are accessible and use kompose to generate manifests:

`set -a && source .env && set +a && kompose convert -f docker-compose.generated.yml`

***There are some instances where the same volume is mounted twice. You'll need to change the volume name of one or both of those instances in order for kompose to succeed.***

Here is RTL as an example:
```
    volumes:
    - "bitcoin_datadir:/etc/bitcoin"
    - "lnd_bitcoin_datadir:/etc/lnd"
    - "lnd_bitcoin_datadir:/root/.lnd"
    - "lndloop_bitcoin_datadir:/etc/lndloop"
    - "lnd_bitcoin_rtl_datadir:/data"
```

You would change the `lnd_bitcoin_datadir` volumes to something like `lnd_bitcoin_datadir_etc` and `lnd_bitcoin_datadir_root`:
```
    volumes:
    - "bitcoin_datadir:/etc/bitcoin"
    - "lnd_bitcoin_datadir_etc:/etc/lnd"
    - "lnd_bitcoin_datadir_root:/root/.lnd"
    - "lndloop_bitcoin_datadir:/etc/lndloop"
    - "lnd_bitcoin_rtl_datadir:/data"
```

If you have to do this, make sure to revert the change in the manifest of the corresponding deployment. If you do not, you are likely to encounter a pod that will remain stuck in a `ContainerCreating` state with no events or logs to guide you.

## Clean up manifests

Things like duplicate ports, replacing underscores `_` in metadata with dashes `-`, and duplicate PersistentVolumeClaim names may need to be addressed. Kompose may have missed some of these.

## Create a postgres service

You'll need to create a postgres service so that other pods can make their postgres connections. Here is a basic example:
```
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - name: "5432"
      port: 5432
      targetPort: 5432
```

You'll also need to add matching labels to your postgres deployment:
```
spec:
  ...
  template:
    metadata:
      labels:
        ...
        app: postgres
```

## SSH Keys

### You have 2 options:

1. Remove your SSH environment variables from the btcpayserver deployment. Since this is not a typical docker deployment, any communication with the host from the container is pointless.
2. Add your SSH key and authorized_keys as a secret. (Unless you plan to use the configurator, this is pointless.)

---
# You are technically done here

While this guide contains some more helpful information; at a bare minimum for conversion to kubernetes, you are done! The next section will go over access via tor and cloudflared. Access via ingress is outside the scope of this document.

## Cloudflared
If you want to user cloudflared, you have 2 options:

1. Ensure your tunnel is configured to serve the backend properly. Given the btcpayserver-docker defaults, you will need to configure (or reconfigure if you already have a tunnel) your tunnel to serve `http://btcpayserver:49392`.

2. Cloudflared also provides instructions to build your own tunnel via cli; [install cloudflared and create a tunnel](https://developers.cloudflare.com/cloudflare-one/tutorials/many-cfd-one-tunnel/). And cloudflare provides [this nice template](https://github.com/cloudflare/argo-tunnel-examples/blob/master/named-tunnel-k8s/cloudflared.yaml) for creating your deployment, configmap, and secret!

## Tor configuration
*If you do not use tor, you can skip this section. If you are using tor, you'll need the kube-gen sidecar and configmap again.*

### Tor relay
Add this ConfigMap in it's own section or file:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: tor-relay-template
  namespace: default # Change namespace as needed
data:
  torrc.tmpl: |
    ORPort 9001
    DirPort 9030
    ExitPolicy reject *:*
    CookieAuthentication 1

    Nickname {{ $.Env.TOR_RELAY_NICKNAME }}
    ContactInfo {{ $.Env.TOR_RELAY_EMAIL }}

    {{ if $.Env.ADDITIONAL_TORRC_CONFIG }}
    {{ $.Env.ADDITIONAL_TORRC_CONFIG }}
    {{ end }}
```

In your `tor-relay-deployment.yaml` Append the following to your `spec.template.spec.containers` section:
```
        - name: kube-gen
          image: kylemcc/kube-gen:master
          args:
            - "-watch"
            - "-wait"
            - "5s:30s"
            - "-in-cluster"
            - "/etc/docker-gen/templates/torrc.tmpl"
            - "/usr/local/etc/tor/torrc-2"
          env:
            - name: TOR_RELAY_EMAIL
              value: hello@example.com
            - name: TOR_RELAY_NICKNAME
              value: Hooli
            - name: TMPDIR
              value: "/usr/local/etc/tor"
          volumeMounts:
            - mountPath: /etc/docker-gen/templates
              name: kube-gen-relay-template
            - mountPath: /usr/local/etc/tor
              name: tor-relay-torrcdir

      volumes:
        - name: tor-relay-datadir
          persistentVolumeClaim:
            claimName: tor-relay-datadir
        - name: tor-relay-torrcdir
          emptyDir: {}
        - name: kube-gen-relay-template
          configMap:
            name: tor-relay-template
```

Then add this to your `spec.template.spec.containers.volumes` section:
```
        - name: tor-relay-template
          configMap:
            name: tor-relay-template
        - name: tor-relay-torrcdir
          emptyDir: {}
```

### Tor deployment
Add this ConfigMap to it's own section of `tor-deployment.yaml` or it's own file:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: tor-template
  namespace: default # Adjust namespace as needed
data:
  torrc.tmpl: |
    ORPort 9001
    DirPort 9030
    ExitPolicy reject *:*
    CookieAuthentication 1

    Nickname {{ $.Env.TOR_RELAY_NICKNAME }}
    ContactInfo {{ $.Env.TOR_RELAY_EMAIL }}

    {{ if $.Env.ADDITIONAL_TORRC_CONFIG }}
    {{ $.Env.ADDITIONAL_TORRC_CONFIG }}
    {{ end }}
```

And in your `tor-deployment.yaml` add the sidecar again to `spec.template.spec.containers`:
```
        - name: kube-gen
          image: kylemcc/kube-gen:master
          args:
            - "-watch"
            - "-wait"
            - "5s:30s"
            - "-in-cluster"
            - "/etc/docker-gen/templates/torrc.tmpl"
            - "/usr/local/etc/tor/torrc-2"
          env:
            - name: TOR_RELAY_EMAIL
              value: hello@example.com
            - name: TOR_RELAY_NICKNAME
              value: Hooli
            - name: TMPDIR
              value: "/usr/local/etc/tor"
          volumeMounts:
            - mountPath: /etc/docker-gen/templates
              name: kube-gen-template
            - mountPath: /usr/local/etc/tor
              name: tor-torrcdir
```

And the ConfigMap to `spec.template.spec.containers.volumes`:
```
        - name: tor-template
          configMap:
            name: tor-template
        - name: tor-relay-torrcdir
          emptyDir: {}
```

## Nginx
## ***THIS IS NOT A CONFIGURATION FOR AN INGRESS. AN INGRESS IS OUTSIDE THE SCOPE OF THIS DOCUMENT***
### *You do **NOT** need an nginx proxy as kubernetes services can be used for routing all your traffic*
Whether you use cloudflare tunnels or not, if you want to use nginx, you'll have to replace docker-gen, as you're not working in Docker anymore.

**I have done nothing more with this than ensure that kube-gen correctly builds the nginx config and nginx will run.**

Remove the `nginx-gen-deployment.yaml` manfiest. That will be replaced with `kube-gen` as a sidecar. kube-gen will need to be able to see your pods, services, and endpoints in order to generate the nginx config files for your services. That requires a ClusterRole and ClusterRoleBinding. Create a `rbac.yaml` file with these contents:
```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-gen-cluster-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-gen-cluster-rolebinding
subjects:
  - kind: ServiceAccount
    name: default
    namespace: btcpay
roleRef:
  kind: ClusterRole
  name: kube-gen-cluster-role
  apiGroup: rbac.authorization.k8s.io
```

*In this step, pay close attention to your resolver and search domain; `kube-dns` and `cluster.local` in this example.*

Next create a ConfigMap for the kube-gen template. I have converted the `nginx.tmpl` template from the btcpayserver-docker project to be compatible with kube-gen:
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-gen-template
  labels:
    app: nginx
  annotations:
    description: ConfigMap containing the nginx.tmpl file for kube-gen.
data:
  nginx.tmpl: |
    {{ $defaultHost := .Env.DEFAULT_HOST }}
    {{ if not $defaultHost }}
    {{ $defaultHost = "default.example.com" }}
    {{ end }}

    {{ define "upstream" }}
      {{ if .Address }}
        # {{ .Service.Name }} in {{ .Service.Namespace }}
        server {{ .Address }}:{{ .Port }};
      {{ else }}
        # Service has no address
        server 127.0.0.1 down;
      {{ end }}
    {{ end }}

    # Apply fix for very long server names
    server_names_hash_bucket_size 128;

    # Prevent Nginx Information Disclosure
    server_tokens off;

    # Default gzip settings
    gzip on;
    gzip_min_length 1000;
    gzip_types image/svg+xml text/plain text/css application/javascript application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    # Logging format
    log_format vhost '$host $remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log vhost;

    # Use resolvers specified in the environment
    {{ if .Env.RESOLVERS }}
    resolver {{ .Env.RESOLVERS }};
    {{ else }}
    resolver coredns.kube-system.svc.cluster.local valid=5s;
    {{ end }}

    # Default catch-all server
    server {
      listen 80 default_server;
      server_name _;
      access_log off;
      return 503;
    }

    # Main server configurations
    {{ range $service := .Services }}
      {{ if eq $service.ObjectMeta.Namespace .Env.POD_NAMESPACE }}
        {{ $serviceName := $service.ObjectMeta.Name }}
        {{ $namespace := $service.ObjectMeta.Namespace }}
        {{ $dnsName := printf "%s.%s.svc.cluster.local" $serviceName $namespace }}
        {{ range $port := $service.Spec.Ports }}
    server {
      listen {{ $port.Port }};
      server_name {{ $defaultHost }};
      
      location / {
        proxy_pass http://{{ $dnsName }}:{{ $port.Port }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }

      {{ if eq $serviceName "lnd-bitcoin" }}
      location ~* ^/(lnrpc|routerrpc|verrpc|walletrpc)\. {
        grpc_read_timeout 6000s;
        grpc_send_timeout 6000s;
        grpc_pass grpcs://{{ $dnsName }}:{{ $port.Port }};
      }
      location /lnd-rest/btc/ {
        rewrite ^/lnd-rest/btc/(.*) /$1 break;
        proxy_pass http://{{ $dnsName }}:8080/;
      }
      {{ end }}

      {{ if eq $serviceName "bitcoin-rtl" }}
      location /rtl/ {
        proxy_pass http://{{ $dnsName }}:3000/rtl/;
      }
      {{ end }}

      {{ if eq $serviceName "joinmarket" }}
      location /obwatch/ {
        proxy_pass http://{{ $dnsName }}:62601/;
      }
      {{ end }}

      {{ if eq $serviceName "bitcoin-thub" }}
      location /thub {
        proxy_pass http://{{ $dnsName }}:3000/thub;
      }
      {{ end }}

      {{ if eq $serviceName "btctransmuter" }}
      location /btctransmuter/ {
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_pass http://{{ $dnsName }};
      }
      {{ end }}

      {{ if eq $serviceName "helipad" }}
      location /helipad/ {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://{{ $dnsName }}:2112/;
      }
      {{ end }}

      {{ if eq $serviceName "torq" }}
      location /torq/ {
        proxy_pass http://{{ $dnsName }}:8080/;
      }
      {{ end }}
    }
        {{ end }}
      {{ end }}
    {{ end }}
```

Lastly, add kube-gen as a sidecar container to nginx and add a script to the nginx container to wait to start until after kube-gen has generated the config file:
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
    io.kompose.service: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      io.kompose.service: nginx
  template:
    metadata:
      labels:
        app: nginx
        io.kompose.service: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.3-bookworm
          command: ["/bin/sh", "-c"]
          args:
            - |
              echo "Waiting for kube-gen to generate configuration...";
              while [ ! -f /etc/nginx/conf.d/default.conf ]; do
                sleep 2;
              done;
              echo "Configuration found, starting NGINX.";
              nginx -g "daemon off;";
          ports:
            - containerPort: 80
              protocol: TCP
          volumeMounts:
            - mountPath: /etc/nginx/conf.d
              name: nginx-conf

        - name: kube-gen
          image: kylemcc/kube-gen:master
          args:
            - "-watch"
            - "-wait"
            - "5s:30s"
            - "-type"
            - "services"
            - "-in-cluster"
            - "/etc/docker-gen/templates/nginx.tmpl"
            - "/etc/nginx/conf.d/default.conf"
          env:
            - name: DEFAULT_HOST
              value: btcpay.example.com
            - name: TMPDIR
              value: "/etc/nginx/conf.d"
          volumeMounts:
            - mountPath: /etc/docker-gen/templates
              name: kube-gen-template
            - mountPath: /etc/nginx/conf.d
              name: nginx-conf

      volumes:
        - name: nginx-conf
          emptyDir: {}
        - name: kube-gen-template
          configMap:
            name: nginx-gen-template
```

## HA LND

*More to come in this section*

See https://docs.lightning.engineering/lightning-network-tools/lnd/leader_election#identifying-the-leader-node if you want to setup leader election for high availability lnd.