概述
本文利用jenkins在k8s中简单实践了一下CI/CD，部分实验内容来自Set Up a CI/CD Pipeline with Kubernetes ，除此外，还试验了一把利用jenkins kubernetes plugin实现动态分配资源构建。
在kubernetes中简单实践jenkins
首先简单介绍下jenkins,jenkins是一个java编写的开源的持续集成工具。具体来说，他可以将软件构建，测试，发布等一系列流程自动化，达到一键部署的目的。在进行本实验前，首先要有一个k8s环境，这里不再赘述。
部署jenkins
这里存储用的是nfs，所以先创建一个pv和pvc：
jenkins-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
 # namespace: kube-ops
  labels:
    name: pv-01
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    #FIXME: use the right IP
    server: 192.168.3.175
    path: "/data/nfs"


jenkins-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: kube-ops
  labels:
    name: pv-01
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ""
  resources:
    requests:
      storage: 4Gi

部署jenkins:
首先创建两个configmap
kubectl create configmap image-registry-auth 
--from-file=/root/.docker/config.json  --namespace=kube-ops


kubectl create configmap kube-configmap 
--from-file=/root/.kube/config --namespace=kube-ops


jenkins-deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: kube-ops
  labels:
    app: jenkins
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: jenkins
        tier: jenkins
    spec:
      containers:
      - image: chadmoon/jenkins-docker-kubectl:latest
        name: jenkins
        securityContext:
          privileged: true
        ports:
        - containerPort: 8080
          name: jenkins
        - containerPort: 50000
          name: agent
          protocol: TCP
        volumeMounts:
        - name: docker
          mountPath: /var/run/docker.sock
        - name: jenkins-persistent-storage
          mountPath: /root/.jenkins
          subPath: jenkins
        - name: kube-config
          mountPath: /root/.kube/
        - name: image-registry
          mountPath: /root/.docker
      volumes:
      - name: docker
        hostPath:
          path: /var/run/docker.sock
      - name: jenkins-persistent-storage
        persistentVolumeClaim:
          claimName: jenkins-pvc
      - name: kube-config
        configMap:
          name: kube-configmap
        #hostPath:
        #  path: /root/.kube/
      - name: image-registry
        configMap:
          name: image-registry-auth

简单解释一下：
该镜像除了安装jenkins，还装了docker cli（与host docker daemon交互），kubectl（与k8s apiserver交互）
容器开了两个端口，一个用于web-ui,一个用于后面实验jenkins kubernetes plugin时与JNLP slave agents 交互。
挂载了四个volume，依次是，一个用于docker cli，一个用于存储jenkins数据，一个用于kubectl与k8s交互验证，最后挂载了一个configmap，与image registry（我们用的harbor）交互验证。
部署jenkins service & ingress:
jenkins-service-ingress.yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-web-ui
  namespace: kube-ops
  labels:
    app: jenkins
spec:
  ports:
    - port: 80
      targetPort: 8080
      name: web-ui
    - port: 50000
      targetPort: 50000
      name: agent
  selector:
    app: jenkins
    tier: jenkins
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-web-ui
  namespace: kube-ops
spec:
  rules:
  - host: jenkins.hitales.ai
    http:
      paths:
      - backend:
          serviceName: jenkins-web-ui
          servicePort: 80

完成上述所有操作后，查看一下对应的pod 是否running。
配置pipeline
按照ingress的配置，修改本地host，然后浏览器中输入http://jenkins.com ，就进入到jenkins 的web-ui了。
按照提示创建完用户后，开始进入CICD 的实验环节：
新建一个item，命名并选中pipeline:

pipeline 配置如下：

在Git Repository URL部分添加github url,这里用的是我的github-test-kubernetes-ci-cd，是直接fork自kubernetes-ci-cd，并做了一些更改,之后保存就可以了。
进入刚创建的item，点击立即构建：

之后就可以看到构建信息了，如果出错也可以查看对应步骤的log。
同时，我们的应用也已经部署到k8s中了。

步骤详解
接下来看一下点击“立即构建”后发生了什么，点击后，jenkins首先是从github检出项目代码，然后根据检出的项目中根目录下的Jenkinsfile进行项目构建，看下该项目的Jenkinsfile。
node {

    checkout scm

    env.DOCKER_API_VERSION="1.23"
    
    sh "git rev-parse --short HEAD > commit-id"

    tag = readFile('commit-id').replace("\n", "").replace("\r", "")
    appName = "hello-kenzan"
    registryHost = "harbor.hitales.ai/cicd/"
    imageName = "${registryHost}${appName}:${tag}"
    env.BUILDIMG=imageName

    stage "Build"
    
        sh "docker build -t ${imageName} -f applications/hello-kenzan/Dockerfile applications/hello-kenzan"
    
    stage "Push"

        sh "docker push ${imageName}"

    stage "Deploy"

        sh "sed 's#127.0.0.1:30400/hello-kenzan:latest#'$BUILDIMG'#' applications/hello-kenzan/k8s/deployment.yaml | kubectl apply -f - --validate=false"
        sh "kubectl rollout status deployment/hello-kenzan"
}


