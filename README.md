## 🖥️ AI 서비스 상용화 프로젝트

#### 대상 프로젝트

> 청년들의 문화생활 플랫폼 '틴플레이(Teenplay)'  
> Git-hub: https://github.com/hyuncoding/teenplay_server  
> Website: http://teenplay.site

#### AI 서비스

> 회원별 활동 추천 서비스

### 📌 목차

1. 데이터 수집
   1) 모임 데이터
   2) 활동 데이터
2. 데이터 탐색 및 전처리
3. 모델 학습
4. 모델 평가
5. Django 프로젝트 상용화 화면
6. 트러블슈팅 및 느낀 점

---

### 1. 데이터 수집

##### 1) 모임 데이터

-   데이터 수집 사이트: https://m.blog.naver.com/so_moim?tab=1
-   `selenium` 라이브러리를 통한 크롤링으로 데이터를 수집하였습니다.
-   크롤링 목적은 '활동' 데이터 수집에 앞서, 각 활동을 개설한 '모임' 데이터 수집을 위한 모임 이름 더미 데이터 수집입니다.
-   <details>
    <summary>Click to see full code</summary>

          from selenium import webdriver
          from selenium.webdriver.common.by import By
          from selenium.webdriver.chrome.service import Service
          from selenium.webdriver.chrome.options import Options
          from webdriver_manager.chrome import ChromeDriverManager
          import csv
          import time
          from tqdm import tqdm

          with open('scraped_data2.csv', 'w', newline='', encoding='utf-8-sig') as file:
              writer = csv.writer(file, quoting=csv.QUOTE_NONE)  # CSV 파일에 쓰기 위한 writer 객체 생성
              writer.writerow(['club_title'])  # CSV 헤더 작성
              # ChromeDriver 설정
              service = Service(ChromeDriverManager().install())
              options = Options()
              options.add_argument('--headless')  # 브라우저를 백그라운드에서 실행
              driver = webdriver.Chrome(service=service, options=options)

              # 웹사이트 열기
              driver.get("https://m.blog.naver.com/so_moim?tab=1")

              # 잠시 대기
              driver.implicitly_wait(10)

              # 페이지 끝까지 스크롤 다운하여 콘텐츠 로드
              SCROLL_PAUSE_TIME = 2

              last_height = driver.execute_script("return document.body.scrollHeight")

              title_set = set()

              for i in tqdm(range(50)):
                  # 페이지 끝까지 스크롤 다운
                  driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")

                  # 새로운 콘텐츠가 로드될 때까지 대기
                  time.sleep(SCROLL_PAUSE_TIME)

                  # 새로운 스크롤 높이 계산
                  new_height = driver.execute_script("return document.body.scrollHeight")

                  # 더 이상 새로운 콘텐츠가 없으면 루프 종료
                  if new_height == last_height:
                      break

                  last_height = new_height

              # 제목 텍스트들 찾기
              titles = driver.find_elements(By.CSS_SELECTOR, ".title__UUn4H span")

              # 제목 텍스트 전처리 후 writer로 파일에 출력
              for title in tqdm(titles):
                  title = title.text
                  if "<" in title and ">" in title:
                      title = title.split("<")[-1]
                      if title[-1] != ">":
                          target_idx = title.index(">")
                          title = title[:target_idx]
                      else:
                          title = title[:-1]
                  else:
                      continue
                  title = title.replace('"', "")
                  title = title.replace(',', '')
                  if title not in title_set:
                      writer.writerow([title])
                      title_set.add(title)

              # 브라우저 종료
              driver.quit()

</details>

-   크롤링을 통해 추출한 데이터를 csv로 내보낸 결과는 아래와 같습니다.

- <details>
    <summary>Click to see full code</summary>

        import pandas as pd

        c_df = pd.read_csv('./datasets/scraped_data2.csv')
        c_df

</details>

