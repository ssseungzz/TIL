## K6로 부하 테스트 하기

> K6란?

그라파나 진영에서 만든 오픈 소스 부하 테스트 툴로, 자바스크립트 코드 기반으로 부하 테스트 및 테스트 시나리오를 작성 가능하다. 재사용성이 훌륭하고, 사용 방법이 간편하다. 문서화도 잘 되어 있는 편. 어지간한 건 [공식 documentation](https://grafana.com/docs/k6/latest/)에 잘 설명되어 있다.

### K6로 부하 테스트 시작하기

OS별로 직접 설치하거나, 도커를 통해 구동할 수 있다. 쿠버네티스 클러스터에서 분산 테스트를 할 수 있도록 오퍼레이터도 제공한다. 설치하고 나면, 아래와 같은 간단한 명령어로 테스트 스크립트를 실행할 수 있다.

```bash
k6 run --vus 10 --duration 30s script.js
```

* VU(가상 유저)를 10으로 설정하고, 30초 동안 테스트 스크립트의 내용을 실행한다.

### 부하 테스트 스크립트 작성하기

```javascript
import http from 'k6/http';
import { sleep } from 'k6';
export const options = {
  vus: 10,
  duration: '30s',
};
export default function () {
  http.get('http://test.k6.io');
  sleep(1);
}
```

가장 단순한 스크립트 형태는 위와 같다. 필요한 모듈을 `import` 해 주고, `options`로 VU 수, 테스트 시간을 설정해 주면 VU 하나당 `default` 함수의 내용을 반복적으로 실행한다. 

`options`에서는 VU 수, 테스트 시간뿐 아니라 테스트 스테이지별 VU 수와 duration, 그룹과 그룹별 시나리오 지정 등 다양한 기능을 제공한다. 이외에도 REST API 서버 주소, JSON 포맷의 설정, DNS, 로그 포맷, K6 자체의 설정 ON/OFF 등 다양한 옵션을 지정 가능하다. 

### 부하 테스트 메트릭

![k6 results - console/stdout output](https://grafana.com/media/docs/k6-oss/k6-results-stdout.png)

테스트를 하고 나면 위와 같은 기본적인 메트릭을 제공한다. HTTP 총 요청 수, 소요 시간 등 기본적인 빌트 인 메트릭뿐만 아니라, 커스텀 메트릭도 추가 가능하다. 또, 테스트 결과를 csv, JSON 등 다양한 형식으로 출력 가능하고, Amazon CloudWatch, prometheus, Datadog, Elasticsearch 등 다양한 소스로 전송 가능하다.

```bash
k6 run \
--out json=test.json \
--out influxdb=http://localhost:8086/k6
```

또, 웹 대시보드 옵션을 통해 테스트가 실행되는 동안 웹 대시보드로 결과를 볼 수도 있다.

```bash 
K6_WEB_DASHBOARD=true ./k6 run script.js

         /\      Grafana   /‾‾/
    /\  /  \     |\  __   /  /
   /  \/    \    | |/ /  /   ‾‾\
  /          \   |   (  |  (‾)  |
 / __________ \  |_|\_\  \_____/

     execution: local
        script: ../extensions/xk6-dashboard/script.js
 web dashboard: http://127.0.0.1:5665
        output: -
```

![Web dashboard screenshot](https://grafana.com/media/docs/k6-oss/web-dashboard-overview.png)

만들어지는 웹 대시보드는 위와 같다. 테스트가 끝나고, html 형식의 레포트를 받을 수도 있다.

추가로 그라파나 진영에서 만들어진 만큼, 대시보드를 간편하게 구성할 수 있도록 [기본 템플릿](https://github.com/grafana/xk6-output-prometheus-remote/tree/main/grafana/dashboards)도 제공되고 있다.

### K6 자세히 살펴보기

#### Life Cycle

K6의 테스트 라이프 사이클은 크게 4가지로 이루어진다.

1. `init` 컨텍스트는 스크립트 준비, 파일 로딩, 모듈 임포트, 테스트 라이프 사이클 함수 정의 등으로 이루어진다. 필수이다. 라이프사이클 함수에 있는 코드 외의 코드들은 모두 `init` 컨텍스트에 포함된다. 가장 먼저 실행된다.
2. `setUp`은 테스트 환경과 데이터를 생성하는 함수다. 필수는 아니며, 여기서 설정된 값은 모든 VU가 전역으로 접근 가능하다.
3. VU 코드는 `default` 또는 시나리오 함수로 동작한다. `options`에서 정의된 대로 동작한다. 필수이다.
4. `teardown`은 데이터 후처리 및 테스트 환경을 닫는다. 필수는 아니다.

![Diagram showing data getting returned by setup, then used (separately) by default and teardown functions](https://grafana.com/media/docs/k6-oss/lifecycle.png)

추가로, `handleSummary` 함수를 통해 테스트 결과 요약을 커스텀해서 보여 줄 수 있다.

#### HTTP

```javascript
import http from 'k6/http';

export default function () {
  const url = 'http://test.k6.io/login';
  const payload = JSON.stringify({
    email: 'aaa',
    password: 'bbb',
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
    },
  };

  http.post(url, payload, params);
}
```

다양한 HTTP method를 제공한다. 또, HTTP 요청을 태그로 구분하여 다양한 용도로 사용할 수 있도록 태그 기능을 지원한다. 태그의 이름을 사용해서 URL 그룹을 지정할 수 있다.

```javascript
import http from 'k6/http';

export default function () {
  for (let id = 1; id <= 100; id++) {
    http.get(`http://example.com/posts/${id}`, {
      tags: { name: 'PostsItemURL' }, // PostsItemURL 태그를 사용하는 uri의 메트릭을 따로 추출하는 등의 기능 사용 가능
    });
  }
}
// tags.name=\"PostsItemURL\",
// tags.name=\"PostsItemURL\",
```

#### Websocket

K6는 HTTP 외에도 HTTP/2, gRPC, SST/TLS, Websocket 등 다양한 프로토콜을 지원한다. K6를 선택한 가장 큰 이유이기도 하다. nGrinder 등 기존에 사용하던 테스트 툴은 Websocket에 대한 지원이 미비한 것에 비해 K6는 비교적 풍부한 웹소켓 테스트를 지원한다.

```javascript
import ws from 'k6/ws';
import { check } from 'k6';

export default function () {
  const url = 'ws://echo.websocket.org';
  const params = { tags: { my_tag: 'hello' } };

  const res = ws.connect(url, params, function (socket) {
    socket.on('open', () => console.log('connected'));
    socket.on('message', (data) => console.log('Message received: ', data));
    socket.on('close', () => console.log('disconnected'));
  });

  check(res, { 'status is 101': (r) => r && r.status === 101 });
}
```

가장 단순한 구조는 위와 같으며, K6 웹소켓 모듈인 k6/ws은 웹소켓 API 표준을 구현한다. HTTP와의 가장 큰 차이점은 `default` 함수를 계속 도는 것과 달리, 각 VU가 **비동기적인 이벤트 루프로 동작**한다는 것이다. 즉, `default` 함수의 반복 실행이 아니라 `default` 함수 내 구현된 이벤트 핸들러가 비동기적 이벤트 루프로 동작하는 것이다.

일단 `connect` 함수를 통해 연결이 시작되고 나면, `connect` 함수의 세 번째 인자로 주어지는 `run` 함수가 즉지 호출되며 그 안의 함수들이 실행된다. 

```javascript
import ws from 'k6/ws';
import { check } from 'k6';

export default function () {
  const url = 'ws://echo.websocket.org';
  const params = { tags: { my_tag: 'hello' } };

  const res = ws.connect(url, params, function (socket) {
    socket.on('open', function open() {
      // ...
    });

    socket.on('error', function (e) {
      if (e.error() != 'websocket: close sent') {
        console.log('An unexpected error occured: ', e.error());
      }
    });
  });

  check(res, { 'status is 101': (r) => r && r.status === 101 });
}
```

`error` 핸들러를 등록해서 에러를 처리한다. 비슷한 방식으로 특정 주기마다 실행되는 행동(`setInterval`), 타임아웃(`setTimeout`) 등을 설정할 수 있다.