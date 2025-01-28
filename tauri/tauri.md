
### 프로젝트 init
`npm create tauri-app@latest`

### 프로젝트 명령어
공식문서) https://v2.tauri.app/reference/cli/
```npm
# 프로젝트 npm 명령어
dev
  vite
build
  vite build
preview
  vite preview
tauri
  tauri
  
# 프로젝트 tauri 명령어
Usage: npm run tauri [OPTIONS] <COMMAND>

Commands:
  init         Initialize a Tauri project in an existing directory
  dev          Run your app in development mode
  build        Build your app in release mode and generate bundles and installers
  bundle       Generate bundles and installers for your app (already built by `tauri build`)
  android      Android commands
  migrate      Migrate from v1 to v2
  info         Show a concise list of information about the environment, Rust, Node.js and their versions as well as a few relevant project configurations
  add          Add a tauri plugin to the project
  remove       Remove a tauri plugin from the project
  plugin       Manage or create Tauri plugins
  icon         Generate various icons for all major platforms
  signer       Generate signing keys for Tauri updater or sign files
  completions  Generate Tauri CLI shell completions for Bash, Zsh, PowerShell or Fish
  permission   Manage or create permissions for your app or plugin
  capability   Manage or create capabilities for your app
  inspect      Manage or create permissions for your app or plugin
  help         Print this message or the help of the given subcommand(s)

Options:
  -v, --verbose...  Enables verbose logging
  -h, --help        Print help
  -V, --version     Print version


### 프로젝트 tauri android 명령어
Usage: npm run tauri android [OPTIONS] <COMMAND>

Commands:
  init   Initialize Android target in the project
  dev    Run your app in development mode on Android
  build  Build your app in release mode for Android and generate APKs and AABs
  help   Print this message or the help of the given subcommand(s)

Options:
  -v, --verbose...  Enables verbose logging
  -h, --help        Print help
  -V, --version     Print version
```

##### 안드로이드 빌드
공식문서) https://v2.tauri.app/distribute/sign/android/
앱 빌드(apk 생성)를 하기 위해서는 npm run tauri android build 후에 안드로이드 스튜디오에서 generate signed app 통해 서명 파일을 생성하고 
build.gradle.kts 파일의 android 블럭 안에 아래 내용을 추가해서 서명을 추가한 후에 재빌드 해야 한다.
```
android {
    ...
    signingConfigs {
        create("release") {
            storeFile = file("path/to/your/keystore/file.jks") // keystore 파일 경로
            storePassword = "yourKeystorePassword" // keystore 비밀번호
            keyAlias = "yourKeyAlias" // 키 alias
            keyPassword = "yourKeyPassword" // 키 비밀번호
        }
    }

    buildTypes {
        getByName("release") {
            isMinifyEnabled = true // 릴리즈 빌드에서 ProGuard 설정 (옵션)
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("release") // 서명 설정
        }
    }
}
```