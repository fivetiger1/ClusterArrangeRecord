#    kubernetes Used
##   1. 使用组件说明
        一般来说使用的组件资源是由yaml文件通过kubectl命令进行创建更新和管理，那么其实更多的是在于yaml文件的配置上，我们可以通过yaml文件的kind字段判断资源类型。另外，组件的使用也依靠于镜像的构建，下面我们就从镜像的构建推送开始说起k8s中组件资源的使用
##   2. 镜像仓库搭建和使用
###  2.1 镜像仓库部署
        这里使用的是阿里镜像仓库个人版本，具体的开通流程看阿里文档（点几下就可以在这里不做演示），参考链接https://cr.console.aliyun.com/cn-hangzhou/instance/repositories
###  2.2 镜像仓库使用
        镜像仓库使用包括镜像构建，镜像推送，镜像拉取。我这里使用的是go简单测试样例，目录规划可以参考具体项目，需要说明的是，容器化服务不需要本身服务器支持语言的依赖，在构建镜像过程中引入对应的父镜像即可。
        # mkdir -p /home/go/dockerBuild/

        # cat main.go
        package main

        import (
            "fmt"
            "net/http"
            "os"
            "io"
        )

        func hello(w http.ResponseWriter, r *http.Request) {
                host, _ := os.Hostname()
                message := os.Getenv("MESSAGE")
                namespace := os.Getenv("NAMESPACE")
                dbURL := os.Getenv("DB_URL")
                dbPassword := os.Getenv("DB_PASSWORD")
                io.WriteString(w, fmt.Sprintf("[v6] Hello, Helm! Message from helm values: %s, From namespace: %s, From host: %s, Get Database Connect URL: %s, Database Connect Password: %s", message, namespace, host, dbURL, dbPassword))
        }

        func main() {
            http.HandleFunc("/", hello)
            http.ListenAndServe(":3000", nil)
        }
####    当前go项目处于初学者状态，样例主要是启动一个go程序，端口为3000，访问/hello目录 打印出格式化文本（包括欢迎样式，pod所在namespace，pod名称，待读取其他yaml配置的数据库url和数据库密码）
        # cat Dockerfile
        FROM golang:1.15-alpine as builder
        WORKDIR /app
        COPY . .
        RUN go build -o main .

        FROM alpine:3.12
        WORKDIR /app
        COPY --from=builder /app/main .
        ENTRYPOINT ["./main"]
####    这个dockerfile配置前半段是指获取golang编译环境的镜像，在这基础上进行编译，后半段是指将编译后的二进制文件拷贝到一个linux镜像中进行运行
        # docker build -t registry.cn-hangzhou.aliyuncs.com/goproject/k8s_project:go-app-v7 .
####    这个命令是指使用当前目录构建镜像，并打标签，以":"分割，前半部分为镜像名称，后半部分为tag名称
        # docker push  registry.cn-hangzhou.aliyuncs.com/goproject/k8s_project:go-app-v7
####    目前我们已经在远程镜像仓库推送了我们自定义的镜像，在后面集群部署服务时，就可以通过这个路径来拉取对应的镜像，但是有一个前提是，其他节点也需要统一docker login 一下
##   3. ingress组件使用
####    ingress是暴露pod服务的访问方式的优势，具体划分流量和平滑升级的优势，常用的class是nginx，是一种七层的代理。相比较于service自有的代理NodePort报漏方式访问，管理上和安全上效果比较明显，后者是基于NodeIP+端口来进行访问是一种四层的代理，因此会暴露真实的NodeIP有节点安全的隐患，其次在单台node上服务多的情况下，node的端口维护也是一个问题。因此我们在自用测试环境就直接使用ingress来暴露服务的访问。
####    ingress，nginx class的实现原理，ingress本身就是一个nginx-pod，只是他通过控制器的方式，在ingress配置有更新时，动态的更新ingress所代表的nginx-pod中，因此ingress部署需要控制器的支持，其次是ingress可以配置对应的更新策略可以实现金丝雀发布，蓝绿发布等模式，主要都是通过流量控制操作完成。
        # cat /home/go/project/hellok8s.yaml
        apiVersion: v1
        kind: Service
        metadata: 
          name: service-hellok8s-clusterip
        spec:
          type: ClusterIP
          selector:
            app: hellok8s
          ports:
            - port: 3000
              targetPort: 3000

        --- 

        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: hellok8s-deployment
        spec:
          replicas: 3
          selector:
            matchLabels:
              app: hellok8s
          template:
            metadata:
              labels:
                app: hellok8s
            spec:
              containers:
              - name: hellok8s-container
                image: registry.cn-hangzhou.aliyuncs.com/goproject/k8s_project:go-app-v3
