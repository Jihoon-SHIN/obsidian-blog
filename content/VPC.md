
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