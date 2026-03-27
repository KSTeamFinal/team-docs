React Router 기반 라우팅 환경 구축

1. 개요
프로젝트 CLAIR의 멀티 페이지 전환을 위해 react-router-dom의 최신 데이터 라우터(Data Router) 방식을 도입하고, 각 화면(Screens) 간의 네비게이션 흐름을 연결함

2. 주요 작업 내용

2-1. 라이브러리 설치 및 환경 설정
브라우저 환경에 최적화된 라우팅을 위해 react-router-dom을 설치하고, TypeScript 환경에서 타입 오류가 발생하지 않도록 @types/react-dom을 추가함
    - 패키지 : react-router-dom을

2-2. 중앙 집중형 라우트 관리 (scr/app/routes.tsx)
createBrowserRouter를 사용하여 애플리케이션의 모든 경로와 컴포넌트를 한곳에서 관리하도록 설정함
    - 설정 경로 : 
        - / : StartScreen (메인 시작 화면)
        - /onboarding : Onboarding (서비스 가이드)
        - /login : Login(사용자 인증)
        - 그 외 SingUp, Home, Upload, ResultDashboard 등 전체 화면 매핑 완ㅇ료

2-3. 최상단 라우터 주입 (src/main.txt & App.tsx)
기존의 <BrowserRouter> 방식 대신, 최신 권장 방식인 RouterProvider를 App.tsx에 적용하여 라우팅 컨텍스트를 하위 컴포넌트에 주입함
    - main.tsx에서 App을 호출하고, App 내부에서 routes.tsx에서 정의한 router 객체를 전달함

2-4. 컴포넌트 간 이동 로직 구현(useNavigate)
사용자 경험(UX)을 위해 버튼 클릭 시 페이지가 새로고침 없이 부드럽게 전환되도록, useNavigate 훅을 적용함.
    -StartScreen 적용 예시 : 
        -[시작하기] 버튼 클릭 시 /onboarding으로 이동
        -[로그인] 링크 클릭 시 /login으로 이동

3. 기술적 이슈 및 해결 (Troubleshooting)
    -Issue : useNavigate 사용시 "used only in the context of a <Router>" 에러 발생
    -Cause : react-router(코어)와 react-router-dom(웹 전용) 패키지가 혼용되어 라우터 컨텍스트가 정상적으로 전달되지 않음
    -Solution : 모든 임포트 경로를 react-router-dom으로 동일하고, RouterProvider 계층 구조를 명확히 하여 해결함

4. 결과물
    - 브라우저 주소창에 직접 경로 입력 시 해당 화면 정상 렌더링 확인
    - 화면 내 버튼 클릭을 통한 페이지 전환 기능 정상 작동 확인

