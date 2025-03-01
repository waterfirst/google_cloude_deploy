# Google Cloud에 Streamlit 앱 배포하기

이 가이드는 Streamlit 애플리케이션을 Google Cloud Run에 배포하는 전체 과정을 안내합니다. Docker 이미지 생성부터 인증 문제 해결까지 모든 단계를 포함합니다.

## 목차
1. [사전 준비](#사전-준비)
   - [Google Cloud 계정 설정](#google-cloud-계정-설정)
   - [필요한 소프트웨어 설치](#필요한-소프트웨어-설치)
2. [Google Cloud 프로젝트 설정](#google-cloud-프로젝트-설정)
   - [프로젝트 생성](#프로젝트-생성)
   - [서비스 계정 및 API 활성화](#서비스-계정-및-api-활성화)
   - [서비스 계정 키 다운로드](#서비스-계정-키-다운로드)
3. [Streamlit 앱 준비](#streamlit-앱-준비)
   - [프로젝트 구조](#프로젝트-구조)
   - [필요한 파일 생성](#필요한-파일-생성)
4. [Docker 이미지 생성](#docker-이미지-생성)
   - [Dockerfile 작성](#dockerfile-작성)
   - [이미지 빌드](#이미지-빌드)
5. [Google Cloud에 배포](#google-cloud에-배포)
   - [인증 설정](#인증-설정)
   - [이미지 푸시](#이미지-푸시)
   - [Cloud Run 서비스 배포](#cloud-run-서비스-배포)
6. [문제 해결](#문제-해결)
   - [인증 오류](#인증-오류)
   - [이미지 푸시 문제](#이미지-푸시-문제)
7. [마무리](#마무리)

## 사전 준비

### Google Cloud 계정 설정
1. [Google Cloud](https://cloud.google.com/) 웹사이트 방문
2. Google 계정으로 로그인 또는 새 계정 생성
3. "콘솔로 이동" 버튼을 클릭하여 Google Cloud Console 접속
4. 결제 정보 설정:
   - 왼쪽 메뉴에서 "결제" 선택
   - "결제 계정 연결" 또는 "결제 설정" 클릭
   - 신용카드 정보 입력 (무료 체험 크레딧 $300 제공)

### 필요한 소프트웨어 설치
1. **Docker Desktop**: 
   - [Docker 공식 웹사이트](https://www.docker.com/products/docker-desktop)에서 다운로드 및 설치
   - 설치 후 시스템 재시작 필요할 수 있음

2. **Google Cloud SDK**: 
   - [Google Cloud SDK 다운로드 페이지](https://cloud.google.com/sdk/docs/install)에서 운영체제에 맞는 버전 다운로드
   - 설치 프로그램 실행 및 지시에 따라 설치 완료

3. **Visual Studio Code**: 
   - [VS Code 다운로드 페이지](https://code.visualstudio.com/download)에서 다운로드 및 설치
   - 추천 확장 프로그램: Python, Docker, Remote - Containers

## Google Cloud 프로젝트 설정

### 프로젝트 생성
1. Google Cloud Console에서 상단의 프로젝트 선택 드롭다운 클릭
2. "새 프로젝트" 선택
3. 프로젝트 이름 입력 (예: "streamlit-app") 및 "만들기" 클릭
4. 생성된 프로젝트 선택

### 서비스 계정 및 API 활성화
1. 왼쪽 메뉴에서 "IAM 및 관리" > "서비스 계정" 선택
2. "서비스 계정 만들기" 클릭
3. 서비스 계정 이름 입력 (예: "streamlit-deployer")
4. "만들기 및 계속" 클릭
5. 다음 역할 부여:
   - Artifact Registry 관리자
   - Cloud Run 관리자
   - Storage 관리자
6. "완료" 클릭
7. 필요한 API 활성화:
   - 왼쪽 메뉴에서 "API 및 서비스" > "라이브러리" 선택
   - 검색창에서 "Cloud Run", "Artifact Registry" 검색 후 각각 활성화

### 서비스 계정 키 다운로드
1. "IAM 및 관리" > "서비스 계정" 메뉴로 이동
2. 앞서 생성한 서비스 계정 행에서 오른쪽 "작업" 메뉴(점 세 개) 클릭
3. "키 관리" 선택
4. "키 추가" > "새 키 만들기" 선택
5. 키 유형 "JSON" 선택 후 "만들기" 클릭
6. JSON 키 파일이 자동으로 다운로드됨 (예: `프로젝트ID-XXXXXXXX.json`)
7. 다운로드된 키 파일을 프로젝트 폴더로 이동 (주의: 이 파일을 안전하게 보관하고 절대 공개 저장소에 업로드하지 마세요)

## Streamlit 앱 준비

### 프로젝트 구조
프로젝트 디렉토리를 생성하고 다음과 같은 구조로 파일을 준비합니다:
```
streamlit-app/
├── app.py             # Streamlit 애플리케이션 코드
├── requirements.txt   # 필요한 Python 패키지 목록
├── Dockerfile         # Docker 이미지 빌드 설정
└── 프로젝트ID-XXXXXXXX.json  # 서비스 계정 키 파일
```

### 필요한 파일 생성
1. **app.py** - Streamlit 앱 코드 작성:
```python
import streamlit as st

st.title("마크다운 프레젠테이션 변환기")
st.write("시로 받은 보고서를 프레젠테이션으로 변환하세요")

# 파일 업로드 기능
uploaded_file = st.file_uploader("마크다운 파일을 업로드하세요", type=["md"])

if uploaded_file:
    content = uploaded_file.read().decode("utf-8")
    st.markdown("### 업로드된 내용")
    st.text(content)
    
    # 여기에 변환 로직 추가
    
    st.success("변환 완료!")
```

2. **requirements.txt** - 필요한 패키지 목록:
```
streamlit==1.24.0
```

3. **Dockerfile** - Docker 이미지 설정:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8501

CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

## Docker 이미지 생성

### Dockerfile 작성
앞서 생성한 Dockerfile을 사용합니다. 이 파일은 Python 3.9 기반 이미지를 사용하고, 필요한 패키지를 설치한 후, Streamlit 앱을 실행하는 명령을 포함합니다.

### 이미지 빌드
VS Code에서 터미널을 열고 다음 명령어를 실행하여 Docker 이미지를 빌드합니다:

```bash
# 프로젝트 디렉토리로 이동
cd streamlit-app

# Docker 이미지 빌드
docker build -t streamlit-app4 .

# 빌드된 이미지 확인
docker images | grep streamlit-app4
```

## Google Cloud에 배포

### 인증 설정
Google Cloud와 Docker를 연결하기 위한 인증 설정을 합니다:

1. 서비스 계정으로 로그인:
```bash
# 서비스 계정 키 파일로 인증
gcloud auth activate-service-account --key-file=프로젝트ID-XXXXXXXX.json
```

2. 프로젝트 설정:
```bash
# Google Cloud 프로젝트 ID 설정
gcloud config set project 프로젝트ID
```

3. Docker 인증 설정:
```bash
# Google Container Registry 인증 설정
gcloud auth configure-docker
```

4. 추가 인증 단계 (인증 문제가 발생할 경우):
```bash
# 액세스 토큰 받기
$env:TOKEN = $(gcloud auth print-access-token)

# 토큰으로 Docker 로그인
echo $env:TOKEN | docker login -u oauth2accesstoken --password-stdin https://gcr.io
```

### 이미지 푸시
빌드된 Docker 이미지를 Google Container Registry(GCR)에 푸시합니다:

1. 이미지에 GCR 태그 지정:
```bash
docker tag streamlit-app4 gcr.io/프로젝트ID/streamlit-app4
```

2. 이미지 푸시:
```bash
docker push gcr.io/프로젝트ID/streamlit-app4
```

### Cloud Run 서비스 배포
이제 Cloud Run에 서비스를 배포합니다:

```bash
gcloud run deploy streamlit-app4 \
  --image=gcr.io/프로젝트ID/streamlit-app4 \
  --region=asia-northeast3 \
  --platform=managed \
  --allow-unauthenticated
```

명령이 완료되면 Service URL이 제공됩니다 (예: `https://streamlit-app4-XXXXXXXX.asia-northeast3.run.app`).

## 문제 해결

### 인증 오류
GCR에 이미지를 푸시할 때 인증 오류가 발생할 경우:

1. **서비스 계정 권한 확인**:
   - IAM 및 관리 > IAM에서 서비스 계정에 필요한 권한이 있는지 확인
   - 필요한 권한: Artifact Registry 관리자, Storage 관리자

2. **서비스 계정 재인증**:
```bash
gcloud auth activate-service-account --key-file=프로젝트ID-XXXXXXXX.json
```

3. **직접 토큰으로 로그인**:
```bash
$env:TOKEN = $(gcloud auth print-access-token)
echo $env:TOKEN | docker login -u oauth2accesstoken --password-stdin https://gcr.io
```

### 이미지 푸시 문제
이미지 푸시가 계속 실패하는 경우:

1. **Docker Hub 대안 사용**:
```bash
# Docker Hub 로그인
docker login

# 이미지 태그 변경
docker tag streamlit-app4 사용자이름/streamlit-app4

# Docker Hub에 푸시
docker push 사용자이름/streamlit-app4

# Cloud Run에서 Docker Hub 이미지 사용
gcloud run deploy streamlit-app4 --image=docker.io/사용자이름/streamlit-app4 --region=asia-northeast3 --platform=managed
```

2. **Cloud Console에서 직접 배포**:
   - Google Cloud Console > Cloud Run > 서비스 만들기
   - "컨테이너 이미지 URL" 필드에 이미지 경로 입력
   - 필요한 설정 구성 후 "만들기" 클릭

## 마무리
이제 Streamlit 앱이 성공적으로 Google Cloud Run에 배포되었습니다! 제공된 URL을 통해 웹 브라우저에서 앱에 접근할 수 있습니다.

몇 가지 추가 팁:

1. **자동 배포 설정**: GitHub Actions 또는 Cloud Build를 사용하여 코드 변경 시 자동 배포 구성
2. **비용 모니터링**: Google Cloud Console의 결제 섹션에서 사용량 및 비용 모니터링
3. **로깅 및 모니터링**: Cloud Run 서비스 페이지에서 로그 확인 및 성능 모니터링

이 가이드를 따라하면 Streamlit 앱을 Google Cloud에 성공적으로 배포할 수 있습니다. 문제가 발생하면 Google Cloud 문서와 커뮤니티 포럼을 참고하세요.
