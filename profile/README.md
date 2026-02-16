# ICE GitOps - 한국외대 정보통신공학과 프로젝트 배포 플랫폼

코드를 push하면 **자동으로 웹서비스가 배포**됩니다!

```
https://iceweb.hufs.ac.kr/{프로젝트명}/
```

---

## 🎯 어떻게 동작하나요?

```
코드 Push → 자동 빌드 → 자동 배포 → 웹서비스 오픈!
```

| 단계 | 설명 |
|------|------|
| 1️⃣ | Organization에서 레포 생성 & 코드 push |
| 2️⃣ | GitHub Actions가 자동으로 Docker 이미지 빌드 |
| 3️⃣ | ArgoCD가 자동으로 K8s에 배포 |
| 4️⃣ | `iceweb.hufs.ac.kr/{프로젝트명}/` 으로 접속! |

---

## 📋 시작하기

### 1. 레포 생성

1. 이 Organization에서 **New repository** 클릭
2. **Owner**: `ICE-GitOps` 선택 (⚠️ 개인 계정 아님!)
3. **Repository name**: 프로젝트명 (영문 소문자, 하이픈) → 예: `my-portfolio`
4. **Public** 선택
5. Create!

> 레포 이름 = URL 경로: `my-portfolio` → `iceweb.hufs.ac.kr/my-portfolio/`

### 2. Dockerfile 작성

프로젝트 루트에 `Dockerfile` 생성. 아래는 React 예시:

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ENV PUBLIC_URL=/my-portfolio
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

> ⚠️ **base path 설정 필수!** `PUBLIC_URL=/프로젝트명` 을 꼭 설정하세요.

<details>
<summary>📦 다른 프레임워크 Dockerfile 예시 (클릭)</summary>

#### Vue.js
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ENV BASE_URL=/my-portfolio/
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
> `vue.config.js`에 `publicPath: '/my-portfolio/'` 추가

#### Next.js
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```
> `next.config.js`에 `basePath: '/my-portfolio'` 추가

#### Spring Boot
```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./gradlew bootJar

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```
> `application.yml`에 `server.servlet.context-path: /my-portfolio` 추가

#### FastAPI
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080", "--root-path", "/my-portfolio"]
```

#### Express.js
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 8080
CMD ["node", "index.js"]
```
> `app.use('/my-portfolio', router)` 로 라우터 prefix 설정

</details>

### 3. CI 파이프라인 설정

아래 내용을 자신의 레포에 `.github/workflows/ci.yml` 파일로 생성합니다.

<details>
<summary>📄 ci.yml 전체 내용 (클릭하여 복사)</summary>

```yaml
name: CI - Build & Deploy

on:
  push:
    branches: [main]

env:
  PROJECT_NAME: my-portfolio  # ← 자신의 프로젝트명으로 변경!
  REGISTRY: ghcr.io
  GITOPS_REPO: ICE-GitOps/gitops

jobs:
  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/ice-gitops/${{ env.PROJECT_NAME }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  update-gitops:
    name: Update GitOps Manifests
    needs: build-and-push
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      - name: Setup Kustomize
        uses: imranismail/setup-kustomize@v2

      - name: Update image tag
        working-directory: gitops/projects/${{ env.PROJECT_NAME }}
        run: |
          kustomize edit set image \
            ghcr.io/ice-gitops/${{ env.PROJECT_NAME }}=ghcr.io/ice-gitops/${{ env.PROJECT_NAME }}:${{ github.sha }}

      - name: Commit and push
        working-directory: gitops
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "🚀 deploy(${{ env.PROJECT_NAME }}): update image to ${{ github.sha }}"
          git push
```

</details>

**⚠️ `PROJECT_NAME`을 반드시 자신의 프로젝트명으로 변경하세요!**

### 4. 관리자에게 알리기

위 1~3 완료 후 관리자에게 전달:

| 항목 | 예시 |
|------|------|
| 프로젝트명 | `my-portfolio` |
| 컨테이너 포트 | `80` (프론트) / `8080` (백엔드) |

관리자가 ArgoCD에 등록하면 **다음 push부터 자동 배포!** 🎉

---

## ❓ FAQ

| 질문 | 답변 |
|------|------|
| **CI 실패해요** | 1. `PROJECT_NAME`이 레포명과 일치하는지 확인 <br> 2. **코드 빌드 에러**일 확률 99%! <br> &nbsp;&nbsp; - 로컬에서 `localhost`나 하드코딩된 IP(`127.0.0.1`)를 쓰고 있는지 확인하세요. <br> &nbsp;&nbsp; - 로컬에서 `docker build`가 잘 되는지 먼저 트러블슈팅 필수 |
| 페이지 안 나와요 | base path 설정 했는지 확인 (Dockerfile 내 `PUBLIC_URL` 등) |
| CSS/JS 깨져요 | base path가 프로젝트명과 일치하는지 확인 |
| 환경변수/Secret | **GitHub Secret 설정 불필요!** (관리자가 다 해둠). <br> `PROJECT_NAME`만 잘 바꾸면 됩니다. |
| DB 필요해요 | 관리자에게 문의 |

---

## 📌 참고

- **리소스 제한**: CPU 2코어, 메모리 4Gi, Pod 5개 (프로젝트당)
- **문의**: 한국외대 정보통신공학과 21학번 고태규 (taekueko714@hufs.ac.kr)