![club_original_data_csv](https://github.com/hyuncoding/django_with_ai/assets/134760674/6c3180f0-8b5d-4908-bab5-d793995683eb)

##### 2) 활동 데이터

- 데이터 수집 사이트
    - https://www.ppomppu.co.kr/zboard/zboard.php?id=experience
    - https://kmong.com/category/24001
    - https://www.frip.co.kr/category/beauty/all?page=2
    - https://meta-chehumdan.com/campaign_list.php?category_id=001A

### 2. 데이터 탐색 및 전처리

- 크롤링으로 수집한 모임 이름(club_title) 데이터에 대해, 데이터베이스의 컬럼명과 일치할 수 있도록 컬럼명을 바꿔주었습니다.

- <details>
    <summary>Click to see full code</summary>

        c_df = c_df.rename(columns={'club_title': 'club_name'})
        c_df

</details>

![club_rename](https://github.com/hyuncoding/django_with_ai/assets/134760674/1be13d35-434a-4152-a821-88a5e06ca5e0)

- 데이터베이스에 INSERT하기 위해 필요한 컬럼들을 추가하였습니다.
- 대표 카테고리의 경우 모임의 종류에 맞는 카테고리의 id를 직접 넣어주었습니다.
- 지역의 경우 `random.randint()`을 통해 지역 id를 무작위로 넣었습니다.
- 카테고리id에 맞는 카테고리명을 '모임 소개(club_intro)'와 '모임 정보(club_info)'에 입력하였습니다.
- '상태(status)'의 경우 1(True)을 입력하였습니다.
- '모임장 id(member_id)'의 경우 데이터베이스에 존재하는 회원의 id중 1개를 무작위로 입력하였습니다.
- '생성일(created_date)'과 '수정일(updated_date)'은 2024년 1월 1일과 현재 날짜 사이의 날짜를 임의로 넣었으며, 수정일이 생성일보다 앞서지 않도록 조건을 추가하였습니다.
- '모임 프로필 사진(club_profile_path)'과 '모임 배너 사진(club_banner_path)'에는 데이터베이스 테이블 INSERT 시 null이 허용되지 않으므로,  
  공백(' ')으로 넣어주었습니다.

- <details>
    <summary>Click to see full code</summary>

        import random
        import numpy as np
        from datetime import datetime, timedelta

        regions = ['서울', '경기', '인천', '부산', '울산' '경남', '대구', '경북', '충청', '대전', '세종', '전라', '광주', '강원', '제주']
        
        c_df['club_region_id'] = c_df['club_region_id'].apply(lambda x: random.randint(1, 15))
        
        categories = ['취미', '문화·예술', '운동·액티비티', '푸드·드링크', 
                          '여행·동행', '성장·자기개발', '동네·또래', '연애·소개팅',
                          '재테크', '외국어', '스터디', '지역축제', '기타']
        
        # 카테고리 id에 따른 카테고리 이름을 매핑하는 딕셔너리
        category_dict = {i+1: categories[i] for i in range(13)}
        
        
        intro_templates = [
            "{club_name}는 {category_name}에 관심이 많은 분들을 위한 모임입니다.",
            "{club_name}는 {category_name}을(를) 함께 즐기고 배우는 모임입니다.",
            "이 모임, {club_name}는 {category_name}에 흥미를 가진 분들이 모이는 곳입니다.",
            "{category_name}을(를) 좋아하는 사람들이 모인 {club_name}에 오신 것을 환영합니다.",
            "{club_name}는 {category_name} 관련 다양한 활동을 함께하는 모임입니다."
        ]
        
        # 다양한 정보 문구 리스트
        info_templates = [
            "{club_name} 모임은 {category_name}에 대한 열정을 가진 사람들이 모이는 곳입니다.",
            "{club_name} 모임에서는 {category_name}에 관한 다양한 주제로 활동을 합니다.",
            "{category_name}을(를) 좋아하는 사람들과 함께하는 {club_name} 모임입니다.",
            "{category_name} 관련 정보를 교환하고 함께 성장하는 {club_name} 모임입니다.",
            "{club_name} 모임은 {category_name}에 관심 있는 분들을 위한 다양한 프로그램을 제공합니다."
        ]
        
        # 'club_intro'와 'club_info' 컬럼 생성
        c_df['club_intro'] = c_df.apply(lambda row: random.choice(intro_templates).format(club_name=row['club_name'], category_name=category_dict[row['club_main_category_id']]), axis=1)
        c_df['club_info'] = c_df.apply(lambda row: random.choice(info_templates).format(club_name=row['club_name'], category_name=category_dict[row['club_main_category_id']]), axis=1)
        
        c_df['status'] = 1
        
        member_ids = [i for i in range(2, 10019)]
        c_df['member_id'] = c_df['member_id'].apply(lambda x: random.choice(member_ids))
        
        # 랜덤 날짜 생성 함수
        def random_date(start, end):
            return start + timedelta(
                seconds=random.randint(0, int((end - start).total_seconds())),
            )
        
        # 현재 시간 및 범위 설정
        now = datetime(2024, 5, 19, 4, 23)
        start = datetime(2024, 1, 1, 12, 0)
        
        # 'created_date'와 'updated_date' 컬럼 생성
        created_dates = [random_date(start, now) for _ in range(len(c_df))]
        updated_dates = [random_date(created_date, now) for created_date in created_dates]
        
        c_df['created_date'] = [date.strftime('%Y-%m-%d %H:%M:%S') for date in created_dates]
        c_df['updated_date'] = [date.strftime('%Y-%m-%d %H:%M:%S') for date in updated_dates]

        c_df['club_profile_path'] = ' '
        c_df['club_banner_path'] = ' '

        c_df.to_csv('./datasets/club_data.csv', index=False)

</details>

- csv로 내보낸 파일을 데이터베이스로 IMPORT 한 후, 모임마다 대표 카테고리를 제외한 나머지 카테고리 중 1~3개를 무작위로 골라 데이터프레임으로 생성하였습니다.
- 해당 데이터를 입력할 데이터베이스 테이블의 컬럼에 맞도록 조정하였습니다.
- '생성일(created_date)'과 '수정일(updated_date)'은 모임 데이터와 동일한 방식으로 생성하였습니다.

- <details>
    <summary>Click to see full code</summary>

        import pandas as pd
        import random
        from datetime import datetime, timedelta
        
        # 현재 시간 및 범위 설정
        now = datetime(2024, 5, 19, 4, 23)
        start = datetime(2024, 1, 1, 12, 0)
        
        # 랜덤 날짜 생성 함수
        def random_date(start, end):
            return start + timedelta(
                seconds=random.randint(0, int((end - start).total_seconds()))
            )
        
        # tbl_club_category 데이터프레임 생성
        club_category_data = []
        
        created_dates = [random_date(start, now) for _ in range(len(c_df))]
        updated_dates = [random_date(created_date, now) for created_date in created_dates]
        
        for _, row in c_df.iterrows():
            club_id = row['id']
            main_category_id = row['club_main_category_id']
            
            # 대표 카테고리를 제외한 1~13의 카테고리 중 랜덤 선택 (1~3개)
            category_ids = [i for i in range(1, 14) if i != main_category_id]
            chosen_categories = random.sample(category_ids, random.randint(1, 3))
           
            for category_id in chosen_categories:
                created_date = random_date(start, now)
                updated_date = random_date(created_date, now).strftime('%Y-%m-%d %H:%M:%S')
                club_category_data.append({
                    'created_date': created_date.strftime('%Y-%m-%d %H:%M:%S'),
                    'updated_date': updated_date,
                    'status': 1,  
                    'category_id': category_id,
                    'club_id': club_id
                })
        
        # 데이터프레임으로 변환
        club_category_df = pd.DataFrame(club_category_data)
        
        # CSV 파일로 저장
        club_category_df.to_csv('./datasets/tbl_club_category.csv', index=False)

        club_category_df

  </details>

![club_category](https://github.com/hyuncoding/django_with_ai/assets/134760674/6cf6e333-05f9-4a0f-a504-e05203cd05ec)


