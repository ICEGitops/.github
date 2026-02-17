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
2. **Owner**: `ICEGitops` 선택 (⚠️ 개인 계정 아님!)
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
<summary>📂 프론트엔드 + 백엔드 같이 있는 경우 (Dockerfile 위치)</summary>

레포지토리 하나에 `frontend/`와 `backend/` 폴더가 따로 있다면, **각 폴더 안에 `Dockerfile`을 각각 만들어야 합니다.**

```text
my-repo/
├── frontend/
│   ├── Dockerfile  (프론트용)
│   ├── package.json
│   └── ...
├── backend/
│   ├── Dockerfile  (백엔드용)
│   ├── build.gradle
│   └── ...
└── .github/
    └── workflows/
        └── ci.yml
```
</details>

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

**⚠️ [필독] 풀스택(Front+Back) 사용자의 경우 Base Path 설정**
프론트엔드와 백엔드를 모두 배포하면 **서로 다른 주소**를 사용하게 됩니다.
Dockerfile 작성 시 **Base Path(Context Path)를 각각 다르게 설정**해야 합니다!

| 구분 | 주소 (URL) | Dockerfile 설정 예시 (Base Path) |
|---|---|---|
| **Frontend** | `.../my-team/` | `PUBLIC_URL=/my-team` |
| **Backend** | `.../my-team-back/` | `ContextPath=/my-team-back` |

> **주의**: 백엔드 API를 호출할 때도 `/my-team-back`을 prefix로 붙여서 호출해야 합니다.


### 3. CI 파이프라인 설정

아래 내용을 자신의 레포에 `.github/workflows/ci.yml` 파일로 생성합니다.

<details>
<summary>📦 프론트엔드 또는 백엔드 단독 배포 시 (기본형)</summary>

레포지토리에 `Dockerfile`이 하나만 있다면, 아래 내용을 `.github/workflows/ci.yml` 파일로 저장하세요.

```yaml
name: CI - Basic Build

on:
  push:
    branches: [main]

env:
  PROJECT_NAME: my-portfolio  # ← 자신의 프로젝트명으로 변경!
  REGISTRY: ghcr.io
  GITOPS_REPO: ICEGitops/gitops

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
          images: ghcr.io/icegitops/${{ env.PROJECT_NAME }}
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
            ghcr.io/icegitops/${{ env.PROJECT_NAME }}=ghcr.io/icegitops/${{ env.PROJECT_NAME }}:${{ github.sha }}

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

<details>
<summary>📦 프론트엔드 + 백엔드 같이 배포할 때 CI 설정</summary>

만약 한 레포지토리에 `frontend/`와 `backend/` 폴더가 따로 있다면, 아래처럼 `ci.yml`을 설정해야 합니다.

```yaml
name: CI - Monorepo Build

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  GITOPS_REPO: ICEGitops/gitops
  # 👇 팀 이름으로 수정
  FRONT_IMAGE: ghcr.io/icegitops/my-team-front
  BACK_IMAGE: ghcr.io/icegitops/my-team-back

jobs:
  build-front:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: ./frontend  # 👈 프론트엔드 폴더 지정
          push: true
          tags: ${{ env.FRONT_IMAGE }}:${{ github.sha }},${{ env.FRONT_IMAGE }}:latest

  build-back:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: ./backend  # 👈 백엔드 폴더 지정
          push: true
          tags: ${{ env.BACK_IMAGE }}:${{ github.sha }},${{ env.BACK_IMAGE }}:latest

  update-gitops:
    needs: [build-front, build-back]
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops
      - uses: imranismail/setup-kustomize@v2
      
      # 프론트엔드 이미지 태그 업데이트
      - run: |
          cd gitops/projects/my-team  # 👈 프론트 프로젝트명 확인
          kustomize edit set image ${{ env.FRONT_IMAGE }}=${{ env.FRONT_IMAGE }}:${{ github.sha }}
      
      # 백엔드 이미지 태그 업데이트
      - run: |
          cd gitops/projects/my-team-back  # 👈 백엔드 프로젝트명 확인
          kustomize edit set image ${{ env.BACK_IMAGE }}=${{ env.BACK_IMAGE }}:${{ github.sha }}
          
      - run: |
          cd gitops
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git commit -m "🚀 deploy(monorepo): update images to ${{ github.sha }}"
          git push
