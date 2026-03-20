---
title: "AWS Certified Developer - Associate 취득 후기"
date: 2020-10-04
categories: 
  - aws
image: "../../images/aws.webp"
tags:
  - aws
  - certification
  - cloud
translationKey: "posts/aws-certification-associate-developer"
---

입사한 뒤 스스로 세운 목표는 1년에 하나 이상의 자격을 따는 것입니다. 전직 전에는 엔지니어로서의 기본기가 부족하다고 느꼈고, 무엇부터 공부해야 할지 막막한 경우도 많았기 때문입니다. 그래서 자격 공부를 하면서 최소한 시험 범위라도 꾸준히 익히고, 스스로의 실력을 조금씩 올려 보자는 생각을 하고 있습니다.
그런 흐름 속에서 이번에는 AWS 자격을 응시했습니다. AWS 자격 종류는 다음 이미지와 같고, 제가 응시한 것은 개발자 - 어소시에이트입니다. 업무에서 AWS의 기본 서비스(EC2와 S3 등)를 다뤄 본 적도 있고, Azure에서도 Functions나 Queue를 이용한 간단한 서버리스 앱을 만들어 보거나 Bastion 같은 서비스를 사용해 본 경험도 있어서 크게 고민하지 않고 개발자 트랙을 선택했습니다. 다만 막상 풀어 보니, 조금 써 본 정도로 쉽게 도전할 수 있는 시험은 아니라는 걸 실감했습니다…
![AWS Certification](aws_certifications.webp)
*출처: AWS 공식 사이트 - [AWS 인증](https://aws.amazon.com/ko/certification/?nc1=h_ls)*
어쨌든 지난 Java 자격 후기 때와 마찬가지로, 이번에도 제 기준에서 시험이 어땠는지 간단히 정리해 보려 합니다.
## 시험의 전체 인상

시험 방식만 놓고 보면, 학생 때 보던 암기형 시험에 가까웠습니다. 자격시험이 기본적으로는 응시자의 지식을 확인하는 자리이니, AWS 서비스의 사양과 특징을 묻는 형식이 되는 건 자연스럽습니다. 다만 실제 문제는 "이런 애플리케이션을 만들려면 어떤 서비스 조합을 써야 하는가?"보다는 "이 요구사항이라면 어떤 CLI 명령을 써야 하는가?"처럼 세부적인 지식을 더 직접적으로 묻는 편이었습니다. 특정 서비스를 설정할 때 쓰는 파일 이름을 고르게 하는 문제도 있었습니다.

AWS는 서비스가 워낙 많고, 역할이 겹치는 영역도 적지 않습니다. 그래서 각 서비스 특성을 정확히 모르면 설계도 구현도 애매해질 수밖에 없고, 시험도 자연히 이런 방향으로 흘러가는 것 같습니다. 특히 등급이 올라갈수록 개발 효율과 비용까지 함께 판단해야 하니, 서비스별 세부 사양까지 익혀 둘 필요가 있습니다.

다만 개인적으로는 자격 이름이 `Developer`인 만큼, 조금 더 개발자다운 질문이 있었어도 좋지 않았을까 하는 생각이 들었습니다. 예를 들어 Lambda 문제라면, 트래픽 패턴이나 처리량에 따라 어느 언어가 더 적합한지 같은 질문이 나와도 흥미로웠을 것 같습니다.
## 어떻게 준비했는가

저는 먼저 어소시에이트 3종 자격을 개괄한 책을 샀지만, 그 책은 어디까지나 서비스 소개에 가까워서 시험 대비용으로는 조금 부족했습니다. 그래서 [Udemy](https://www.udemy.com/)에서 모의고사 5회분 강의를 따로 구매했습니다. 다만 이것도 개발자용 시험만 딱 겨냥했다기보다 프로페셔널 레벨 내용까지 조금 섞여 있는 인상이었습니다. 체감상 실제 시험보다 Udemy 모의고사가 더 어려웠습니다. 모의고사도 종류가 여러 개라 어떤 걸 고르느냐에 따라 실제 시험에 대한 인상도 꽤 달라질 것 같습니다.

개인적으로는 난도가 조금 높은 모의고사를 사서 반복해서 푸는 방식이 가장 효율적이었습니다. AWS 자격은 서로 겹치는 지식이 적지 않아서, 한 자격을 준비하는 과정이 다른 자격에도 그대로 도움이 됩니다. 그 밖에도 AWS 공식 온라인 강의와 문서를 함께 봤습니다. 공식 [트레이닝 코스](https://aws.amazon.com/ko/training/course-descriptions/)에서 동영상과 PDF 자료를 제공하니 한 번쯤 훑어 보는 걸 추천합니다.

또 AWS 서비스는 약칭으로 자주 불리기 때문에 이름만 보고는 무슨 서비스인지 감이 잘 안 올 때가 많습니다. 그래서 적어도 모의고사에 자주 등장하는 서비스 정도는 자신만의 요약 노트를 만들어 두는 편이 좋습니다. 예를 들어 [EBS](https://aws.amazon.com/ko/ebs)와 [EFS](https://aws.amazon.com/ko/efs)는 둘 다 스토리지지만, [ECS](https://aws.amazon.com/ko/ecs)는 컨테이너 서비스입니다. [SQS](https://aws.amazon.com/kr/sqs)와 [SNS](https://aws.amazon.com/kr/sns)는 메시징이고, [STS](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)는 보안 쪽입니다.

결국 이 시험은 어느 정도 암기력도 필요합니다.
## 무엇을 주로 묻는가

이제 실제 시험 내용을 보겠습니다. AWS 공식 [소개 페이지](https://aws.amazon.com/korea/certification/certified-developer-associate/)에는 **추천되는 지식과 경험**이 정리돼 있지만, 어디까지나 큰 범위를 보여 주는 수준입니다. 실제 시험에서 어떤 형태로 묻는지는 샘플 문제나 모의고사를 직접 풀어 봐야 감이 옵니다.

저는 주로 모의고사로 준비했는데, 어떤 문제가 개발자 시험 성격과 가장 비슷한지 끝까지 확신이 서지는 않았습니다. 그래서 가능하다면 AWS 공식 모의고사도 함께 보는 편이 좋겠다는 생각이 들었습니다.

어쨌든 실제 시험에서 자주 나온다고 느낀 주제는 대체로 아래와 같았습니다.
### 서버리스 아키텍처

가장 많이 나온 패턴은 서버리스 아키텍처 관련 문제였습니다. 예를 들어 [AWS Lambda](https://aws.amazon.com/ko/lambda)와 [Amazon DynamoDB](https://aws.amazon.com/ko/dynamodb), [Amazon Kinesis](https://aws.amazon.com/kr/kinesis/)를 조합해 애플리케이션을 만든 상황을 제시한 뒤, 여기서 주의할 점이나 장애 대응 방법, 설정 방식 등을 묻는 식입니다.

즉 "서버리스가 무엇인가" 정도만 이해해서는 부족합니다. 적어도 이 세 서비스는 세부 설정까지 한 번 정리해 두는 편이 좋습니다. Lambda의 의존성 배포 방식, Kinesis 샤드 설정, DynamoDB의 RCU/WCU 같은 개념은 특히 자주 부딪히는 느낌이었습니다.

또 서버리스라고 해도 엔드포인트 설계나 보안 설정은 일반적인 애플리케이션 개발과 크게 다르지 않습니다. 그래서 [Amazon API Gateway](https://aws.amazon.com/ko/api-gateway) 설정, S3 활용, CLI 명령 사용법처럼 개발 실무와 맞닿아 있는 문제도 함께 나왔습니다.
### 보안

보안도 빠지지 않습니다. 주로 [IAM](https://aws.amazon.com/ko/iam)의 정책, 역할, 그룹 설정과 [KMS](https://aws.amazon.com/ko/kms)를 이용한 키 관리가 중심이었습니다. Lambda 비중이 높은 시험인 만큼, Lambda 권한 설정과 연결된 문제도 자연스럽게 함께 나옵니다.
## 로그 및 모니터링

애플리케이션에 장애가 생겼을 때 어떻게 추적할지, 어떤 요구사항이면 어떤 모니터링 설정을 해야 할지도 자주 묻습니다. 예를 들어 [X-Ray](https://aws.amazon.com/x-ray) 데몬 설정 방법이나 [CloudWatch](https://aws.amazon.com/kr/cloudwatch)와 [CloudTrail](https://aws.amazon.com/kr/cloudtrail)의 차이를 구분하는 문제가 나옵니다.

### Code 계열 서비스

CI/CD 관련 문제도 나옵니다. [CodePipeline](https://aws.amazon.com/ko/codepipeline), [CodeDeploy](https://aws.amazon.com/ko/codedeploy), [CodeBuild](https://aws.amazon.com/ko/codebuild), [CodeStar](https://aws.amazon.com/ko/codestar)의 역할만 정확히 구분해도 꽤 도움이 됩니다.

다만 기능이 [Elastic Beanstalk](https://aws.amazon.com/ko/elasticbeanstalk)나 [CloudFormation](https://aws.amazon.com/ko/cloudformation)과 일부 겹쳐 보일 수 있어서, 배포 상황에 따라 무엇을 선택해야 하는지 묻는 문제가 있었던 것으로 기억합니다. 이런 차이는 한 번 표로 정리해 두면 편합니다.

저는 아래 이미지를 보고 개념 정리에 꽤 도움을 받았습니다.
![AWS Code Services](aws_code_services.webp)
*출처: AWS 공식 사이트 트레이닝 코스 `Exam Readiness: AWS Certified Developer - Associate (Digital)`*
이 부분은 AWS 공식 동영상에서 비교적 잘 설명해 주니 한 번쯤 들어 보는 걸 추천합니다.
## 참고로 좋은 정보

그 밖에 가볍게라도 기억해 두면 도움이 되는 것들을 적어 보면 다음과 같습니다.
### 이름만 알고 싶은 서비스

깊게 묻지는 않지만, 이름과 용도 정도는 알아 두면 좋은 서비스도 있습니다. 예를 들면 [OpsWorks](https://aws.amazon.com/ko/opsworks), [Step Functions](https://aws.amazon.com/ko/step-functions), [Systems Manager](https://aws.amazon.com/kr/systems-manager)의 Parameter Store 같은 것들입니다. 다만 여기까지 완벽하게 외우려 들면 공부 범위가 너무 넓어집니다. 우선순위를 잘 나눠서 보는 편이 낫습니다.
### AWS Elastic Load Balancer

[ELB](https://aws.amazon.com/ko/elasticloadbalancing)도 문제에 등장합니다. 저도 처음에는 ALB, NLB, CLB 차이가 잘 안 잡혀서 꽤 헷갈렸습니다. AWS 자격이 처음이거나 ELB를 실무에서 거의 안 써 봤다면, 아래 정도는 정리해 두는 편이 좋습니다. 이 내용은 Cloud Practitioner 쪽에서도 자주 나옵니다.
| 항목 | ALB | NLB | CLB |
|---|---|---|---|
| 프로토콜 | HTTP, HTTPS | TCP, TLS | TCP, SSL/TLS, HTTP, HTTPS |
| 플랫폼 | VPC | VPC | EC2-Classic, VPC |
| 헬스 체크 | O | O | O |
| CloudWatch 지표 | O | O | O |
| 로깅 | O | O | O |
| Zonal Fail-Over | O | O | O |
| Connection Draining | O | O | O |
| 같은 인스턴스의 여러 포트로 분산 | O | O | - |
| WebSockets | O | O | - |
| 대상(IP 주소) | O | O | - |
| 대상(Lambda) | O | - | - |
| 로드 밸런서 삭제 방지 | O | O | - |
| 경로 기반 라우팅 | O | - | - |
| 호스트 기반 라우팅 | O | - | - |
| Native HTTP/2 | O | - | - |
| Idle Connection Timeout 설정 | O | O | - |
| Zone 간 분산 | O | O | O |
| SSL 오프로드 | O | O | O |
| SNI(Server Name Indication) | O | - | - |
| Sticky Session | O | - | O |
| 백엔드 서버 암호화 | O | O | O |
| 고정 IP | - | O | - |
| Elastic IP 주소 | - | O | - |
| 원본 IP 주소 유지 | - | O | - |
| 리소스 기반 IAM 권한 | O | O | O |
| 태그 기반 IAM 권한 | O | O | - |
| Slow Start | O | - | - |
| 사용자 인증 | O | - | - |
| 리다이렉트 | O | - | - |
| 고정 응답 | O | - | - |

## 취득 후 혜택

시험을 치르고 나면 조금 뒤에 몇 가지 혜택을 안내하는 메일이 옵니다. 자세한 내용은 [AWS 인증 사이트](https://aws.amazon.com/kr/certification/benefits)에서 확인할 수 있지만, 간단히 적으면 대략 다음과 같습니다.
- AWS 인증 로고가 달린 상품을 구입할 수 있습니다.
- AWS 인증 모의고사를 구매할 때 쓸 수 있는 쿠폰을 받습니다.
- [LinkedIn](https://www.linkedin.com) AWS 커뮤니티에 참여할 수 있습니다.
- 다음 시험 응시료가 반값이 되는 쿠폰을 받습니다.
- 관련 이벤트 알림을 받을 수 있습니다.
- [Acclaim](https://www.youracclaim.com)에 등록할 수 있는 디지털 배지를 받습니다.

솔직히 디지털 배지나 굿즈 구매 권한은 아주 큰 메리트라고 느끼진 않았습니다. 그래도 다음 시험 할인 쿠폰과 모의고사 쿠폰은 꽤 실용적입니다. 저처럼 다음 자격도 이어서 볼 생각이 있다면 특히 그렇습니다. 물론 제 경우는 회사 지원을 받았기 때문에 체감은 조금 덜했습니다.
## 마지막으로

그래서 이 시험이 가치 있었느냐고 묻는다면, 제 대답은 그렇다는 쪽입니다. 암기에 약해서 준비 과정은 꽤 힘들었지만, 그 덕분에 AWS 서비스를 훨씬 입체적으로 이해하게 됐습니다. 다른 클라우드 벤더를 보더라도 큰 개념은 AWS와 통하는 부분이 많아서, Azure나 GCP를 다룰 때도 충분히 도움이 됩니다. 실제로 AWS를 공부하며 익힌 개념 덕분에 Oracle Cloud VM을 활용할 때도 훨씬 수월했습니다.

또 클라우드마다 무료 티어 구성이 다르지만, AWS는 Lambda와 DynamoDB를 무료로 활용할 수 있습니다. 그래서 이 자격을 준비하면서 익힌 지식만으로도 간단한 서버리스 애플리케이션은 충분히 만들어 볼 수 있겠다는 감각이 생겼습니다. Oracle Cloud나 GCP 무료 VM과 조합하면, 무료 플랜만으로도 꽤 괜찮은 서비스를 시작할 수 있겠다는 생각도 들었습니다. 이런 시야는 자격 준비를 하지 않았다면 쉽게 얻기 어려웠을 것 같습니다.

한 가지 고민되는 점은 AWS 자격의 유효기간이 3년이라는 점입니다. 클라우드 서비스는 계속 변하고 새 기능도 빠르게 나오니 이해는 가지만, 자격을 어디에 어떻게 활용할지는 미리 생각해 둘 필요가 있습니다. 또 AWS 자격끼리는 공통 범위가 적지 않아서, 하나를 딴 뒤에 다른 자격도 이어서 준비하는 편이 효율적이겠다는 생각이 들었습니다. 저 역시 언젠가는 솔루션 아키텍트 쪽에도 다시 도전해 보고 싶습니다.

다만 지금은 잠깐 쉬고 싶긴 합니다.
