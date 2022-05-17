---
layout: post
title:  "[HGT] FTT Price Model"
date:   2022-05-17 15:00:00 +0900
categories: hgt
---

![ss](/assets/images/pricemodelss.png)

FTT Price Model is web service which provides price prediction information of [FTT(from FTX)](https://ftx.com/ftt).

[HGT](https://talenthandongsite.github.io/)'s FTT model 

## Background


HGT의 FTT모델은 FTX 거래소가 FTT를 소각하기 위해 사용된 USD와 FTT의 장기적인 가격이 상관관계가 있다는 가정하에 주별로 FTT소각에 사용된 USD 데이터와 해당되는 주의 FTT 평균 거래 가격 데이터를 가져와 다항 회기 분석을 통해 모델을 만들고 이를 API화 하여 웹 페이지로 제공한다.

<br>
<br>
<br>

## 배경

![execl](/assets/images/ftt-model-excel.png)

<br>

### 비즈니스적인 배경

- FTT 프라이스 모델 엑셀의 특성: 매주 매뉴얼하게 데이터를 추가하고 관리해 줘야함, 하지만 귀찮기 때문에 자동화하여 웹 페이지로 제공하면 좋음
- 현재로써는 데이터 모수가 적어 모델이 큰 의미가 없지만 개발 및 유지의 난이도가 낮아 쉽게 개발하고 장기적으로 데이터를 쌓아 나갈 수 있어 추후 역할이 기대됨

<br>

### 개발적인 배경

- Node.js를 개발언어로 선택 후 서비스의 배포 시간 및 사용되는 자원이 서비스 사이즈에 비해 크게 소모됨 (최종 도커 이미지 기준 약 140mb정도 크기)
- Jenkins를 배포 파이프라인으로 선택 후 서버 컴퓨터의 성능이 크게 요구 됨 (AWS기준 t2.medium정도 되어야 안정적으로 서비스 가능)

<br>
<br>
<br>

## 아키텍처 설계 및 기획
![arch](/assets/images/ftt-model-arch.png)

<br>

### 컴포넌트

- 클라이언트: 웹 사이트에 접속했을 때 보이는 화면, 프레임워크 및 라이브러리 없이 기본 HTML/CSS/JS로 구현
- 서버: 데이터를 주기적으로 받아오고 정제, 및 저장 하는 로직을 담당. Go로 구성
- 몽고 아틀라스: 데이터베이스 엔진, BSON형태로 데이터를 저장함, 무료임
- 외부 서비스
    - FTX: FTT의 소각 데이터(사용된 USD, 일시)를 받아오는 소스
    - Coin Market Cap: FTT의 가격 데이터를 받아오는 소스

<br>  

### 프로세스

- 주기적으로 데이터를 가져와 정제하기(보라색)  
    1. 매주 1회 (목요일 새벽, 보통 수요일에 FTT가 소각데이터를 추가함) 서버에서 스케쥴링을 통해 프로세스 시작  
    2. FTX에서 USD소각량 데이터, CMC에서 FTT의 가격 데이터를 HTTP요청을 통해 가져옴  
    3. 가져온 데이터를 데이터베이스의 데이터 형식에 맞춰 변환, 현재까지의 데이터들을 모두 가져와 회귀 분석하여 모델을 만듦  
    4. 데이터베이스에 해당 데이터를 모두 저장함  
- 유저가 브라우저를 통해 FTT모델 페이지를 열람(노란색)  
    1. 유저가 브라우저를 통해 FTT모델 페이지에 접속함, JS 스크립트에서 FTT에 대한 정보를 서버에 요청함  
    2. 해당되는 데이터를 데이터베이스에서 가져옴  
    3. 브라우저는 데이터를 받아서 뷰를 유저에게 보여줌  

<br>
<br>
<br>

## 개발 과정

아키텍처 설계 후 개발을 하는 과정은 다음과 같이 이루어졌다
1. 데이터 소스 찾기
2. 회기 분석 모델 만들기
3. Go 언어 스터디 및 서버 구현
4. 프론트엔드 뷰 구성
5. Github Action을 사용하여 서버에 디플로이

<br> 

### 데이터 소스를 찾기

![raw-data](/assets/images/ftt-raw-data.png)  

가장 먼저 시도한 것은 기존 엑셀의 수식을 분석하여 어떤 정보들을 토대로 모델을 만드는지 파악하는 것이었다. 다음의 정보를 얻어와야 모델을 만들 수 있었다
- 각 소각이 일어난 시간
- 해당 소각에 사용된 USD 예상치
- 해당 시간의 FTT가격  

위 3가지 데이터를 이용하여 누적 소각 금액 등의 여타 데이터들을 만들 수 있기 때문에 3개의 데이터는 꼭 외부 어디에선가 가져와야만 했다

<br>

![ftx-ftt](/assets/images/ftx-ftt.png)

FTX 웹사이트에서는 매주 소각량에 대한 데이터가 제공된다. 이 부분의 경우 웹사이트의 네트워크 트래픽을 살펴보면 Ftx서버에 /ftt-burns라는 API를 호출하여 받아온다는 것을 찾을 수 있었다. 다행히도 FTX서버에서 해당 엔드포인트의 CORS(Cross Origin Resource Sharing - 주로 서버에서 보안적인 이유로 Origin이 다른 사이트로부터의 API요청을 막는 역할을 함)를 막아놓지 않아 우리 서버에서 해당 API를 자유롭게 호출 할 수 있었다.

<br>

![cmc-quote](/assets/images/cmc-quote.png)

Coin Market Cap에서는 공식적으로 코인에 대한 가격 및 메타 정보를 가져올 수 있는 API를 제공한다. 한 달마다 무료로 33번 API요청을 할 수 있는데 FTT모델의 경우 일주일에 1번을 요청하므로 무료로 충분히 이용할 수 있다.

<br>

![mongo-quote](/assets/images/mongo-quote.png)

해당 데이터를 받아와서 필요한 부분을 정제하여 데이터 베이스에 저장할 수 있었다. 2주간 실험해 본 결과 FTX에서는 매주 수요일경 FTT 소각 데이터를 업로드하는 것을 파악할 수 있었다. 약간의 오차가 있어도 데이터가 수집 되도록 목요일 새벽 마다 FTX와 코인 마켓캡에 API요청을 하여 데이터를 받아와 데이터베이스에 저장하는 스케쥴링을 만들었다.

<br>

### 회기 분석 모델 만들기

엑셀에서 보이는 모델의 경우 엑셀의 자체 기능으로 계산된 것이다. 윈도우 버전의 엑셀에서는 [Forcast Sheet](https://support.microsoft.com/en-us/office/create-a-forecast-in-excel-for-windows-22c500da-6da7-45e5-bfdc-60a7062329fd)라는 기능을 제공한다. 이 기능은 엑셀에 인풋과 아웃풋을 넣으면 해당 데이터를 토대로 다항 회기 분석을 해주는 기능이다. 이 기능은 기존의 엑셀 파일에서 데이터를 넣은 뒤 차트를 그릴 때 사용되었다.

문제는 엑셀 내부의 기능이기 때문에 외부에서 사용할 수 없다는 점이었다. 따라서 엑셀의 forcast sheet와 동일할 역할을 하는, 즉 다항 회기 분석을 할 수 있는 툴을 만들어야 했다. 다음 2개의 레퍼런스를 통해 다항 회기 분석을 하는 법에 대해 학습하였다.
- [다항 회기 분석](https://www.youtube.com/watch?v=nGcMl03LPC0)
- [다항 회기 분석을 할 때 차수를 결정하는 방법](https://youtu.be/-BR4WElPIXg)

다항 회기 분석을 하는 함수는 해당 프로젝트와는 별도로 구현하였으며 다른 곳에서도 재 사용될 수 있도록 [pkg.go.dev](https://pkg.go.dev)(Go 언어에서 사용할 수 있는 패키지를 다운로드 할 수 있는 사이트)에 업로드 하였다.
- [pregression](https://pkg.go.dev/github.com/cocm1324/pregression)


![modeled](/assets/images/modeled.png)  
완성된 다항 회기 분석 함수를 통해 모델을 생성한다. 모델을 생성하는 주기는 새로운  데이터베이스에 저장한 모습이다. FTX와 코인 마켓캡에서 데이터를 가져오는 목요일 새벽마다 새롭게 모델을 만들어 데이터베이스에 저장하도록 구현하였다.

<br>

### Go 언어 스터디 및 서버 구현 

기존의 Node.js환경에서 개발을 하는 것은 여러 이점(프론트와 백엔드 언어 통일, 패키지 많음)이 있었다. 하지만 V8엔진 위에 스크립트가 돌아가는 Node.js의 특성상 리소스가 비교적 많이 소모되고 성능에 손해가 있다. 개발 프로젝트 상의 용의성을 위해 TypeScript로 코드를 작성하지만 이는 빌드 타임을 길게 만든다.

Go 언어를 선택한 이유는 가볍고 비교적 높은 성능의 언어이며 빌드타임, 자원 소모에서 Node.js에 비해 우위라고 판단했기 때문이다. 실제로 Node.js 프로젝트가 끝나고 서버에 배포시 이미지 사이즈가 140mb정도임에 비해 Go로 작성한 이미지의 경우 2mb정도로 큰 차이를 보여주었다. 

(구현에 대한 설명 생략)
[(코드 보기)](https://github.com/talenthandongsite/server-api)

참고할 만한 자료
- [A Tour of Go](https://go.dev/tour/welcome/1)(Go 언어 기본 튜토리얼)
- [Traversy Media - Golang REST API With Mux](https://www.youtube.com/watch?v=SonwZ6MF5BE&t=52s)(REST API 서버 구성)
- [Go Standard Project Layout](https://github.com/golang-standards/project-layout)(Go의 표준 프로젝트 레이아웃)

<br>

### FrontEnd 뷰 구성

(설명 생략)  
[(코드 보기)](https://github.com/talenthandongsite/server-api/tree/master/web)

![plotly](/assets/images/plotlyjs.png)  
차트를 구현할 때 [Plotly.js](https://plotly.com/javascript/)라는 라이브러리를 사용함

<br>

### Github Action의 구성

기존에는 Jenkins를 통해 서버 배포를 진행하였다. Jenkins는 서버에 설치되어 돌아가는 CICD프로그램인데 기존에 사용했던 기능은 다음과 같다
- 소스코드의 변화를 감지
- 소스코드가 변화 할 시 정해진 파이프라인에 따라 빌드
- 서버에 배포
- Secret관리(DB커넥션, 비밀번호, 비밀키, API키 등)   

Jenkins의 경우 용량이 큰 프로그램이기 때문에 AWS기준 t2.medium 미만의 인스턴스에서 불안정함을 경험한 적이 있다. 따라서 Jenkins를 사용할 때는 t2.medium인스턴스를 사용하고 있었다. Github Action으로 변경하는 이유는 젠킨스에 비해 기능이 적지만 서버의 부담이 거의 없는 구조(github에서 runner가 직접 배포를 하는 구조)이기 때문에 불필요하게 큰 인스턴스를 쓸 필요가 없기 때문이다.  

![action](/assets/images/github-action.png)

기존 Jenkins에서 실행되던 기능을 Github Action으로 옮겨와 구성하였다. 

Github Action으로의 변경 후 인스턴스를 t2.small로 조정하여 서버 비용을 평균 월 50000KRW에서 월 30000KRW정도로 낮출 수 있었다.

하지만 Github Action의 경우 github안의 runner가 실행하는 것이므로 talent서버에 github runner가 접근할 수 있도록 방화벽을 내려야 했다.


<br>
<br>
<br>

## 결론 

### 성과
- [서비스 구현](http://talent-handong.site/app/fttModel)
- 데이터를 쌓는 파이프라인이 생김
- 서비스의 자원 효율 증가 (서비스 소모 메모리 140mb -> 2mb)
- 서버 비용 감소 (t2.medium -> t2.small, Approx. KRW50000/month -> KRW30000/month)

<br>

### 한계
- 데이터 모수의 부족으로 인한 낮은 활용성(현재 150, 연간 50건씩 쌓일 예정)
- Github Action 파이프라인의 문제(방화벽)