```
</details>

</details>

</details>

</details>

</details>

<br>

---

### ⚠️ CI 설정 파일 수정 가이드 (반드시 확인하세요!)

#### 1. [기본형] 프론트엔드 또는 백엔드 단독 배포 시

*   **`env` > `PROJECT_NAME`**
    *   변경 전: `my-portfolio`
    *   변경 후: **본인의 프로젝트 이름** (예: `my-portfolio`)

<br>

#### 2. [풀스택] 프론트엔드 + 백엔드 같이 배포 시

*   **`env` 섹션 (이미지 이름)**
    *   `FRONT_IMAGE`: `[팀이름]-front` 로 변경
    *   `BACK_IMAGE`: `[팀이름]-back` 로 변경
    *   *예시: `ghcr.io/icegitops/my-team-front`*

*   **`jobs` > `steps` > `context` (폴더 위치)**
    *   소스 코드가 있는 폴더명(`frontend`, `backend`)과 일치하는지 확인하세요.

*   **`update-gitops` > `run` (GitOps 경로)**
    *   `cd gitops/projects/my-team` 에서 **`my-team` 부분만** 프로젝트명으로 변경하세요.
    *   ⚠️ `gitops/projects/` 경로는 지우지 마세요!

---

<br>

### 4. 배포 요청 (관리자에게 이메일 발송)

1~3단계(CI 빌드 성공 확인)를 모두 마쳤다면, 아래 양식에 맞춰 관리자에게 메일을 보내주세요.
**이 메일을 보내야 실제 서버에 배포(CD)가 설정됩니다.**

- **수신인**: `taekueko714@hufs.ac.kr` (관리자 메일 주소)
- **제목**: `[ICE GitOps] 배포 요청 - {팀이름}`

**[메일 본문 양식 (복사해서 사용)]**
```text
1. 팀 이름 (URL로 사용됨): 
   (예: my-team) -> https://iceweb.hufs.ac.kr/my-team/

2. 프로젝트명: 
   (예: 학생회 소개 웹사이트)

3. 구성 아키텍처: 
   ( ) 프론트엔드 단독
   ( ) 프론트엔드 + 백엔드
   ( ) 백엔드 단독

4. 사용하는 컨테이너 포트:
   - 프론트엔드: (예: 80)
   - 백엔드: (예: 8080)

5. 데이터베이스 사용 여부:
   ( ) 사용 안 함
   ( ) 외부 DB 사용 (AWS RDS 등) -> 백엔드와 연결을 위해 6번 항목에 접속 정보(Host, ID, PW) 기재 필수
   
   ⚠️ **[내부 DB 요청 관련 유의사항]**
   학과 클러스터는 점검/장애 등으로 인해 **예고 없이 재시작될 수 있으며, 이 경우 데이터가 손실될 수 있습니다.**
   중요한 데이터는 반드시 **외부 DB (AWS RDS, PlanetScale 등)**를 사용하시기 바랍니다.
   내부 DB는 단순 캐싱(Redis)이나 데이터 유실이 상관없는 테스트 용도로만 신청해 주세요.
   
   ( ) 내부 DB 요청 (Redis) -> 위 데이터 손실 위험을 인지하였으며 동의함
   ( ) 내부 DB 요청 (MySQL) -> 위 데이터 손실 위험을 인지하였으며 동의함

6. 추가 요구사항 (외부 DB 사용 시 필수 기재):
   - DB_HOST (주소): 
   - DB_USER (아이디): 
   - DB_PASS (비밀번호): 
   - 기타 요청사항: 
```

관리자가 확인 후 **ArgoCD에 등록**하면, `https://iceweb.hufs.ac.kr/{팀이름}/` 주소가 활성화됩니다! 🎉

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
- **문의**: 정보통신공학과 21학번 고태규 (taekueko714@hufs.ac.kr)
