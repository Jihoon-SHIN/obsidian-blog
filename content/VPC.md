
### 서버 Region 늘리기
- 머니워크가 3개국에서 22개국으로 배포를 늘리게 되면서, 서버 Region 을 프랑크푸르트에 신설하기로 함
- 이에, mongoDB atlas 도 함께 프랑크 푸르트에 readOnly-node 를 올리려함
- vpc peering 등등 해야된다
- vpc 개념이나 다시 잡고 넘어가자

### VPC ( Virtual Private Cloud )
- VPC는 AWS 클라우드 내에서 논리적으로 격리된 가상 네트워크
- 핵심 구성 요소
	- Subnet : VPC 내의 IP 주소 범위. Public/Private 으로 구분
	- CIDR Block : VPC의 IP 주소 범위
	- Route Table : 트래픽 라우팅 규칙
	- Security Group : 인스턴스 레벨 방화벽
	- VPC Peering : 서로 다른 VPC 간 연결
- 예를 들어, CIRP Block 을 10.0.0.0/16 으로 설정했다고 예를 들어보자. 그럼 16 은 앞에 16비트를 고정한다는 의미가 되므로 이 IP 주소는 10.0.0.0 ~ 10.0.255.255 까지의 범위를 가질 수 있다. 
	- Subnet 은 이 VPC 의 IP 범위를 용도별로 쪼갠 것을 의미한다. 보통 Public/Private 으로 구분하여 라우팅을 분리하여 보안을 챙길 수 있다.
		- 또한 가용영역를 분리하거나 리소스를 격리하여 서비스의 가용성을 높이는 데 사용된다.
	- Public 은 NAT Gateway 로 외부망과 연결 시킬 수 있도록 한다.