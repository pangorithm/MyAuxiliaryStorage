[pm2 logs doc](https://pm2.keymetrics.io/docs/usage/log-management/)

### pm2 logging
pm2에서는 기본적으로 로그에 stdout과 stderr를 각각 out_file과 error_file에 저장한다.
js의 경우 console.out은 stdout을 console.error는 stderr를 사용해 출력하기 때문에 이를 통해 간단하게 확인이 가능하다.  
따라서 try-catch등에서 error를 로깅할 때 console.error나 다른 로그 모듈을 사용해 stderr로 출력하면 에러 로그를 error_file에 분리하여 로깅 가능하기 때문에 오류 메세지를 찾기가 더 수월해진다.