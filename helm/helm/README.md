## Helm

### Helm 解决的问题
K8s 原生 YAML 碎片化、无版本、多环境难以维护、部署流程杂乱的问题。

极简大白话：
- 解决 YAML 太多太散：一堆 deployment/service/ingress 零散文件不好管理；
- 解决 多环境重复写配置：开发 / 测试 / 生产不用重复改 YAML，一套模板复用；
- 解决 应用无版本：能记录版本、升级、一键回滚；
- 解决 依赖混乱：统一管理组件依赖，标准化交付。

### 如何解决的
1. 怎么实现「统一封装应用配置」？
👉 靠 Chart

    把一个应用所有 K8s 资源（Deployment、Service、Ingress、ConfigMap、Secret…）
    全部收拢、打包成一个统一Chart 模板。
    不再散落几十个独立 YAML，整体打包、整体管理。
    模板化写法，公共内容只写一份，杜绝重复复制粘贴 YAML。

2. 怎么实现「多环境部署」？
👉 靠 模板渲染 + Values 变量文件

    Chart 是通用模板，里面留可变占位符；
    不同环境单独维护各自 values.yaml：
        开发：低配、调试开启
        测试：常规规格
        生产：高可用、资源限制、安全配置
    部署时：Chart 模板 + 对应环境 Values = 最终生效 YAML
    一套模板，多套配置，秒切环境，不用改代码、改模板。

3. 怎么实现「版本管理」？
👉 靠 Release 发布记录 + Chart 版本

    每次安装 / 升级，Helm 会在集群生成一个 Release 版本记录
    记录每一次变更：版本、配置、修改内容
    支持：
        版本升级
        一键回滚到老版本
        历史配置比对、变更追溯
        弥补 K8s 原生没有应用级别版本控制的问题。

4. 怎么实现「一键启停 / 一键部署」？
👉 靠 封装好的生命周期命令K8s 原生要 apply 一堆 yaml、顺序还要注意；Helm 一条命令搞定全生命周期：

    helm install 一键部署整套应用
    helm upgrade 一键更新
    helm uninstall 一键彻底卸载
    不用关心资源顺序、零散文件，极简运维。


### 与 ansible 区别

Ansible template：解决「机器配置文件」动态化
Helm Chart+Values：解决「K8s 整套应用」打包、模板化、版本化、多环境交付

### 实践

初始环境：我用的mac m1, 已经安装docker-desktop + enable k8s。
#### 1. 安装helm和创建helm app
```bash
brew install helm
helm create my-nginx
```
生成的初始目录如下：
```bash
carawang@192 helm % tree my-nginx
my-nginx
├── Chart.yaml  # 包信息：应用名、版本、描述
├── charts  # 子依赖包存放目录
├── templates  # k8s yaml 模板（deployment/service 等）
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── httproute.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml # 所有可变配置 = 变量库

4 directories, 11 files
```

#### 2. 把templates下的文件全删除，然后创建ngnix k8s resources
```bash
carawang@192 my-nginx % tree templates 
templates
├── deployment.yaml
└── service.yaml
1 directory, 2 files
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```
#### 3. 将deployment.yaml和service.yaml模版化。

- helm内置变量：
```txt
.Release.Name # helm install 后面写的名字, 生成不重复资源名，避免冲突
.Values # 来自 values.yaml 的所有配置
.Chart # 来自 Chart.yaml 里的信息
.Release.Namespace # 应用部署到 K8s 哪个命名空间
.Release.IsUpgrade 判断当前是安装
.Release.IsInstall # 判断当前是升级（高级用）
```
模版化的后的templates为：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-nginx
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:{{ .Values.image.tag }}
          ports:
          - containerPort: 80
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc
spec:
  type: ClusterIP
  selector:
    app: {{ .Release.Name }}
  ports:
  - port: 80

```
values.yaml为：
```yaml
replicaCount: 1
image:
  tag: "1.25"
```
#### 4. 安装charts并验证
```bash
carawang@192 helm % helm install my-app my-nginx
NAME: my-app
LAST DEPLOYED: Sat Apr 25 12:44:47 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
```
用```helm list```和```kubectl get pods, svc```验证
```bash
carawang@192 helm % helm list
NAME  	NAMESPACE	REVISION	UPDATED                            	STATUS  	CHART               	APP VERSION
my-app	default  	1       	2026-04-25 12:44:47.74188 +0800 CST	deployed	my-nginx-0.1.0	1.16.0     
carawang@192 helm % kubectl get pods,svc
NAME                                READY   STATUS    RESTARTS   AGE
pod/my-app-nginx-654547678b-rc8hb   1/1     Running   0          4m29s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   12d
service/my-app-svc   ClusterIP   10.96.57.219   <none>        80/TCP    4m29s
```
#### 5. 实现多环境部署
```bash
helm install dev-app ./my-nginx -f values-dev.yaml
helm install prod-app ./my-nginx -f values-prod.yaml
```

#### 5. 卸载
```bash
helm uninstall my-app
```

#### 6. 升级和降级
helm history, helm upgrade, helm rollback
```bash
carawang@192 helm % helm history  my-app
REVISION	UPDATED                 	STATUS  	CHART               	APP VERSION	DESCRIPTION     
1       	Sat Apr 25 13:03:14 2026	deployed	my-nginx-chart-0.1.0	1.16.0     	Install complete
carawang@192 helm % helm upgrade my-app my-nginx --set replicaCount=2
Release "my-app" has been upgraded. Happy Helming!
NAME: my-app
LAST DEPLOYED: Sat Apr 25 13:04:04 2026
NAMESPACE: default
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None
carawang@192 helm % helm history  my-app                             
REVISION	UPDATED                 	STATUS    	CHART               	APP VERSION	DESCRIPTION     
1       	Sat Apr 25 13:03:14 2026	superseded	my-nginx-chart-0.1.0	1.16.0     	Install complete
2       	Sat Apr 25 13:04:04 2026	deployed  	my-nginx-chart-0.1.0	1.16.0     	Upgrade complete
carawang@192 helm % helm rollback my-app 1
Rollback was a success! Happy Helming!
carawang@192 helm % helm history  my-app  
REVISION	UPDATED                 	STATUS    	CHART               	APP VERSION	DESCRIPTION     
1       	Sat Apr 25 13:03:14 2026	superseded	my-nginx-chart-0.1.0	1.16.0     	Install complete
2       	Sat Apr 25 13:04:04 2026	superseded	my-nginx-chart-0.1.0	1.16.0     	Upgrade complete
3       	Sat Apr 25 13:04:38 2026	deployed  	my-nginx-chart-0.1.0	1.16.0     	Rollback to 1   
```