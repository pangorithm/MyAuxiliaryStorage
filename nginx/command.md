
[# 명령줄 매개변수](https://nginxstore.com/docs/nginx/%eb%aa%85%eb%a0%b9%ec%a4%84-%eb%a7%a4%ea%b0%9c%eb%b3%80%ec%88%98/)

nginx는 다음의 명령줄 매개변수를 지원합니다.

- -? | -h — 명령줄 매개변수 도움말을 출력합니다.
- -c _file_ — 기본 파일 대신 대체 구성 \_file\_을 사용합니다.
- -e _file_ — 기본 파일 대신 대체 _error log file_ 을 사용하여 로그를 저장합니다. 특수 값 stderr은 표준 오류 파일을 선택합니다.
- -g _directives_ — 전역 구성 명령을 설정합니다. 예를 들어 다음과 같습니다.

```
nginx -g "pid /var/run/nginx.pid; worker_processes `sysctl -n hw.ncpu`;"
```

- -p _prefix_ — nginx 경로 접두사를 설정합니다. 즉, 서버 파일을 저장하는 디렉터리입니다(기본값: _/usr/local/nginx_).
- -q — 구성 테스트 동안 오류가 아닌 메시지를 숨깁니다.
- -s _signal_ — _signal_을 마스터 프로세스로 보냅니다. _signal_ 인수는 다음 중 하나가 될 수 있습니다.
    - stop — 빠른 종료
    - quit — 점진적 종료
    - reload — 구성 다시 로드, 새로운 구성으로 새로운 작업자 프로세스 시작, 기존 작업자 프로세스를 적절히 종료.
    - reopen — 로그 파일 다시 열기
- -t — 구성 파일 테스트: nginx는 올바른 구문에 대해 구성을 검사하고, 구성에서 참조된 파일을 열려고 시도합니다.
- -T — -t와 동일하지만, 추가로 구성 파일을 표준 출력으로 덤핑합니다.
- -v — nginx 버전을 출력합니다.
- -V — nginx 버전, 컴파일러 버전, 구성 매개변수를 출력합니다.


가장 대표적인 커멘드 사용으로는 설정 파일 변경 후 적용 전에 ```
``` nginx
nginx -t
```
명령어를 통해 설정 파일 문법을 검사 한 후
```nginx
nginx -s reload
```
명령어를 사용하여 변경된 설정을 적용해 다시 로드한다.