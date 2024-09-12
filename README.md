# NetSys_Obsidian
SKKU NetSys Lab Undergraduate Obsidian for Linux Network Kernel Analysis

# 협업 컨벤션

## issue

- Assignees와 Label을 달아주세요
- 해결(구현)하고자 하는 문제와, 해결(구현) 방식에 대해 간략히 설명해주세요

## branch

<issue number>-<description>
ex) 123-edit-root-layout

<description>

- 이슈를 요약
- 소문자 사용
- 동사 원형으로 시작
- 마침표 붙이지 않음
- `-`(dash) 사용

대부분의 경우, 브랜치는 최신 main 브랜치에서 분기합니다. 

```bash
git switch main
git pull
git checkout -b [브랜치] // 새 브랜치를 파서 이동하는 명령
```

## commit

Angular commit convention
<type>: <description>

ex) feat: add register button

<type>
- feat: 새로운 기능 추가
- fix: 버그 수정
- docs: 문서화
- style: 코드 스타일 변경 linting
- refactor: 코드 리팩토링
- test: 테스트 코드
- chore: 사소한 작업
- hotfix: 긴급 버그 수정

<description>

- 변경 사항을 요약
- 소문자 사용
- 동사 원형으로 시작
- 마침표 붙이지 않음
- 띄어쓰기 구분

## pull request

: 기능 브랜치에 구현 후, ‘main’에 병합하기 전에 코드 리뷰를 요청하고 검토받기 위한 절차

- 해결(구현)한 내용과 방식에 대해 자세히 설명해주세요
- ‘Closes #’ 을 입력하고 해당 이슈 번호를 달아주세요
    - 예시로 Closes #11은, 해당 PR이 병합될 때 11번 이슈도 같이 Close 됩니다
    - 만약 한 PR에 이슈 여러 개가 있는 경우 각각 달아주세요
    Closes #1
    Closes #2
    - 반대로 한 이슈에 여러 PR이 생기는 경우,
    이슈가 종료되기 전에 Close 하면 안되겠죠?
- PR의 제목은 커밋 컨벤션과 동일합니다
- Reviewer, Assignees, Label을 달아주세요
- 1명 이상이 코드리뷰 후 승인을 한 경우에만 병합할 수 있습니다.

### 코드리뷰 요령

- PR 페이지에서 . 키를 누르면
- 터미널을 킵니다
- 코드 스페이스를 생성합니다
- 코드 각 줄 왼쪽의 + 버튼을 통해 코멘트를 달 수 있습니다

### 코멘트

- 불필요한 코드
- 개선이 필요한 코드
- 다른 구현 방법이 존재하는 코드

개선이나 대체할 방법이 존재하는 경우 해당 방법에 대해서도 언급해주세요! 

마지막으로 PR 승인을 할 때에는 PR에 대해 칭찬해 주세요

ex) 컴포넌트화가 잘 됐어요, 주석이 자세해요, 좋은 라이브러리네요, 구현이 깔끔해요~
