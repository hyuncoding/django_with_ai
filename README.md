## 🖥️ AI 서비스 상용화 프로젝트

#### 대상 프로젝트

> 청년들의 문화생활 플랫폼 '틴플레이(Teenplay)'  
> Git-hub: https://github.com/hyuncoding/teenplay_server  
> Website: http://teenplay.site

#### AI 서비스

> 회원별 활동 추천 서비스

### 📌 목차

1. 데이터 수집
2. 데이터 탐색 및 전처리
3. 모델 학습
4. 모델 평가

---

### 1. 데이터 수집

-   `selenium` 라이브러리를 통한 크롤링으로 데이터를 수집하였습니다.

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
              titles = driver.find_elements(By.CSS_SELECTOR, ".title__UUn4H span")  # "title-class-name"을 실제 클래스 이름으로 변경

              # 제목 텍스트 출력
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
