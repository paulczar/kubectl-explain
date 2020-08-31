# $ kubectl explain configmap

[$ kubectl explain](../)

<iframe width="420" height="315"
src="https://www.youtube.com/embed/Fy6nQOeGb5w">
</iframe>

A ConfigMap is an API object used to store data in key-value pairs. ConfigMaps are designed for non-confidential text based data. For confidential data you would use a [Secret](../secret) which we will talk about later. While ConfigMaps are text based, its not uncommon to encode small amounts of binary data using `base64` or a similar text based encoding method to be used in a ConfigMap. I won't tell you not to do that, but I will sit here and quietly judge you for it.

Pods can consume ConfigMaps as environment variables or as files in a [Volume](../volume). This allows you to decouple your application's container image from your configuration to improve the portability and cloud native-ness (is that a word? it is now!) of your application.

> **Warning** ConfigMaps do not provide any secrecy and should not be used for any confidential data. Use [Secrets](../secret) or even better an external tool like [Hashicorp Vault](https://www.vaultproject.io/) for passwords, keys, connection strings, etc.

## Why ConfigMaps ?

I just mentioned the primary why of ConfigMaps but lets dig a little deeper into it. Part of the [twelve-factor app](https://12factor.net/) is the [separation](https://12factor.net/config) of config and code. While the exact prescriptions of the twelve-factor app don't directly translate to Cloud Native Readiness and running software on Kubernetes, the concepts are sound.

Separating your config and code helps to ensure that your application is as portable as possible. Common practice across Cloud Native development is to allow you to configure an app via a combination of configuration files, command line arguments, and environment variables. This then gives ultimate flexibility to the platform itself on how it provides the configuration required to run an aplication.

With ConfigMaps you can actually provide config in any of those three formats quite easily. Given that even older applications are configured by Config files they are not only great for cutting edge cloud native applications, but also for the lift and shift of existing legacy applications.

## Show me!

A ConfigMap is in my opinion the simplest Kubernetes resource to write. Its certainly about the only resource that I can confidently create from scratch without referring to documentation or existing examples. Apart from defining the resource itself, the config map has just one primary field called `data` this field contains one or more key value pairs.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # literal key value pairs
  port: 8080
  ip: 0.0.0.0
  #
  # file based key value pairs
  config.properties: |
    [server]
    webroot=/opt/app/html
    player.maximum-lives=5
```

Ordinarily I wouldn't mix literal and file based key/value pairs in the same ConfigMap, but for the sake of demonstration here we are.

You could also create a ConfigMap from the command line like so:

```bash
kubectl create configmap app-config \
  --from-literal="port=8080" \
  --from-literal="ip=0.0.0.0" \
  --from-file="config.properties"

```

You can then access ConfigMap from a pod using the key value pairs as environment variables or files in a volume.

### Files in a volume

The easiest way to do this is to load the whole ConfigMap up as a volume, this will create a read only volume at the specified path and create every key/value pair as a file (with the key being the name of the file).

The advantage to using this method is that if you update the ConfigMap the contents of the volume will also be updated. If your application supports reloading configuration dynamically this may allow you to reconfigure your application on the fly.

Here's a Pod definition that will use the ConfigMap as files in a volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: docker.io/user/my-app
    volumeMounts:
    - name: config
      mountPath: "/opt/app/config"
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
```

Using the ConfigMap from earlier this would create three files inside the volume `/opt/app/config` named `ip`, `port`, and `config.properties`. This is why I wouldn't usually mix literal and file based entries in the same ConfigMap. However you can actually get specific about which files to load like so:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: docker.io/user/my-app
    volumeMounts:
    - name: config
      mountPath: "/opt/app/config"
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
      items:
      - key: application.properties
        path: my.properties
```

This version of the pod manifest would only load a single file `/opt/app/config/my.properties` which could contain the contents of the key `application.properties`.

## Environment Variables

Just like you can load an entire ConfigMap into a volume of files, you can also load a whole configmap up as environment variables like so, and you can even reference those variables in your applications entrypoint:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: docker.io/user/my-app
    envFrom:
    - configMapRef:
        name: app-config
    command: ["/opt/app/bin/start"]
    args: ["--port", "${port}, --ip ${ip}"]
```

However like before along with the `$port` and `$ip` environment variables we'd also get a variable `$config.properties` with a config file loaded into it, likely not what was intended. To get around this you can load up specific keys as environment variables.

Let's say the application expects to see the variables `MYAPP_PORT` and `MYAPP_IP` we could configure our Pod like so:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: docker.io/user/my-app
    env:
    - name: MYAPP_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: port
    - name: MYAPP_IP
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: ip
```

Now we don't even need to provide any command line arguments via the Pod configuration as the application happily reads the expected environment variables, we also no longer have a `application.properties` environment variable cluttering things up.

## Secrets

Secrets are almost exactly like ConfigMaps, except that they are base64 encoded by default. This is not for the sake of encryption but to make it safe to store binary secrets like keys.

Kubernetes does restrict access to Secrets more tightly than it does ConfigMaps. If you configure `etcd` to be encrypted at rest then Secrets may be secure enough for some forms of secrets, however it is recommended that you use even more secure secret management via tools like Vault.

## Conclusion

ConfigMaps join Pods and Services to form the Holy Trinity of Kubernetes resources consisting of "compute" "networking" and "configuration". Of course there are many more Kubernetes resources that extend these concepts or work alongside these concepts (like Volumes) to provide an even more feature-rich platform.


## Sources & Further Reading

* [Kubernetes Docs - ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
* [Kubernetes Docs - Configure a Pod to use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

[$ kubectl explain](../)