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

> ./on-prem-helm/values.yaml
>
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
>   port: 443
>   name: mydb
> ```
