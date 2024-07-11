### package.json
* 애플리케이션이 필요로 하는 패키지 목록 나열
* 각 패키지는 시맨틱 버저닝 규칙으로 필요한 버전을 기술한다.
* 다른 개발자와 같은 빌드 환경을 구성한다.

##### 유의적 버전(Semantic Versioning, SemVer)
[Major].[Minor].[Patch]-[label]
* Major: 이전 버전과 호환이 불가능할 경우 증가.
* Minor: 기능이 추가되는 경우 증가
* Patch: 버그 수정 패치를 적용할 경우 증가
* label: 선택 사항. pre, alpha, beta와 같은 부가 설명

##### 버전 범위
* `ver`, `=ver`: 완전히 일치
* `>ver`: 큰 버전
* `>=ver`: 크거나 같은 버전
* `<ver` : 작거나 같은 버전
* `~ver`: 지정한 마지막 자리가 일치하는 버전
* `^ver`: 지정한 버전 이상의 Major가 일치하는 버전

##### package-lock.json
npm install 시에 생성되며 node_modules나 package.json 파일의 내용이 바뀌면 npm install 명령을 수행할 때 자동으로 수정된다.  package-lock.json 파일은 package.json에 선언된 패키지들이 설치될 때의 정확한 버전과 서로 간의 의존성을 표현한다. 따라서 npm install 명령을 실행하면 npm은 package-lock.json과 package.json을 모두 읽고 패키지 정보가 일치할 경우 package-lock.json 읽어서 의존성 패키지를 설치하고 package-lock.json이 없을 경우나 패키지 정보가 일치하지 않을 경우에 package.json을 통해 의존성 패키지를 설치하고 package-lock.json을 작성한다.

##### package.json의 프로퍼티
* name: 패키지명
* private: 패키지 공개 여부
* version: 버전
* description: 패키지에 대한 설명
* license: 패키지의 라이선스
* scripts: npm run 명령어로 수행할 단축 명령어
* dependencies: 패키지가 의존하는 다른 패키지
* devDependencies: 개발 환경에서만 사용할 패키지
* jest: 테스팅 라이브러리인 Jest를 위한 환경 구성 옵션(NestJS의 경우 기본적으로 Jest를 제공)