# Tomcat in kubernetes : Bullet proff sessions

## introduction

This is a sample project that will demostrate the default jgroup session persistence from tomcat with a kubernetes

## For the impatient (minikube)

### Launch the POC

First you need to set you /etc/hosts to resolve "khool-session.info khoolsession.khool-session.info khool-session-manual.khool-session.info".

```bash
minikube start -p khool-session       #take around 5 min on my laptop (I use Virtualbox engine).
echo "$(minikube ip -p khool-session) khool-session.info khoolsession.khool-session.info khool-session-manual.khool-session.info"
```

and copy the result in `/etc/hosts`

**Notice** : you could do a oneliner with `sudo bash -c "echo \"$(minikube ip -p khool-session) khool-session.info khoolsession.khool-session.info khool-session-manual.khool-session.info\" >> /etc/hosts"` but it feel too error prone for me, specially if you do only one ">" so `sudo vim` is ok for me ;-)

```bash
kubectl get ns                               #to check kubernetes si set correctly
eval $(minikube docker-env -p khool-session) # to use minikube docker registry
mvn package k8s:build k8s:deploy    # to builds and deploy 3 instance (it take around 5 minutes also)
```

You should be able to acces the demo page <http://khoolsession.khool-session.info/khoolsession/BasicServlet>
that dispaly :

- Session ID (unique for your browswer)
- HostName (is this case the pod name)

#### Test 

If you call the endpoint several time without keeping cookie, you'll have a new Session ID each time. (we use grep to keep only informations we need)

```bash
version=khoolsession; for i in {1..5} ; do curl -s http://${version}.khool-session.info/khoolsession/BasicServlet | grep "Session ID:"; done

# Session ID: 9238BFD68F9C5F6FE2222FD12F2ED256<br />
# Session ID: 90928813B6DDFB05CF652D3BAD9265A1<br />
# Session ID: 2DC154CEEA05D09048A71B86255A308F<br />
# Session ID: 6BBD6C1F029E2D0FDB3281A197BE94A4<br />
# Session ID: 873511BB13598B340CD5237E7C08DD47<br />

```

If we keep cookie (`-b /tmp/cookies -c /tmp/cookies`) we can see that we keep the same session id.

```bash
version=khoolsession; for i in {1..5} ; do curl -b /tmp/cookies -c /tmp/cookies -s http://${version}.khool-session.info/khoolsession/BasicServlet | grep "Session ID:"; done

# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />

```

And we widen the grep scope with a `-A 1` to add on line after our grep match, in order to check pod's name.

```bash
version=khoolsession; for i in {1..5} ; do curl -b /tmp/cookies -c /tmp/cookies -s http://${version}.khool-session.info/khoolsession/BasicServlet | grep -A1 "Session ID:"; done
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-wpb8m</b><br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-wpb8m</b><br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-wpb8m</b><br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-wpb8m</b><br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-wpb8m</b><br />
```

We can see that we are always on the same pod. Sticky session works !

Now let's kill this pod.

```bash
version=khoolsession; kubectl -n khoolsession delete po  $(curl -b /tmp/cookies -c /tmp/cookies -s http://${version}.khool-session.info/khoolsession/BasicServlet | grep "HostName" | awk -F"<|>" '{print $3}')
```

And check that we still have the same session

```bash
version=khoolsession; for i in {1..5} ; do curl -b /tmp/cookies -c /tmp/cookies -s http://${version}.khool-session.info/khoolsession/BasicServlet | grep -A1 "Session ID:"; done
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-bdsq2</b><br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-bdsq2</b><br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-bdsq2</b><br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-bdsq2</b><br />
# Session ID: 3625FB20BCA4181613683F3737B6F4E7<br />
# HostName: <b>khoolsession-78b49bd75-bdsq2</b><br />
```

now lets looks to the details version

## For the detail

### the webapp

#### the servlet

The servlet retrieve the client session dans check for info within the session

[BasicServlet](src/main/java/io/github/kandefromparis/khool/tomcat/kube/BasicServlet.java)

<!-- look for insert tag -->
```java
HttpSession session = request.getSession(true);
String hostName = InetAddress.getLocalHost().getHostName();

StringBuilder sb = new StringBuilder();
sb.append("<html><head>\n"
...
+ "Session ID: "+session.getId()+"<br />\n"
+ "HostName: <b>"+hostName.toString()+"</b><br />\n"
...
");

```

#### the webapps context

In order to allows sessions so be shared, we need to set the info into the webapp context description 

[context.xml](src/main/webapp/META-INF/context.xml)

```xml
<Context distributable="true" path="/khoolsession"/>
```

### The webapps server (tomcat)

In order to allows sessions so be shared, we need to configure tomcat. I will not paraphrase my senior look at : https://tomcat.apache.org/tomcat-9.0-doc/cluster-howto.html

[tomcat-server.xml](src/main/conf/conf-tomcat-server.xml)

Nevertheless, notice that we need two port to allow the cluster to communicate "45564" and "4000".

Also that tomcat's context need to be distribuable.
<!-- It might be tweak later ?-->

[context.xml](src/main/conf/conf-context.xml)

```xml
<Context distributable="true">
```

### Cluster in kubernetes

## Namespace

The namespace is 'khool-session-manual' different from the one we automated with fabric8 mvn plugin

[010-ns.yaml](src/main/k8s.manual/010-ns.yaml)

## Deployment

The deployement yaml is pretty straightforward, notice the ports and the 3 replicats. Clustering means more then two ;-)

[020-deploy.yaml](src/main/k8s.manual/020-deploy.yaml)
<!-- @todo check for the sessionAffinity into a real cluster -->

## service and ingress

The service map to the pod via selector (label run=khool-session).

[030-svc.yaml](src/main/k8s.manual/030-svc.yaml)

The ingress rules use nginx as ingress controller configuered via annotation.

```yaml
nginx.ingress.kubernetes.io/affinity: "cookie"
nginx.ingress.kubernetes.io/session-cookie-name: "route"
nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
nginx.ingress.kubernetes.io/rewrite-target: /
```

and regular rules that map khool-session-manual.khool-session.info hostname to our service khool-session within the namespace

```yaml
rules:
- host: khool-session-manual.khool-session.info
  http:
    paths:
    - path: /
      backend:
        serviceName: khool-session
        servicePort: 8080
```

[040-ing.yaml](src/main/k8s.manual/030-ing.yaml)

### Manual deployment

Basically, we use maven to build the webapp 'mvn package', docker to build the tomcat image with our configuration and our web app 'docker build -f src/main/k8s.manual/Dockerfile.manual . -t khool-session-manual:0.2', last but not least kubectl to deploy our image in 3 instance into our kubenetes cluster
'kubectl apply -f src/main/k8s.manual/'

```
mvn package &&\
docker build -f src/main/k8s.manual/Dockerfile.manual . -t khool-session-manual:0.2 && \
kubectl apply -f src/main/k8s.manual/
```

