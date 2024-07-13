# 모듈
여러 컴포넌트를 조합하여 좀 더 큰 작업을 수행할 수 있게 하는 단위.
Nest 애플리케이션에는 하나의 root 모듈이 존재하고 이 루트 모듈은 다른 모듈들로 구성된다. 이로 인해 각 모듈에 책임을 나누어 응집도를 높일 수 있다. 모듈이 커지게 될 경우 해당 모듈을 분리하여 MSA로 분리할 수도 있다.

### @Module
클래스 데커레이터로 인수로 ModuleMetadata 객체를 받는다. 다음은 ModuleMetadata의 속성이다.
* import: 이 모듈에서 사용하기 위한 프로바이더를 가지고 있는 다른 모듈을 가져온다.
* controllers, providers: 이 모듈에서 컨트롤러와 프로바이더를 사용할 수 있도록 Nest가 객체를 생성하고 주입할 수 있게 등록해준다.
* export: 이 모듈에서 제공하는 컴포넌트를 다른 모듈에서 import 할 수 있도록 export 해준다. export 하게되면 해당 컴포넌트는 public 인터페이스 또는 API로 간주된다.(import한 모듈을 다시 export 하는 것 또한 가능하다.)

### @Global
모듈에 @Global 데커레이터를 사용할 경우 해당 모듈은 전역 모듈이 된다. 단, 전역 모듈은 루트 모듈이나 코어 모듈에서 한 번만 등록해야 한다.

### 동적 모듈
##### dotenv를 활용한 동적 config 설정
1. 환경 별로 별개의 .env 파일을 생성한다.
2.  package.json의 script에 어떤 환경인지 명시하는 환경 변수를 설정하도록 기입한다.
3.  main.ts 파일의 bootstrap이 실행되기 전에 path 속성을 가진 객체를 인자로 dotenv.config 메서드를 실행하고 path 속성의 값을 환경 변수의 값을 읽어서 해당 환경에 따라 해당되는 .env 파일의 경로를 설정한다.
4. process.env.환경변수명 변수를 통해 .env 파일의 값을 읽어서 사용한다.

##### @nestjs/config 패키지를 활용한 ConfigModule 동적 생성
1. 루트 모듈에 ConfigModle.forRoot( )를 (DynamicModule을 return 하는 메서드) import 해준다.
2. 위의 메서드에 인자로 envFilePath 속성을 갖는 `ConfigModuleOptions`객체를 넣어준다. 해당 속성은 dotenv의 path 속성의 역할과 같다.
3. ConfigService 프로바이더를 컴포넌트에 주입하여 .env 파일에서 읽은 환경 변수를 ConfigService의 get 인스턴스 메서드를 통해 해당 컴포넌트에서 사용할 수 있다.
`ConfigModuleOptions`에는 여러 속성을 통해 다양한 옵션을 제공한다. [@nestjs/config 공식 튜토리얼 문서](https://docs.nestjs.com/techniques/configuration)
- **`isGlobal`**: `ConfigModule`을 애플리케이션 전체에 글로벌 모듈로 등록할지 여부를 설정한다.
- **`envFilePath`**: 환경 변수를 로드할 `.env` 파일의 경로 또는 경로 배열을 지정한다.
- **`ignoreEnvFile`**: `.env` 파일을 무시하고 환경 변수를 로드하지 않을지 여부를 설정한다.
- **`expandVariables`**: 환경 변수 값 내에서 `${VAR_NAME}` 형태로 다른 환경 변수를 참조할 수 있게 한다.
- **`validationSchema`**: `Joi` 라이브러리를 사용하여 환경 변수의 유효성을 검증하는 스키마를 정의한다.
- **`validationOptions`**: `Joi` 검증에 대한 추가 옵션을 설정한다.
- **`load`**: 설정 객체를 동적으로 로드하기 위한 팩토리 함수의 배열을 제공한다.
- **`cache`**: 설정 값을 캐시할지 여부를 결정다.