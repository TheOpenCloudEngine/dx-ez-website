# Install MSAez

* 설치 전, 설치 된 GitLab에 Application 등록하여, OAuth ID와 Secrets 발급이 필요하다.

### GitLab Application 등록

1. 설치된 Gitlab의 Admin 계정으로 접속
2.  Admin Area -> Applications\


    <figure><img src="../.gitbook/assets/Pasted image 20231110122240 (1).png" alt=""><figcaption><p>GitLab Application</p></figcaption></figure>


3. **Add New application Click**
4.  Application 설정\


    <figure><img src="../.gitbook/assets/Pasted image 20231110122407.png" alt=""><figcaption><p>Application 설정 화면</p></figcaption></figure>


5. Application 등록 후, 발급된 ID, Secret은 MSAez Install에 사용되므로, 저장.

### MSAez 설치

1. MSAez 설치는 \[MSAez SourceCode]\([https://github.com/msa-ez/platform](https://github.com/msa-ez/platform)) 해당 소스코드 하위의 on-prem-helm 폴더에서 진행된다.

```bash
$ git clone https://github.com/msa-ez/platform.git
```

***

2. Helm chart Value 수정.

> ```yaml
> # /on-prem-helm/values.yaml
> replicaCount: 1
> image:
>   repository: asia-northeast3-docker.pkg.dev/eventstorming-tool/eventstorming # Image Registry URL
>   eventstorming: evenstorming:v49 # Eventstorming-tool Image URL
>   acebase: acebase:v33 # Acebase Image URL
>
> gitlab: 
> 	url: gitlab.handymes.com # Gitlab URL
> 	oauth: 
> 		id: # Gitlab Application OAUTH ID
> 		secret: # Gitlab Application OAUTH Secrets
>
> db:
>   host: acebase.handymes.com # DB URL
>   port: 443 # 고정
>   name: mydb # 고정
> ```

3. Helm install

```bash
$ cd on-prem-helm
$ helm install msaez .
```

4. 서비스 확인

```bash
# Pod, Service
root@theia-build:/home/kimsanghoon# kubectl get all
NAME                                     READY   STATUS    RESTARTS   AGE
pod/acebase-6c7c8598fd-6fgkp             1/1     Running   0          9d
pod/eventstorming-tool-8554ffc55-h94vd   1/1     Running   0          23h

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/acebase              ClusterIP   10.233.15.103   <none>        80/TCP    21d
service/eventstorming-tool   ClusterIP   10.233.19.127   <none>        80/TCP    21d

```

```bash
# Ingress
root@theia-build:/home/kimsanghoon# kubectl get ing
NAME                     CLASS   HOSTS                  ADDRESS           PORTS     AGE
acebase                  nginx   acebase.handymes.com   000.000.000.000   80, 443   21d
eventstorming-tool-ing   nginx   msa.handymes.com       000.000.000.000   80, 443   21d

```
