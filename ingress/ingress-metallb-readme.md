# title
deploy kubespray without ingress


Get ingress-nginx helm

```
$ helm repo add nginx-stable https://helm.nginx.com/stable
"nginx-stable" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "nginx-stable" chart repository
```

Create namespace for nginx-ingress
```
$ kubectl create ns nginx-ingress
```

Install nginx-ingress helm chart

```
$ helm install my-release nginx-stable/nginx-ingress --namespace nginx-ingress
NAME: my-release
LAST DEPLOYED: Thu Sep  1 01:14:54 2022
NAMESPACE: nginx-ingress
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The NGINX Ingress Controller has been installed.
```

Verify all is running
```
$ k -n nginx-ingress get all
NAME                                            READY   STATUS    RESTARTS   AGE
pod/my-release-nginx-ingress-55c5b7fc96-b59lq   1/1     Running   0          27s

NAME                               TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE
service/my-release-nginx-ingress   LoadBalancer   10.233.29.4   192.168.88.30   80:31022/TCP,443:30062/TCP   28s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-release-nginx-ingress   1/1     1            1           28s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/my-release-nginx-ingress-55c5b7fc96   1         1         1       28s
```

Now we are ready to create an ingress.

## Create a sample service to test the ingress

Lets create a deployment and service called bes-deploy in the namespace with the same name with the nginx image

```
# Create the namespace
$ kubectl create namespace bes-deploy

# Create the deployment
$ kubectl create deployment -n bes-deploy bes-deploy --image nginx

# Create the service
$ kubectl -n bes-deploy expose deployment bes-deploy --port 80

# verify all is created with
$ kubectl -n bes-deploy get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/bes-deploy-569fc9bd4-cgw4j   1/1     Running   0          2d8h

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/bes-deploy   ClusterIP   10.233.50.138   <none>        80/TCP    2d8h

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/bes-deploy   1/1     1            1           2d8h

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/bes-deploy-569fc9bd4   1         1         1       2d8h

```

Now lets create our ingress rule. We are going to test to reach the sample service at  `bes-deploy.cloudalbania.com/` in both http and https.

Let's create the sample cert for our tls endpoint:
```
$ KEY_FILE=./key.pem
$ CERT_FILE=./certificate.pem
$ openssl req -newkey rsa:2048 -nodes -keyout ${KEY_FILE} -x509 -days 365 -out ${CERT_FILE}
```

Add this cert as a secret in k8s:
```
$ kubectl create secret tls bes-secret ${CERT_NAME} --key ${KEY_FILE} --cert ${CERT_FILE}
```

Finally create our ingress in the cluster with this manifest. We have one mor final step after this:

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bes-ingress
  namespace: bes-deploy
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - bes-deploy.cloudalbania.com
    secretName: bes-secret
  rules:
    - host: bes-deploy.cloudalbania.com
      http:
        paths:
          - pathType: Prefix
            backend:
              service:
                name: bes-deploy
                port:
                  number: 80
            path: /
EOF
```

To access `bes-deploy.cloudalbania.com` we need to temporary update our /etc/hosts (instead of creating a fully new DNS A record)

Let's get the ingress IP:
```
$ k describe ingress -n bes-deploy bes-ingress | grep Address
Address:          192.168.88.30
```

add this line in our `/etc/hosts`:

```
...
192.168.88.30   bes-deploy.cloudalbania.com
...
```

Check if the pods are running and finally test the access from your local machine:

```
# HTTPS
$ curl -I https://bes-deploy.cloudalbania.com/ -k
HTTP/1.1 200 OK
Server: nginx/1.23.0
Date: Thu, 01 Sep 2022 05:56:06 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
Last-Modified: Tue, 19 Jul 2022 14:05:27 GMT
ETag: "62d6ba27-267"
Accept-Ranges: bytes

# HTTP
$ curl -I http://bes-deploy.cloudalbania.com/
HTTP/1.1 301 Moved Permanently
Server: nginx/1.23.0
Date: Thu, 01 Sep 2022 05:56:37 GMT
Content-Type: text/html
Content-Length: 169
Connection: keep-alive
Location: https://bes-deploy.cloudalbania.com:443/

```

Test with content:
```
$ curl  http://bes-deploy.cloudalbania.com/ -Lk
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
