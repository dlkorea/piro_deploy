# 배포
본 문서는 ubuntu 16.04 버전 기준.
다른 OS 또는 ubuntu 다른 버전에서는 세팅 방법이 다름.

## 1. apt-get update

    sudo apt-get update

## 2. pip3 install

    sudo apt-get install python3-pip

## 3. virtualenv, virtualenvwrapper install
    sudo -H pip3 install --upgrade pip
    sudo -H pip3 install virtualenv virtualenvwrapper

    echo "export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3" >> ~/.bashrc
    echo "export WORKON_HOME=~/Env" >> ~/.bashrc
    echo "source /usr/local/bin/virtualenvwrapper.sh" >> ~/.bashrc
    source ~/.bashrc


## 4. virtualenv 생성 및 활성화

    mkvirtualenv 프로젝트명


## 5. django settings update
ALLOWED_HOSTS, MEDIA_ROOT, STATIC_ROOT 설정


## 6. uwsgi install

    sudo apt-get install python3-dev
    sudo -H pip3 install uwsgi


## 7. uwsgi settings

    sudo mkdir -p /etc/uwsgi/sites
    sudo vim /etc/uwsgi/sites/프로젝트명.ini

아래 내용을 붙여넣기.

    [uwsgi]
    project = 프로젝트명
    uid = ubuntu
    base = /home/%(uid)

    chdir = %(base)/%(project)
    home = %(base)/Env/%(project)
    module = %(project).wsgi:application

    master = true
    processes = 5

    socket = /run/uwsgi/%(project).sock
    chown-socket = %(uid):www-data
    chmod-socket = 660
    vacuum = true

아직 위 설정을 통해 uwsgi를 실행할 수는 없음. www-data 유저가 없기 때문. www-data 유저는 nginx 설치시 자동 생성됨.

## 8. create uwsgi systemd unit files(uwsgi 자동실행 관련 설정 파일)

    sudo vim /etc/systemd/system/uwsgi.service

아래 내용을 붙여넣기.

    [Unit]
    Description=uWSGI Emperor service

    [Service]
    ExecStartPre=/bin/bash -c 'mkdir -p /run/uwsgi; chown ubuntu:www-data /run/uwsgi'
    ExecStart=/usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
    Restart=always
    KillSignal=SIGQUIT
    Type=notify
    NotifyAccess=all

    [Install]
    WantedBy=multi-user.target


## 9. nginx install

    sudo apt-get install nginx


## 10. nginx settings

    sudo vim /etc/nginx/sites-available/프로젝트명

아래 내용을 붙여넣기.

    server {
        listen 80;
        server_name 도메인.com www.도메인.com;

        location = /favicon.ico { access_log off; log_not_found off; }
        location /static/ {
            root /home/ubuntu/프로젝트명;
        }

        location / {
            include         uwsgi_params;
            uwsgi_pass      unix:/run/uwsgi/프로젝트명.sock;
        }
    }


실제 nginx 설정은 sites-enabled 안에 포함된 것들만 반영됨. 따라서 sites-available에 만들어둔 파일에 대해
ln -s 로 sites-enabled에 심볼 링크(일종의 바로가기)를 만들어야 함. 사이트 설정은 때에 따라 빠질수도 수정될 수도 있기에
이렇게 처리해두면 차후 서빙 목록에서 뺄 때 sites-enabled에 있는 심볼 링크만 삭제하면 됨.

    sudo ln -s /etc/nginx/sites-available/프로젝트명 /etc/nginx/sites-enabled


## 11. nginx restart & uwsgi start

nginx 설정에 오류가 없는지 테스트

    sudo nginx -t


이상이 없다면 nginx 재시작(설정 반영)

    sudo systemctl restart nginx


uwsgi 도 시작해줌(nginx 인스톨시 www-data 유저가 생성되어 이제는 실행 가능.)

    sudo systemctl start uwsgi


## 12. ubuntu 방화벽 설정

    sudo ufw delete allow 8080
    sudo ufw allow 'Nginx Full'


## 13. nginx, uwsgi 자동 실행(부팅시) 설정

    sudo systemctl enable nginx
    sudo systemctl enable uwsgi


## 14. 에러 로그 보는 법

/var/log/nginx 에 가보면 error.log error1.log error2.log 등의 파일이 있음(나중에 생김)

tail -F 명령어는 파일의 젤 아랫부분 몇 줄을 모니터링 형식으로 보여줌(파일이 갱신되면 갱신해줌)
이를 이용해서 에러 모니터링을 할 수 있음.

    sudo tail -F /var/log/nginx/error.log


## 15. mysql 설치

    sudo apt-get install mysql-server
    mysql_secure_installation


## 16. mysql에 유저와 database 생성

root 유저로 mysql 실행

    mysql -uroot -p

장고가 사용할 유저 생성

    create user '사용자명'@'localhost' identified by '비밀번호';


장고가 사용할 db 생성(프로젝트명으로 db 생성)

    create database 프로젝트명 default character set utf8 collate utf8_general_ci;

collate utf8_general_ci 가 없어도 됨. 허나 이게 있어야 아이폰 이모티콘 등이 에러 없이 저장됨.
collate utf8_general_ci 없을시 아이폰 유저가 이모티콘 저장 시도할 때 500 에러가 남.


## 17. mysql database 유저 권한 설정

    grant all privileges on 프로젝트명.* to '사용자명'@'localhost';
    flush privileges;


## 18. mysql connector install

    sudo apt-get install libmysqlclient-dev
    pip install mysqlclient


## 19. django database settings update

프로덕션 settings.py 에 아래 설정을 붙여넣기.

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'HOST': 'localhost',
            'NAME': '프로젝트명(db이름)',
            'USER': 'db유저명',
            'PASSWORD': '해당유저 비밀번호',
        },
    }


## 20. nginx, uwsgi restart

    sudo systemctl restart uwsgi
    sudo systemctl restart nginx


## 21. 참고 문서

<https://www.digitalocean.com/community/tutorials/how-to-serve-django-applications-with-uwsgi-and-nginx-on-ubuntu-16-04>
