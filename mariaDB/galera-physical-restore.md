
1. 원본 서버와 복제할 서버의 mariadb 버전이 같아야 함

```bash
mysql --version
```

2. mysql 폴더 압축(병렬)

```bash
tar cf - mysql | pbzip2 > /backup/mysql.tar.bz2
```

3. 복제할 서버에 압축 파일 이동
4. 복제할 서버의 기존 mysql 폴더 비우기

```bash
cd /db/mysql
rm -rf *
```

5. 복제할 서버에서 압축 해제

```bash
cd /db/mysql
cd ..
pbzip2 -d -c mysql.tar.bz2 | tar xf -
```

6. 복제할 서버에서 gvwstate.dat 삭제

```bash
cd mysql
rm gvwstate.dat
```

7. grastate.dat 파일에서 safe_to_bootstrap을 1로 변경

```bash
vi grastate.dat

(중략)
safe_to_bootstrap: 1
```

8. 갈레라 클러스터 구성

```bash
# 마스터
galera_new_cluster
service mysql start --wsrep-new-cluster  # 마리아디비 10.0 버전 (galera_new_cluster 명령 미지원)

# 나머지
systemctl start mysql
```

9. 복제 상태 확인