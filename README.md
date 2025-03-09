## VM기반 AKS, ACR을 활용한 Jenkins Pipeline ##  

Azure VM을 생성하고, Azure Cloud Shell을 통해 VM에 접속하여 Jenkins 를 설치한 다음, 파이프라인 실행에 필요한 스텍을 단계별로 실행한다.

## 1. Azure VM 생성과 설정 ##
- 서비스 검색 란에 '가상 머신'을 입력하여 VM 생성 화면으로 이동한다.
- (user07 계정의 경우,) 가상 머신 이름을 'user07-vm'으로 입력하고, 나머지는 기본설정으로 둔다.
- (VM Size : 2 vcpu, 8GiB로 생성)
- 만들기’ 클릭 후, 팝업 창에서 '프라이빗 키 다운로드’를 클릭한다.  
![image](https://github.com/user-attachments/assets/8d8b90d6-09ad-4655-b853-6eae9de24934)
- 로컬에 SSH 키가 다운로드 되고, VM이 생성된다.

### 1/2. Jenkins 포트 방화벽 설정 ###
- Jenkins 서버가 사용할 포트(8080)를 인바운드 규칙에 등록한다.
- 생성된 VM 메뉴에서 '네트워킹 > 네트워크 설정'을 클릭한다.
- 규칙 세션에서 '포트 규칙 만들기' > 인바운드 포트 규칙'을 클릭한다.
- 대상 포트 범위에서 8080을 확인하고 '추가'를 클릭한다.

### 2/2. 가상 머신 접속 환경 구성 ###
- 생성된 VM 의 상단 메뉴에서 '연결 > 연결'을 눌러 공용 IP에 접속한다.
- 연결 옵션 중, '권장 - Azure CLI를 사용하는 SSH'를 선택한다.
- 팝업에서 자동 구성설정 완료 후 나타나는 '구성 및 연결' 버튼 바로 위 옵션에 체크하고 클릭한다. 
- 화면 아래쪽에 Azure Cloud Shell이 나타나면 "Bash" 쉘을 클릭해 로딩을 계속한다.
- 프롬프트가 나타나면 'yes'를 입력하고 엔터키를 누른다.
- 시작 시, '스토리지 계정 탑재 안함' 옵션을 선택하고 '적용'을 눌러 클라우드 쉡을 시작한다.
    - Azure Cloud Shell에서 설치한 파일이나, 데이타는 기본적으로 세션 종료 시 삭제되나, Azure VM에 Cloud Shell로 접속한 경우, 세션 종료 후에도 VM에 설치된 데이터는 유지
- VM 공용 IP를 브라우저 새로운 탭에 복사해 두고 클라우드 쉘을 최대화(Maximize) 한다.

## 2. 젠킨스 설치 & 초기 설정 ##

- 생성된 VM은 Ubuntu OS 만 탑재되어 있으므로 필요한 SW 스택들을 하나씩 설치한다.
- 클라우드 쉘에서 젠킨스 런타임인 JDK를 설치한다.
```
sudo apt-get update
sudo apt install openjdk-17-jdk
```

- Jenkins Install Repository를 패키지 매니저에 추가하고 설치한다.
```
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null
		
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins
```
- 설치 확인
```
sudo service jenkins status	
```
 
- 젠킨스는 쉘 스크립트를 통해 상호작용하므로 젠킨스 유저에게 관리자 권한을 추가하고 저장 종료한다.(맨 마지막 커맨드만 실행)
```
# sudo vi /etc/sudoers
# jenkins ALL=(ALL) NOPASSWD: ALL

sudo sed -i '$ a jenkins ALL=(ALL) NOPASSWD: ALL' /etc/sudoers
```

### 젠킨스 초기화 ###

- 생성된 VM의 퍼블릭 IP를 조회하여 IP 주소:8080으로 접속해보면 아래와 같은 화면이 나타난다.
![image](https://github.com/acmexii/demo/assets/35618409/95d5c55f-abb9-4db6-8316-4b5b48964490)

- 아래 커맨드로 Initial AdminPassword를 복사 후, 입력한다.
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- 다음 화면에서 'Install suggested plugins'를 클릭하여 플러그인을 초기화 한다. (10분 내외 소요)
- 다음 화면에서 관리자 정보를 입력하여 계정을 생성(admin/ admin)한다.
- 계정 생성 후, Jenkins를 시작하면 아래와 같이 메인화면이 나타난다.
![image](https://github.com/acmexii/demo/assets/123033598/17b31f5c-5c37-4d58-b31d-fd72583f9b18)


## 3. Cloud Stack & Azure Client 설치 ##

Docker와 Azure Cloud Stack을 설치해 준다.

### 1/4. Docker 설치 및 설정 ###

- 도커를 설치하고 젠킨스에서 도커를 사용할 수 있게 Docker Group에 Jenkins user를 추가해 준다.
```
sudo apt install docker.io
sudo usermod -aG docker jenkins
```

### 2/4. Azure CLI 설치 ###

VM에서 az 커맨드를 사용하기 위해 Azure CLI를 설치한다.
```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash 
az version
```

### 3/4. Kubectl 설치 ###

다음 사이트에서 Kubernetes 서버에 맞는 클라이언트(kubectl)을 다운받아 설치한다.
```
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html

# 클러스터가 1.28 버전일 때,
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.8/2024-04-19/bin/linux/amd64/kubectl
# 클러스터가 1.29 버전일 때,
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.5/2024-01-04/bin/linux/amd64/kubectl
```
- 설치 후, 실행에 필요 사후 작업을 수행한다.
```
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version
```
### 4/4. Maven 설치 ###
```
# Install Maven
wget https://dlcdn.apache.org/maven/maven-3/3.8.8/binaries/apache-maven-3.8.8-bin.tar.gz
tar -xvf apache-maven-3.8.8-bin.tar.gz
sudo mv apache-maven-3.8.8 /opt/
```

## 4. Jenkins Server Setting  ##

Jenkins 서버 설정을 위해 관리 콘솔을 로딩한다.

### 1/4. Maven 설정 ###
- Jenkins 관리 > Tools > Maven installations > Add Maven
- Name에 'Maven'을 입력하고, Install automatically 체크를 해제한다.
- Maven Home에는 /opt/apache-maven-3.8.8을 입력한다.
```
/opt/apache-maven-3.8.8
``` 
- 'Save'를 눌러 저장한다.
![image](https://github.com/acmexii/demo/assets/123033598/3664f7e0-132d-4816-91f5-eaf2475b901b)

### 2/4. Jenkins Plugin 설정 ###
- Jenkins 관리 > Plugins > Available plugins에서 다음을 검색해 설치해 준다.
- 아래 Plug-ins를 찾아서 한꺼번에 설치한다.
    - Maven Integration 
    - Pipeline Maven Integration
    - Docker Pipeline
    - Kubernetes 
    - Kubernetes CLI
    - Azure CLI
    - Azure Credentials

- 모두 찾아 선택한 다음, Install after restart를 클릭하자. (Install 버전 옆 추가 메뉴)
- 재시작에 체크하고, Jenkins를 restart한다.

### 3/4. Azure Credentials 설정 ###
- Service Principal을 생성하기 위해 Azure CLI로 Azure에 인증 한다.
```
az ad sp create-for-rbac --name <ServicePrincipalName> --role contributor --scopes /subscriptions/<SubscriptionID>

# e.g. user07 사용자의 구독명이 '종량제1'인 경우, 해당 구독의 ID를 '구독 서비스'에서 획득해 온다.
az ad sp create-for-rbac --name user07-sp --role contributor --scopes /subscriptions/cb5860b7-b8db-408f-89c2-ca518893e315
```
- az 인증이 필요한 경우, az login을 먼저 실행한다.
- 구독 ID와 Service Principal 생성 후 출력된 appId, password, tenant 값을 테스트 에디터에 기록해 둔다.

- Jenkins 관리 > Credentials에서 Azure Credentails을 설정한다.
- Stores scoped to Jenkins → (global)▽ → Add Credentials을 눌러 Credential 생성화면으로 이동한다.
    - Kind: Username with password
    - Scope: Global
    - Username : appId
    - Password : password
    - ID: 'Azure-Cred' 를 입력한다.
- 'Create'를 눌러 생성을 완료한다.

### 4/4. Github Credentials 설정 ###

- Jenkins 관리 > Credentials에서 Git Credentails을 설정한다.
- Stores scoped to Jenkins → (global)▽ → Add Credentials을 눌러 Credential 생성화면으로 이동한다.
    - Kind: Username with password
    - Scope: Global
    - Username: Github ID
    - Password: Github Token
        - Github Token 은 Githb 사이트에서 생성 후, 발급된 토큰을 붙여넣는다.    
        - GitHub Profile > Settings → Developer Settings → Personal access tokens → Tokens(classic)에서 토큰 생성
        - 기본 repo와 admin:repo_hook을 체크하여 Note에 이름을 입력하고 생성한다.
    - ID: 'Github-Cred' 를 입력한다.
- 'Create'를 눌러 생성을 완료한다.

## 5. Jenkins Pipeline 생성 ##

Jenkins 메인으로 이동하여 대상 서비스의 파이프라인을 생성한다.
- new Item 에서 파이프라인 이름을 입력하고, 'Pipeline' 템플릿 선택 후, 하단의 "OK"를 누른다.
    - 우리는 상품 서비스에 대한 파이프라인을 작성할 것이다. 이름에 'Product Pipeline'을 입력한다.
- Build Triggers 옵션은 'GitHub hook trigger for GITScm polling' 을 선택
    - Git 저장소에 코드를 Push 하면 GitHub이 Webhook 메세지를 Jenkins에 보내고 Jenkins는 이때부터 빌드를 진행
- Pipeline > Definition에서 'Pipeline script from SCM' 을 선택하고 Github 정보를 넣는다.
    - SCM : Git
    - Repository URL : target Git Repo.
    - Credentials : Select Git Credential
    - Branch : master, or Main
    - Script Path : Jenkinsfile
- 설정을 확인하고 '저장' 을 누른다. 
![image](https://github.com/acmexii/demo/assets/123033598/ae59e14e-ca8f-4434-a853-d6ea923474ac)

## 6. CI/CD 대상 서비스 및 리소스 생성 ##

- 상품 서비스 코드 리파지토리에 접속해, 내 Git 으로 복제한다.
    - https://github.com/event-storming/reqres_products에 접속하여 복제(Fork) 한다.
- 서비스 루트에 아래 'Jenkinsfile'을 생성한다.
    - Jenkinsfile은 Jenkins에서 파이프라인을 정의하고 구성하는 데 사용되는 파일로 파이프라인의 모든 단계와 작업을 선언적 또는 스크립트 형식으로 정의한다.
    - Jenkinsfile을 사용하면 코드 형태로 CI/CD 파이프라인을 정의할 수 있어 파이프라인이 복잡하고 고도화된 경우, 유지 및 관리가 용이
- environment 속성 항목의 값을 내 정보에 맞도록 (반드시) 수정한다.

### Jenkinsfile ###

```
pipeline {
    agent any

    environment {
        REGISTRY = 'user07.azurecr.io'
        IMAGE_NAME = 'product'
        AKS_CLUSTER = 'user07-aks'
        RESOURCE_GROUP = 'user07-rsrcgrp'
        AKS_NAMESPACE = 'default'
        AZURE_CREDENTIALS_ID = 'Azure-Cred'
        TENANT_ID = 'f46af6a3-e73f-4ab2-a1f7-f33919eda5ac' // Service Principal 등록 후 생성된 ID
    }
 
    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }
        
        stage('Maven Build') {
            steps {
                withMaven(maven: 'Maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    image = docker.build("${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Azure Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.AZURE_CREDENTIALS_ID, usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                        sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant ${TENANT_ID}'
                    }
                }
            }
        }
        
        stage('Push to ACR') {
            steps {
                script {
                    sh "az acr login --name ${REGISTRY.split('\\.')[0]}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('CleanUp Images') {
            steps {
                sh """
                docker rmi ${REGISTRY}/${IMAGE_NAME}:v$BUILD_NUMBER
                """
            }
        }
        
        stage('Deploy to AKS') {
            steps {
                script {
                    sh "az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${AKS_CLUSTER}"
                    sh """
                    sed 's/latest/v${env.BUILD_ID}/g' kubernetes/deploy.yaml > output.yaml
                    cat output.yaml
                    kubectl apply -f output.yaml
                    kubectl apply -f kubernetes/service.yaml
                    rm output.yaml
                    """
                }
            }
        }
    }
}

```

### Kubernetes 배포 YAML 수정 ###

- Jenkins 파이프라인 실행해 주는 배포 Manifest 상의 이미지 이름의 정합성을 맞춘다.
- Git Root / kubernetes / deploy.yaml의 20행 이미지 수정
```
예시 : user07.azurecr.io/product:latest
```

## 7. Jenkins Webhook 설정 ##

### GitHub에 Jenkins WebHook 등록 ###

Git 리파지토리의 코드 반영에 따라 자동으로 Jenkins 파이프라인이 동작하도록 설정한다.

- 파이프라인 대상 GitHub Repository -> Settings -> Webhooks -> add Webhooks 
- Payload URL: 젠킨스 주소에 /github-webhook/를 붙여 입력한다.
```
http://xx.xx.xx.xx:8080/github-webhook/
```
- Content type : application/json 선택
- 하단의 'Add webhook'을 눌러 설정한 WebHook을 저장한다.


### Jenkins Pipeline 동작확인 ###
- 편의 상, Github Web UI를 활용해 파이프라인이 설정된 대상 프로젝트 Root에서 코드 변경을 가하고, Commit해 본다.
- 실행 중 오류가 발생하면 Console Log를 통해 원인을 분석하고 스크립트를 수정한다.
- 정상적으로 파이프라인이 동작했을 경우, 아래와 같이 성공 실행 로그와 상품 쿠버네티스 객체가 조회 된다.
![image](https://github.com/user-attachments/assets/3148c70d-0f5d-49f8-b75f-39cc570d617b)

### ※ 주문,상품,배송 서비스 일괄 배포
주문, 상품, 배송 서비스가 동일한 레포지토리에 함께 존재할 경우, 레포지토리 Root에 있는 Jenkins 파일 하나로 모든 서비스를 배포하는 샘플은 아래와 같다. 
```
pipeline {
    agent any

    environment {
        REGISTRY = 'user19.azurecr.io'
        SERVICES = 'order,delivery,product' // fix your microservices
        AKS_CLUSTER = 'user19-aks'
        RESOURCE_GROUP = 'user19-rsrcgrp'
        AKS_NAMESPACE = 'default'
        AZURE_CREDENTIALS_ID = 'Azure-Cred'
        TENANT_ID = '29d166ad-94ec-45cb-9f65-561c038e1c7a'
    }

    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }

        stage('Build and Deploy Services') {
            steps {
                script {
                    def services = SERVICES.tokenize(',') // Use tokenize to split the string into a list
                    for (int i = 0; i < services.size(); i++) {
                        def service = services[i] // Define service as a def to ensure serialization
                        dir(service) {
                            stage("Maven Build - ${service}") {
                                withMaven(maven: 'Maven') {
                                    sh 'mvn package -DskipTests'
                                }
                            }

                            stage("Docker Build - ${service}") {
                                def image = docker.build("${REGISTRY}/${service}:v${env.BUILD_NUMBER}")
                            }

                            stage('Azure Login') {
                                withCredentials([usernamePassword(credentialsId: env.AZURE_CREDENTIALS_ID, usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                                    sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant ${TENANT_ID}'
                                }
                            }

                            stage("Push to ACR - ${service}") {
                                sh "az acr login --name ${REGISTRY.split('\\.')[0]}"
                                sh "docker push ${REGISTRY}/${service}:v${env.BUILD_NUMBER}"
                            }

                            stage("Deploy to AKS - ${service}") {
                                
                                sh "az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${AKS_CLUSTER}"

                                sh 'pwd'
                                
                                sh """
                                sed 's/latest/v${env.BUILD_ID}/g' kubernetes/deployment.yaml > output.yaml
                                cat output.yaml
                                kubectl apply -f output.yaml
                                kubectl apply -f kubernetes/service.yaml
                                rm output.yaml
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('CleanUp Images') {
            steps {
                script {
                    def services = SERVICES.tokenize(',') 
                    for (int i = 0; i < services.size(); i++) {
                        def service = services[i] 
                        sh "docker rmi ${REGISTRY}/${service}:v${env.BUILD_NUMBER}"
                    }
                }
            }
        }
    }
}
```
- Jenkinsfile을 설정한 후, 파이프라인을 실행하면 각 Microservice별로 Jenkins Pipeline이 동작하고 연결된 클러스터에 배포되는 것을 확인할 수 있다.
