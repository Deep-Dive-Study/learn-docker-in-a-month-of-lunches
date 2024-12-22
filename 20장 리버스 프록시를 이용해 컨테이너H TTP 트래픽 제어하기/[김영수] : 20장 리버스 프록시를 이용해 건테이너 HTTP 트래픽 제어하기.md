# 20장 리버스 프록시를 이용해 컨테이너 HTTP 트래픽 제어하기

외부에서 들어온 트래픽을 컨테이너까지 이어주는 라우팅은 도커 엔진이 담당하지만, 컨테이너가 주시할 수 있는 포트는 하나뿐이다. 클러스터 하나에서 수없이 많은 애플리케이션을 실행해야 하기 때문이고 또한 이들 모두를 http와 https 80/443을 통해 접근가능하게 해야해서 포트가 부족하다.



리버스 프록시는 이런 경우에 유용하다. 

## 20.1 리버스 프록시란?

프록시는 클라이언트를 감춰준다. 리버스 프록시는 서버를 감춰주는것이다.

**리버스 프록시(Reverse Proxy)**는 클라이언트가 직접 백엔드 서버에 요청을 보내지 않고, **프록시 서버가 클라이언트 요청을 받아 백엔드 서버로 전달하고 응답을 다시 클라이언트로 전달**하는 방식의 서버다.

![image-20241221001207357](./images//image-20241221001207357.png)

리버스 프록시 덕분에 컨테이너는 외부에 노출될 필요 없으며, 그만큼 스케일링 업데이트 보안면에서 유리하다.

엔진엑스는 웹 사이트별로 설정 파일을 둘 수 있다(도메인별로)

아래는 각각 다른 파일이다. 

```
1
server {
    server_name api.numbers.local;

    location / {
        proxy_pass             http://numbers-api;
        proxy_set_header       Host $host;
        add_header             X-Host $hostname;         
    }
}


2
server {
    server_name image-gallery.local;

    location = /api/image {
        proxy_pass             http://iotd/image;
        proxy_set_header       Host $host;
        add_header             X-Proxy $hostname;         
        add_header             X-Upstream $upstream_addr;
    }

    location / {
        proxy_pass             http://image-gallery;
        proxy_set_header       Host $host;
        add_header             X-Proxy $hostname;         
        add_header             X-Upstream $upstream_addr;
    }        
}


3
server {
    server_name whoami.local;

    location / {
        proxy_pass             http://whoami;
        proxy_set_header       Host $host;
        add_header             X-Host $hostname;         
    }
}
```

http 요청 헤더에 Host라는 사이트 정보를 넣는데,(도메인) 이 정보로 해당 요청을 처리할 수 있는 사이트의 설정 파일을 찾아 트래픽을 연결한다. 

위 설정파일을 엔진엑스에 넣어주고, 네트워크를 같은 네트워크끼리 묶으면 엔진엑스가 앞단에서 요청을 받아 적절한 컨테이너에게 라우팅 해준다. 

## 20.3 프록시를 이용한 성능 및 신뢰성 개선

캐싱을 활용해서, 로컬디스크나 메모리에 저장해두었다 저장된것을 돌려줄 수 있다.

캐싱 프록시의 장점은 요청 처리하는 시간을 줄일 수 있으며 트래픽을 줄일 수 있어 더많은 요청을 처리할 수 있다.

단 인증 정보를 포함한 요청은 캐싱하지 않으면 개인화된 콘텐츠는 제외할 수 있다. 

```
server {
    server_name image-gallery.local;
    listen 80;
	return 301 https://$server_name$request_uri;
}

server {
	server_name  image-gallery.local;
	listen 443 ssl;
    
    gzip  on;    
    gzip_proxied any;

	ssl_certificate        /etc/nginx/certs/server-cert.pem;
	ssl_certificate_key    /etc/nginx/certs/server-key.pem;
	ssl_session_cache      shared:SSL:10m;
	ssl_session_timeout    20m;
	ssl_protocols          TLSv1 TLSv1.1 TLSv1.2;

	ssl_prefer_server_ciphers on;
	ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';

	add_header  Strict-Transport-Security "max-age=31536000" always;

    # 특정 경로 `/api/image`에 대한 요청을 처리
    # 프록시로 지정된 URL로 요청을 전달 (http://iotd/image)
    # 클라이언트의 Host 헤더를 백엔드 서버로 전달
    # SHORT 캐시 정책을 적용
    # HTTP 상태 코드 200인 응답은 1분 동안 캐싱
    # 캐싱 상태를 나타내는 헤더를 클라이언트 응답에 추가 (X-Cache)
    # 프록시 서버의 호스트 이름을 응답 헤더에 추가 (X-Proxy)
    # 프록시로 전달된 백엔드 서버 주소를 응답 헤더에 추가 (X-Upstream)
    location = /api/image {
        proxy_pass             http://iotd/image;
        proxy_set_header       Host $host;
        proxy_cache            SHORT;
        proxy_cache_valid      200  1m;
        add_header             X-Cache $upstream_cache_status;
        add_header             X-Proxy $hostname;         
        add_header             X-Upstream $upstream_addr;
    }

    # 기본 경로 `/`에 대한 요청 처리
    # 프록시로 지정된 URL로 요청을 전달 (http://image-gallery)
    # 클라이언트의 Host 헤더를 백엔드 서버로 전달
    # LONG 캐시 정책을 적용
    # HTTP 상태 코드 200인 응답은 6시간 동안 캐싱
    # 특정 오류나 시간 초과가 발생하면 캐시된 오래된 데이터를 사용 (http_500, http_502 등 포함)
    # 캐싱 상태를 나타내는 헤더를 클라이언트 응답에 추가 (X-Cache)
    # 프록시 서버의 호스트 이름을 응답 헤더에 추가 (X-Proxy)
    # 프록시로 전달된 백엔드 서버 주소를 응답 헤더에 추가 (X-Upstream)
    location / {
        proxy_pass             http://image-gallery;
        proxy_set_header       Host $host;
        proxy_cache            LONG;
        proxy_cache_valid      200  6h;
        proxy_cache_use_stale  error timeout invalid_header updating
                               http_500 http_502 http_503 http_504;
        add_header             X-Cache $upstream_cache_status;
        add_header             X-Proxy $hostname;         
        add_header             X-Upstream $upstream_addr;
    }

}
```

그 외에도 gzip 압축, 캐시 헤더 추가 등을 활용할 수 있다. 

## 20.5 리버스 프록시를 활용한 패턴의 이해

리버스 프록시가 있어야 적용할 수 있는 세 가지 주요 패턴이 있다.

1. 클라이언트 요청에 포함된 호스트명을 통해 http 혹은 https로 제공되는 애플리케이션에서 콘텐츠 제공하는 패턴. 즉 알맞는 도메인으로 알맞은 컨테이너로 전달하여 요청을 제공한다 
2. 한 애플리케이션이 여러 컨테이너에 걸쳐 실행되는 msa 패턴에 실행된다. 요소 중 일부만을 선택적으로 요청하여 외부에서는 하나의 도메인을 갖지만 경로에 따라 여러 서비스가 요청을 처리하는 구조 
3. 모놀리식 설계를 가진 애플리케이션을 컨테이너로 이주시킬때 유용한 패턴이다. 리버스 프록시를 두어 모놀리식 설게를 가진 애플리케이션의 프론트엔드를 맡기고, 추가되는 기능은 컨테이너로 분할한다. 즉 점진적으로 이동한다. 