可以看到，Jenkinsfile定义了三个阶段，第一个阶段是“Build”,这个阶段是根据给定的Dockerfile创建一个镜像，第二个阶段“Push”,把生成的镜像push到我们的镜像仓库中，最后一个阶段是”Deploy”，编辑了一下deployment.yaml模板，然后调用kubectl命令进行部署。
看一下“Build”阶段的dockerfile:

FROM nginx:latest

COPY index.html /usr/share/nginx/html/index.html
COPY DockerFileEx.jpg /usr/share/nginx/html/DockerFileEx.jpg

EXPOSE 80

就是一个很简单的nginx应用。
再看下deployment.yaml：
apiVersion: v1
kind: Service
metadata:
  name: hello-kenzan
  labels:
    app: hello-kenzan
spec:
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
  selector:
    app: hello-kenzan
    tier: hello-kenzan
  type: NodePort

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello-kenzan
  labels:
    app: hello-kenzan
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: hello-kenzan
        tier: hello-kenzan
    spec:
      containers:
      - image: 127.0.0.1:30400/hello-kenzan:latest
        name: hello-kenzan
        ports:
        - containerPort: 80
          name: hello-kenzan



这里服务发现使用的nodeport，可以改为其他方式（比如ingress）。
接下来，可以试着修改一下index.html，然后push到github中，再构建一下，看看内容是否改变。
利用jenkins kubernetes plugin实现动态分配资源构建
在上述实例中，我们利用jenkins实现了一个小应用的CI/CD，这个小应用非常简单，“build”阶段就是直接在本地调用host docker构建的镜像，设想一下，如果这个应用需要编译，需要测试，那么这个时间就长了，而且如果都在本地构建的话，一个人使用还好，如果多个人一起构建，就会造成拥塞。
为了解决上述问题，我们可以充分利用k8s的容器编排功能，jenkins接收到任务后，调用k8s api，创造新的 agent pod，将任务分发给这些agent pod，agent pod执行任务，任务完成后将结果汇总给jenkins pod，同时删除完成任务的agent pod。
为了实现上述功能，我们需要给jenkins安装一个插件，叫做jenkins kubernetes plugin。
插件安装与配置
安装比较简单，直接到jenkins 界面的系统管理，插件管理界面进行安装就可以了。
安装好之后，进入系统管理—–>系统设置，最下面有一个“云”，选择“新增一个云”—->kubernetes。


这里没有配置k8s，因为如果不配置api-server的话，jenkins会默认使用~/.kube/config下的配置，而我们已经在~/.kube/config做过配置了，所以这里就不做了。Jenkins URL我们使用的是集群内的服务地址。
再往下看 kubernetes pod template配置：

这个pod tempalte就是之后我们创建 agent使用的模板，镜像使用“jenkins/jnlp-slave:alpine”，配置完成后，点击保存。
上图pod的模板名称为jenkins-slave，Container的模板名称为jnlp。这里有非常重要的两点要注意：
当Container的模板名称为jnlp的时候，jenkins-slave才会使用下面配置的docker镜像来启动pod，如果不为jnlp，则会使用默认的镜像jenkins/jnlp-slave:alpine
当使用自定义的docker镜像来启动jenkins slave pod的时候，下面的command to run（默认值是 sh -c）和arguments to pass to the command(默认值是cat)两个值需要清空。否则会出现jenkins slave jnlp连接不上master的情况，尝试100次连接之后销毁pod，然后再创建一个pod继续尝试连接，无限循环。
然后还要配置一下agent与jenkins通信的端口，在系统管理—->Configure Global Security，指定端口为我们之前设定的5000端口：

简单测试
配置完成后做一个简单的测试。
新建一个item，这里选择“构建一个自由风格的软件项目”：

配置时注意在General部分有一个restrict：


Label Expression就写之前我们k8s podtemplate 的label。
在构建部分我们写一个简单的测试命令：echo TRUE

点击立即构建，如果成功的话，我们在“管理主机”模块会看到新增了一个主机：


同时，也会在k8s中发现新创建了一个名为jnlp-slave-8bq5m的pod。
任务结束后，pod删除，主机消失，在console output 会看到执行结果：

出现问题总结
jnlp-slave pod创建失败
查看pod日志，发现是连接不上jenkins ，通过修改Configure Global Security的启用安全，TCP port for JNLP agents
指定端口解决。
jnlp-slave pod 无法删除
因为我们执行构建后，如果 jnlp-slave pod创建失败，它会不断的尝试创建新的pod，并试图连接jenkins，一段时间后，就会创造很多失败的jnlp-slave pod。如果遇到这种情况，需要尽早中断任务并删除失败的pod。
在删除某个pod时 ，该pod一直处于termating阶段，kubectl delete无法删除。后来参考Pods stuck at terminated status，使用如下命令解决：
1 kubectl delete pod NAME --grace-period=0 --force
参考文章
Set Up a CI/CD Pipeline with a Jenkins Pod in Kubernetes
Achieving CI/CD with Kubernetes
容器时代CI/CD平台中的Kubernetes调度器定制方法
Jenkinsfile使用
安装和设置 kubectl
jenkins-kubernetes-plugin
基于Kubernetes 部署 jenkins 并动态分配资源
使用Kubernetes-Jenkins实现CI/CD
本篇文章由pekingzcc采用知识共享署名-非商业性使用 4.0 国际许可协议进行许可,转载请注明。
END

