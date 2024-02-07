### ingress-config.yaml 파일

- 주소만 적으면 hname-svc-default 서비스의 80번
- 주소 뒤에 /ip 적으면 ip-svc 서비스의 80번 포트로 이동

### ingress.yaml

apiVersion: 사용하는 Kubernetes API의 버전을 나타냅니다. 이 경우에는 v1을 사용하고 있습니다.
kind: Kubernetes 오브젝트의 종류를 나타냅니다. 이 경우에는 서비스(Service)입니다.
metadata: 서비스에 대한 메타데이터를 정의합니다. 여기서는 서비스의 이름과 네임스페이스를 지정하고 있습니다.
spec: 서비스의 스펙을 정의합니다. 포트 및 선택자(selector)와 같은 서비스의 세부 구성이 여기에 포함됩니다.
ports: 서비스가 노출하는 포트를 정의합니다. 여기서는 HTTP 포트와 HTTPS 포트를 정의하고 있습니다. 각 포트에는 이름, 프로토콜, 포트 번호, 대상 포트(targetPort) 및 노드 포트(nodePort)가 지정되어 있습니다.
selector: 서비스가 요청을 전달할 파드를 선택하는 데 사용되는 라벨 셀렉터를 정의합니다. 여기서는 Ingress Nginx 애플리케이션을 가리키는 라벨을 사용하고 있습니다.
type: 서비스의 유형을 정의합니다. 이 경우에는 NodePort로 설정되어 있으며, 클러스터의 각 노드에 해당 포트로 서비스에 액세스할 수 있도록 합니다.

### DMZ 설정
- 외부와 내부를 분리하는 과정

#### 연동하기
- kubectl expose deployment in-hname-pod --name=hname-svc-default --port=80,443
- kubectl expose deployment in-ip-pod --name=ip-svc --port=80,443

### Horizontal Pod Autoscaler 
#### 트래픽 만들어서 어떻게 반응하는지 보자!

- kubectl create deployment hpa-hname-pods --image=sysnet4admin/echo-hname 
- kubectl expose deployment hpa-hname-pods --type=LoadBalancer --name=hpa-hname-svc --port=80  (로드밸런서)
- kubectl get services : 서비스 혹인

#### 평가 지표를 계산하는 서비스 설치 (metrics)
##### 트래피에 어떻게 반응하는지 보기 위함
- kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/metrics-server-0.6.3/metrics-server.yaml
- kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

- kubectl top pods : cpu, 메모리 확인
#### m -> milliunits -> 1000m이면 1개의 cpu를 다 쓰고 있다는 뜻

#### 인위적으로 resource 내용 줄이기  (스팩아 적혀져 있는 문서)
- kubectl edit deployment hpa-hname-pods
- resources 부분 수정해주면 된다.
- 10을 넘으면 파드 추가 생성 , 50은 넘지 않게
- 밑에 부분을 변경해주면 된다 

    resources:
          limits:
            cpu: 50m
          requests:
            cpu: 10m

#### min 최소 파드 , max 최대 파드의 . cpu percent는 cpu 사용량
#### cpu-percent는 cpu 사용량이 50% 넘으면 autoscale 해줘

- kubectl autoscale deployment hpa-hname-pods --min=1 --max=30 --cpu-percent=50

- k get services 하면 ip 주소 나옴
- hpa-hname-svc   LoadBalancer   10.100.30.73       a4b19ceb998ee46fb87f8a68d9709a79-1411192838.ap-northeast-2.elb.amazonaws.com   80:31696/TCP   13m
- 위 결과를 보면 내 ip는 31696으로 할당 받음.
- 인스턴스 ip.31696이랑 / 인스턴스 ip/ip

#### watch를 사용해서 모니터링 하기

- watch -n 1 kubectl top pods

- watch -n 1(시간) 보고싶은 서비스 

#### 부하주기
- import requests
url = "http://3.35.228.89:31696/"

while(True):
    r = requests.get(url)

- 이 코드로 부하주고 watch로 모니터링 함.
- 수평확장을 한다 (트래픽에 따라서)

#### 클러스터 삭제
- eksctl delete cluster --name k8s-encore --region ap-northeast-2

#### Elastic container service repository 생성
- 프라이빗 repos 에서 퍼블릭으롯 생성함

#### Elastic Container Service 이란? 

아마존 Elastic Container Service (Amazon ECS)는 AWS에서 제공하는 관리형 컨테이너 오케스트레이션 서비스입니다. Amazon ECS를 사용하면 Docker 컨테이너를 손쉽게 배포, 관리 및 확장할 수 있습니다. 다음은 Amazon ECS의 주요 특징과 기능입니다:

컨테이너 관리: Amazon ECS를 사용하면 Docker 컨테이너를 쉽게 관리할 수 있습니다. 클러스터에서 실행되는 컨테이너의 수명주기를 관리하고, 자동으로 컨테이너를 시작하고 중지할 수 있습니다.

스케일링: Amazon ECS는 컨테이너 인스턴스의 수를 자동으로 조정하여 애플리케이션의 요구 사항에 맞게 조정할 수 있습니다. 사용자가 정의한 조건에 따라 자동으로 스케일링을 수행하므로 애플리케이션이 항상 최적의 성능을 유지할 수 있습니다.

로드 밸런싱: Amazon ECS는 로드 밸런서와 함께 사용하여 애플리케이션 트래픽을 분산할 수 있습니다. 로드 밸런서를 통해 여러 컨테이너 인스턴스 간에 트래픽을 분배하여 애플리케이션의 가용성을 향상시킬 수 있습니다.

클러스터 관리: Amazon ECS를 사용하면 컨테이너 인스턴스를 클러스터로 그룹화하여 관리할 수 있습니다. 클러스터를 통해 여러 컨테이너 인스턴스를 한데 묶어서 애플리케이션을 쉽게 관리하고 배포할 수 있습니다.

인프라 자동화: Amazon ECS는 AWS 인프라와 통합되어 있으며, AWS 리소스를 사용하여 컨테이너 인스턴스를 자동으로 관리합니다. 이를 통해 사용자는 인프라 구성 및 관리에 대한 복잡한 작업을 최소화하고 컨테이너에 집중할 수 있습니다.

Amazon ECS는 모니터링, 보안, 로깅 등 다양한 기능을 제공하여 안전하고 안정적으로 컨테이너 애플리케이션을 운영할 수 있도록 지원합니다. 따라서 Amazon ECS를 사용하면 컨테이너 기반의 애플리케이션을 쉽게 구축하고 운영할 수 있습니다.
