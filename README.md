# 🚀 ipiece-manifests-test — EKS GitOps 자동 배포 연습 환경 정리

## 🎯 0. 오늘 목표

**Spring 백엔드 + Next.js 프론트를 GitOps 기반으로 EKS에 자동 배포하는 전체 파이프라인 구성**

전체 흐름:

1. 코드 GitHub push
2. GitHub Actions → ECR 이미지 빌드 & 푸시
3. `ipiece-manifests-test` 레포에 이미지 태그 자동 반영
4. Argo CD가 이를 감지 → 자동 Sync
5. EKS(`ipiece-eks-dev`)에 즉시 롤링 배포
6. 외부에서 `api.ipiece.store`로 정상 접근

---

# 🟩 1. 백엔드 레포 준비 — `ipiece-backend-test`

## 1-1. Spring Boot 프로젝트

**레포:** `ipiece-backend-test`

**패키지:** `edu.ce.fisa`

```java
@SpringBootApplication
public class IpieceBackendTestApplication {
    public static void main(String[] args) {
        SpringApplication.run(IpieceBackendTestApplication.class, args);
    }
}

```

### 헬스체크 컨트롤러

```java
@RestController
public class HelloController {

    @GetMapping("/healthz")
    public String health() {
        return "ok ci/cd test";
    }

    @GetMapping("/hello")
    public String hello() {
        return "hello from ipiece-backend-test";
    }
}

```

---

# 🟩 2. 백엔드 Docker + CI/CD (ECR 푸시 + GitOps 반영)

## 2-1. Dockerfile

```docker
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app

COPY pom.xml .
COPY .mvn .mvn
COPY mvnw mvnw
RUN ./mvnw -q -B dependency:go-offline

COPY src src
RUN ./mvnw -q -B package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]

```

---

## 2-2. GitHub Actions (`backend-ci.yml`)

동작 요약:

- main/develop push 시
    - Maven 테스트
    - Docker build → ECR push
    - manifests repo의 `deployment.yaml`에 이미지 태그 자동 업데이트
    - Git push → ArgoCD 자동 Sync → EKS 롤링 배포

```yaml
name: Backend CI/CD (ECR, Maven)

on:
  push:
    branches: [ main, develop ]
  workflow_dispatch:

env:
  AWS_REGION: ap-northeast-2
  ECR_REPOSITORY: ipiece-backend-test

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - name: Cache Maven repository
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Run tests
        run: mvn -B test

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Update manifests repo with new image tag
        env:
          GIT_TOKEN: ${{ secrets.MANIFESTS_REPO_TOKEN }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          git config --global user.email "ci@ipiece.local"
          git config --global user.name "ipiece-ci"

          git clone https://x-access-token:${GIT_TOKEN}@github.com/kohtaewoo/ipiece-manifests-test.git
          cd ipiece-manifests-test/backend

          sed -i "s#image: 235625001959.dkr.ecr.ap-northeast-2.amazonaws.com/ipiece-backend-test:.*#image: 235625001959.dkr.ecr.ap-northeast-2.amazonaws.com/ipiece-backend-test:${IMAGE_TAG}#" deployment.yaml

          if git diff --quiet; then
            echo "No changes to commit in manifests repo."
          else
            git commit -am "chore: update backend image to ${IMAGE_TAG}"
            git push
          fi

```

필수 Secrets:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `MANIFESTS_REPO_TOKEN` (GitHub PAT)

---

# 🟦 3. 매니페스트 레포 (`ipiece-manifests-test`)

## 디렉토리 구조

```
ipiece-manifests-test/
├─ backend/
│  ├─ namespace.yaml
│  ├─ deployment.yaml
│  └─ service.yaml
└─ argocd/
   └─ app-backend.yaml

```

---

## 3-2. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ipiece-backend-test

```

---

## 3-3. Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ipiece-backend
  namespace: ipiece-backend-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ipiece-backend
  template:
    metadata:
      labels:
        app: ipiece-backend
    spec:
      containers:
        - name: ipiece-backend
          image: 235625001959.dkr.ecr.ap-northeast-2.amazonaws.com/ipiece-backend-test:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080

```

---

## 3-4. Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ipiece-backend
  namespace: ipiece-backend-test
spec:
  type: LoadBalancer
  selector:
    app: ipiece-backend
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP

```

---

## 3-5. Argo CD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ipiece-backend-test
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/kohtaewoo/ipiece-manifests-test.git'
    targetRevision: main
    path: backend
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: ipiece-backend-test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

```

적용:

```bash
kubectl apply -f argocd/app-backend.yaml

```

---

# 🟨 4. EKS 클러스터 (`ipiece-eks-dev`)

- 리전: **ap-northeast-2**
- Kubernetes: **1.34**
- EKS 자율 모드 + EC2 Node Group 병행
- 서브넷 태깅 필수:

### 퍼블릭 서브넷

```
kubernetes.io/role/elb = 1
kubernetes.io/cluster/ipiece-eks-dev = shared

```

### 프라이빗 서브넷

```
kubernetes.io/role/internal-elb = 1
kubernetes.io/cluster/ipiece-eks-dev = shared

```

---

# 🟦 5. 배포 + ELB + DNS 연결

ELB 생성 후:

```bash
curl http://<ELB-DNS>/healthz

```

DNS CNAME:

```
api → <ELB-DNS>

```

접속:

```
http://api.ipiece.store/healthz

```

---

# 🟧 6. 프론트엔드 GitOps 파이프라인 (진행 중)

## Next.js Dockerfile

```docker
FROM node:20-alpine AS builder
WORKDIR /app

RUN corepack enable

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

FROM node:20-alpine
WORKDIR /app

RUN corepack enable
ENV NODE_ENV=production

COPY --from=builder /app ./

EXPOSE 3000
CMD ["pnpm", "start"]

```

## CI/CD (`frontend-ci.yml`)

GitHub Actions로 빌드 → ECR 푸시 → manifests repo 이미지 태그 변경

(백엔드와 동일 구조)

프론트는 현재 **DialogContent showCloseButton 타입 에러만 해결하면 바로 완성됨.**

---

# 🟩 7. 오늘 결과 요약

### ✔ 백엔드 GitOps 전체 파이프라인 **완전 구축 완료**

- Spring 코드 → GitHub push
- GitHub Actions → ECR → manifests 이미지 태그 업데이트
- ArgoCD 자동 Sync
- EKS 롤링 배포
- DNS(API)까지 완전 연결됨

### ✔ 프론트엔드 GitOps

- 파이프라인 구조 완성
- TypeScript 에러만 해결하면 백엔드와 동일하게 자동 배포 완료됨
