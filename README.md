<img alt="automatic-roadline" src="https://user-images.githubusercontent.com/63290629/155466560-3552fc08-85c6-4f6d-b518-acdd6a3c8c00.png">

<br>

# Automatic-Lane-Guidance System
> ‘컴퓨터비전’ 수업 #2 과제 → **Automatic-Lane-Guidance System**


<Br>


### ✅ 프로젝트 소개  
<br>

- 운전자가 주행할 때 필요한 차선 정보 검출: 2~3초간 주행할 거리만큼의 차선 정보. **차선 검출 기능**
- 현재 이동하고 있는 차선을 이탈한 경우, WARNING → **차선 이탈 감지 기능**

<Br>

### 📝 요구 사항 분석

<br>

- Hough Transform을 이용해서 도로에 있는 선을 검출  
    - 그레이스케일 영상으로 변환
    - 캐니 엣지 검출
    - line 검출
    - 선분 연장해서 차선 그리기  


- 차선 이탈 감지  
    - 검출한 차선을 이탈한 경우, 붉은 색 warning 표시


<br>

### 💻 구현 [#1 차선 검출]  

<br>

🚨 **주의**: 흰색/노란색 차선 뿐만 아니라 **파란색** 차선도 존재함

    try 01) 히스토그램 평활화 후, 엣지 검출
            → 파란색 차선 뚜렷해지지만, 도로의 마모된 부분도 함께 뚜렷해짐

    try 02)  HSV로 변환하고, 3개의 채널로 분리해서 파란색 검출을 위한 H 범위 조절 (40-130)
            → 파란색 차선이 엣지로 제대로 검출되지 않음

    try 03) HSV로 변환하고, 3개의 채널로 분리해서 파란색 검출을 위한 H 범위 조절 (40-130)
            + Edge detection을 위한 S, V 범위 또한 조절
            → 도로의 마모된 부분(검은색)이 필터링되어 엣지로 검출되지 않음

<img alt="lane01" src="https://user-images.githubusercontent.com/63290629/155466744-3b2875d6-a693-4a34-95d0-957d92b187da.png">

<br>

**1. 1차 필터링: 컬러 범위를 통해 후보 영역 필터링**

→ `filterByColor(frame, ROI_colors)`

<br>

**2. RGB 영상을 그레이스케일 영상으로 변환**

→ `cvtColor(ROI_colors, gray, COLOR_BGR2GRAY)`

<Br>

**3. 가우시안 블러로 잡음 제거 후 캐니 에지 검출**

→ `Canny(gray, edges, 100, 200)`

<br>

**4. ROI 지정해서 차선을 검출할 영역 제한**

> 차가 이동할 도로를 중심으로 선을 검출할 수 있도록 ROI 영역을 지정해주었다. 
이 범위 이외의 도로의 정보는 필요없다.

→ `edges = setROI(edges, points)`

<Br>

**5. HoughLineP API를 통한 edge에서 line 검출**

→ `HoughLineP(edges, lines, rho, theta, hough_threshold, minLength, maxLineGap)`

- 앞에서 달리는 차, 지평선, 옆 차선 등에서 각종 line이 함께 검출  
    기울기를 통해 현재 차선 검출하기(2차 필터링)

<Br>

**6. 2차 필터링: 기울기 범위를 통해 현재 차선만 검출**

→ `detectionLane(mark, detected, lines)`

     기울기 범위: slope_min_threshold 이상, slope_max_threshold 이하

<br>

<aside>

**💡 오른쪽 차선과 왼쪽 차선 구분 기준**  

1. 오른쪽 차선의 기울기 > 0, 왼쪽 차선의 기울기 < 0
2. 중앙의 x 좌표 기준

</aside>

<Br>

**7. 원본 영상에 검출한 차선 병합**

→ `addWeighted(frame, 1, detected, 0.2, 0.0, result)`

<img alt="lane08" src="https://user-images.githubusercontent.com/63290629/155466774-0cade259-04dc-4f9e-b7e5-03bd8ae2ace9.png">


<Br>

### 💻 구현 [#2 차선 이탈 감지 기능]

> **WARNING** 상태 정의

- 차선 이탈의 경우, 차선이 프레임의 중심에 들어오게 된다.
- 프레임의 중심에 있는 차선은 기울기가 거의 수직에 가깝다.
- → 기울기가 큰 Line이 프레임 중심에서 검출되면 WARNING!


`bool warning = false` 변수 추가해서 warning 조건 검사

- 기울기 > 2.0 인가?
- line 끝점의 x좌표가 영상의 중앙에 위치하는가?
- → 조건 만족시, warninig = true

1. 오른쪽과 왼쪽의 대표 Line 검출 → `detectionLane(mark, detected, lines)`
2. 원본 영상에 검출한 차선 병합     → `addWeighted(frame, 1, detected, 0.2, 0.0, result)`


<img alt="lane10" src="https://user-images.githubusercontent.com/63290629/155466777-1450c906-9a1f-43e0-80f8-9209d9bddab4.png">


---

<br>

### ✅ 시행 착오

> #1. 각 도로의 차선 범위의 차이, 자동차 높이에 따른 영상 범위의 차이가 존재
> 
- 넓은 차선 범위를 포함하기 위해 ROI의 너비를 넓힘
- *더 많은 엣지 검출(불필요한)*
- → *2차 필터링 진행*
    
<Br>

> #2. 중심 차선(오른쪽, 왼쪽) 검출에서 여러개의 선 검출/검출된 선의 흔들림이 큼
> 
- 여러 개의 선이 검출된 경우, 평균값 계산해서 하나로 취급
- 최근 검출한 10개의 선을 저장하는 save_lines 벡터 선언
- 최근 누적된 10개의 선의 평균값을 이용해 대표 line 지정
- **→ 차선 이동 시, 검출 line 부드러운 이동**


<Br>

### 🙋‍♀️ 역할

- 1차 필터링
    - canny edge 검출
    - ROI 영역 설정
    - 문제 해결: 파란색 검출 실패
        - 색상 변환(RGB → HSV, 범위 조절) → 파란색 검출 성공
- 2차 필터링
    - threshold 값 조정
- 차선 이탈 감지


<Br>


### 💦 아쉬운 점

- HSV로 검은색을 필터링할 수 있으나, 푸른빛의 검은색은 필터링 X
- 여러줄이 검출되어 line들의 평균을 대표 line으로 지정했는데, 평균이므로 outlier에 민감
- 색상 필터링이므로 조명에 많은 영향 받음


### [↪️ 데모 영상](https://github.com/donnyrla10/Automatic-Line-Guidance-System/tree/master/TeamProject%232/Demo)
