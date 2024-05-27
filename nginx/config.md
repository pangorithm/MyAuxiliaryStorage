
### config context example
``` bash

user www-data; # NGINX 서버를 동작시킬 linux 사용자
worker_processes auto; # 사용할 thread 개수
pid /run/nginx.pid; # NGINX pid가 적혀있는 파일
include /etc/nginx/modules-enabled/*.conf; # 외부 설정 파일 가져오기

events { # Nginx 서버의 이벤트 처리와 관련된 설정을 정의
	worker_connections 1024; # 각 worker 프로세스가 동시에 처리할 수 있는 최대 연결 수; 
	multi_accept on; # 한번에 여러 개의 새로운 연결을 수락할 지 지정(기본값: off)
	accept_mutex on; # 여러 worker 프로세스가 동시에 연결을 수락하는 것을 방지하는 
	# 뮤텍스를 사용 여부(기본값: on)
}

http { # HTTP traffic
	
	upstream backend-server { # 로드 밸런싱을 위해 여러 백엔드 서버를 그룹화
		ip_hash; # 로드벨런싱 알고리즘
		server backend1.example.com weight=3; # 로드밸런싱 가중치 부여
		server backend2.example.com max_fails=3 fail_timeout=30s; # 수동적 헬스체크
	}
	
	server { # 가상 서버 설정 
		listen 80; # ip(또는 도메인)와 port를 설정 가능하다
		# 여러개의 가상 서버가 있을 경우 ip가 매핑되는 가상서버가 port보다 우선순위가 높다
		server_name www.example.org; # listen 조건이 같을 경우 사용된다
		
		location / { # 가상 서버에 요청되는 경로
			root /static # 루트 디렉토리 설정
		}
		
		location /data {
			proxy_pass http://front-server.com; # 리버스 프록시 설정
		}
		
		location /api {
			proxy_pass http://backend-server; # upstream을 통해 로드밸런싱 된다
		}
	}
	 
	server {
		listen 127.0.0.1;
		server_name www.example.org;
	}
	 
	server {
		listen 127.0.0.1:80;
	}
	
	include /etc/nginx/conf.d/*.conf; 
	include /etc/nginx/sites-enabled/*; 
}


stream { # TCP and UDP traffic
	
	upstream tcp_backend { # TCP 업스트림 서버 그룹 정의 
		server tcp1.example.com:3306; 
		server tcp2.example.com:3306; 
		server tcp3.example.com:3306; 
	} 
	
	upstream udp_backend { # UDP 업스트림 서버 그룹 정의
		server udp1.example.com:53; 
		server udp2.example.com:53; 
		server udp3.example.com:53; 
	} 
	
	server { # TCP 부하 분산 설정 
		listen 3306; 
		proxy_pass tcp_backend; 
	} 
	
	server { # UDP 부하 분산 설정 
		listen 53 udp; 
		proxy_pass udp_backend; 
	}
}

```