####    这个我们来代表ingress代理的第一个后端服务，也就是由前面构建的go项目镜像启动的服务，这段配置yaml格式分界线前是该服务的service，kind是该yaml配置的组件类型，metadata.name是该组件的名称，spec字段是该组件的具体配置（根据组件的特性来定义）, spec.type是指 service支持的类型，这里使用的cluster是为了集群内部访问的方式，他还有NodePort(IP+端口)和loadbalancer（云负载均衡器）,ExternalName（域名指向型，不支持端口），metadata.selector是一个标签选择器，也就是对应下面Deployment对应的标签，metadata.ports代表着端口映射，将service的3000端口流量转发到目标pod(targetPort)3000端口流量上。分界线后则是Deploment组件，是一个pod副本控制器，用于个性化创建自己的pod在这里可以添加存活和就绪探针，这里spec.replicas 是指pod创建的副本数量，在生产环境中，可以通过该字段和strategy配置来控制每次apply更新的数量保证有最大创建（maxSurge）和最大不正常（maxUnavailable）结合维持服务访问，这里的spec.metadata.labels是一个标签选择创建，spec.template.spec则是具体的pod创建要求，名称和镜像来源。
        # kubectl apply -f /home/go/project/hellok8s.yaml

        # cat /home/go/project/nginx.yaml
        apiVersion: v1
        kind: Service
        metadata:
        name: service-nginx-clusterip
        spec:
        type: ClusterIP
        selector:
            app: nginx
        ports:
        - port: 4000
            targetPort: 80

        ---

        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: nginx-deployment
        spec:
        replicas: 2
        selector:
            matchLabels:
            app: nginx
        template:
            metadata:
            labels:
                app: nginx
            spec:
            containers:
            - name: nginx-container
                image: nginx
####    这个是用来划分流量的第二个服务，nginx，在这里和ingress选择的class-nginx不一样，不过放心后续描述中会特别区分，在这里基本配置和上面的hellok8s基本一致这里就不再赘述。
        # kubectl apply -f /home/go/project/nginx.yaml

        # cat /home/go/project/ingress.yaml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: hello-ingress
          annotations:
            nginx.ingress.kubernetes.io/ssl-redirect: "false"
        spec:
          ingressClassName: nginx
          rules:
            - http:
                paths:
                - path: /hello
                    pathType: Prefix
                    backend:
                      service:
                        name: service-hellok8s-clusterip
                        port:
                          number: 3000
                - path: /
                    pathType: Prefix
                    backend:
                      service:
                        name: service-nginx-clusterip
                        port:
                          number: 4000
####    这是一个简单的ingress配置目的是通过ingress-class-nginx模块通过标签指向特定的两个sercice 3000端口和4000端口，metadata.annotations是指忽略ssl证书关闭https跳转只做http访问支持，spec.ingressClassName是是用的ingress-class模块这里使用的是nginx，spec.rules是具体的流量划分规则，spec.rules[0].paths是指具体的后端指向，其中pathType这里使用的是前缀匹配模式（Prefix），还有精确匹配模式（Exact）。而后面的banckend字段这里使用的是service也是常用的一种后端模式，就是指定service组件（使用servicename和端口来确认服务）。但是还有另外一种模式Resource，这种是较新的后端类型，支持集群内任意种资源访问。
        # kubectl apply -f /home/go/project/ingress.yaml

        # cat /home/go/project/deploy.yaml
        spec:
          externalTrafficPolicy: Cluster
          externalIPs: ['192.168.1.13']
        
                #image: registry.k8s.io/ingress-nginx/controller:v1.3.1@sha256:54f7fe2c6c5a9db9a0ebf1131797109bb7a4d91f56b9b36        2bde2abd237dd1974
                image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.3.1
                #image: registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.3.0@sha256:549e71a6ca248c5abd51cdb73dbc3083df62c        f92ed5e6147c780e30f7e007a47
                image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.3.0
                #image: registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.3.0@sha256:549e71a6ca248c5abd51cdb73dbc3083df62c        f92ed5e6147c780e30f7e007a47
                image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.3.0
####    这里只截出了修改的部分，完全的yaml文件过长 不过不需要我们从零开始写，直接下载后修改运行即可，这里面修改的第一部分是默认的ingress代理模式是NodeBalancer,我们修改修改成service,并且分配一个ingress的访问ip，事实上ingress也是起一个service来使用，他指向的也是nginx-pod（这里的nginx-pod以及service和上面作为后端服务的不一样，这里是作为ingress-class-nginx使用的），而具体的ingress-nginx-pod则根据配置文件指向具体的后端服务。修改的第二部分就是容器镜像源，调整为国内镜像源。
        # kubectl apply -f /home/go/project/deploy.yaml
####    查看pod时会发现属于ingress控制器的两个pod并没有处于runing状态，不要紧张，这两个pod属于job类型pod,特殊情况下才会运行，ingress-nginx-admission-create 是在创建ingress控制器资源验证ingress格式以及设置初始状态，保证ingress控制器能正确的处理新的ingress资源，而ingress-nginx-admission-patch 是在控制器运行时，检测到ingress配置更新，对ingress资源进行必要的修改或验证（例如修改后服务路由不会冲突，更新过程中保持服务的连续性）。只要状态处于completed中就是正常的。
        # kubectl get ingress 
####    在这里可以看到ingress已经存在，并且已分配地址，我这里选择的地址是属于该网段的一个未使用ip，这个是需要配置到某个节点上。然后我们通过访问ingress.yaml中已配置好的路径/hello和/进行访问即可验证ingress已经正常部署，划分流量。
##   4. ingress发布模式，金丝雀发布(问题记录，configmap怎么不重启pod情况下动态修改)

