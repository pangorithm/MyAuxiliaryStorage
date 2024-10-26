### HTTP 예제
```rust
use hyper::{body::HttpBody as _, Client};
use tokio::io::{stdout, AsyncWriteExt as _};

#[tokio::main]
async fn main() {
    let client = Client::new();
    let uri = "http://httpbin.org/ip".parse().unwrap();
    let mut resp = client.get(uri).await.unwrap(); // HTTP 요청을 보냅니다.
    println!("Response: {}", resp.status());

    while let Some(chunk) = resp.body_mut().data().await {
        // body 값을 확인합니다.
        stdout().write_all(&chunk.unwrap()).await.unwrap();
    }
}
```

### REST API 예제
```rust
use hyper::body::{self, Buf};
use hyper::{client, Client};
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct User {
    id: i32,
    name: String,
}

#[tokio::main]
async fn main() {
    let url = "http://jsonplaceholder.typicode.com/users".parse().unwrap();

    let client = Client::new();
    let res = client.get(url).await.unwrap();
    let body = hyper::body::aggregate(res).await.unwrap();

    let users: Vec<User> = serde_json::from_reader(body.reader()).unwrap(); // 받은 JSON을 serde로 역직렬화

    println!("사용자: {:#?}", users);
}
```

### 간단한 웹서버 예제
```rust
use hyper::service::{make_service_fn, service_fn};
use hyper::{Body, Method, Request, Response, Server, StatusCode};

type GenericError = Box<dyn std::error::Error + Send + Sync>;
type Result<T> = std::result::Result<T, GenericError>;

async fn response_examples(req: Request<Body>) -> Result<Response<Body>> {
    let index_html = String::from("<h1>Hello World!</h1>");
    let notfound_html = String::from("<h1>404 not found</h1>");

    match (req.method(), req.uri().path()) {
        // method와 url을 기준으로 핸들러를 할당
        (&Method::GET, "/") => Ok(Response::new(index_html.into())),
        _ => {
            // 404 오류 발생
            Ok(Response::builder()
                .status(StatusCode::NOT_FOUND)
                .body(notfound_html.into())
                .unwrap())
        }
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let addr = "127.0.0.1:8080".parse().unwrap();
    let new_service = make_service_fn(move |_| {
        // 서비스를 생성한다.
        async { Ok::<_, GenericError>(service_fn(move |req| response_examples(req))) }
    });

    let server = Server::bind(&addr).serve(new_service); // 서비스를 구동한다.
    println!("Listening on http://{}", addr);
    server.await?;
    Ok(())
}
```