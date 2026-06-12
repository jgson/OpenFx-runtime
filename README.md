# OpenFx Runtime

OpenFx Runtime은 [OpenFx](https://github.com/keti-openfx/openfx) 서버리스 FaaS(Function as a Service) 플랫폼에서 다양한 프로그래밍 언어로 함수를 실행할 수 있도록 지원하는 런타임 컨테이너 모음입니다.

각 런타임은 `fxwatcher`와 연동되어 gRPC 기반으로 함수 요청을 처리합니다.

---

## 지원 런타임

| 언어 | 디렉토리 | 핸들러 파일 | 포트 |
|------|----------|-------------|------|
| Go | `go/` | `handler.go` | 50051, 50052 |
| Python 3 | `python3/` | `handler.py` | 50051, 50052 |
| Python 2 | `python2/` | `handler.py` | 50051, 50052 |
| Node.js | `nodejs/` | `handler.js` | 50051 |
| Java | `java/` | `Handler.java` | 50051 |
| C++ | `cpp/` | `handler.cc` | 50051, 50052 |
| C# | `csharp/` | `handler.cs` | - |
| Ruby | `ruby/` | `handler.rb` | - |

---

## 아키텍처

각 런타임은 다음 구조로 동작합니다:

```
클라이언트 요청
     │
     ▼
 fxwatcher (gRPC 서버)
     │
     ▼
 Handler 함수 (사용자 코드)
     │
     ▼
  응답 반환
```

- **fxwatcher**: 각 언어별 베이스 이미지에 포함된 gRPC 서버로, 함수 요청을 수신하고 사용자 핸들러를 호출합니다.
- **Handler**: 사용자가 작성하는 비즈니스 로직 함수입니다.
- **Mesh Call**: 함수 간 호출(Function-to-Function)을 지원하는 메시 네트워크 기능도 제공합니다.

---

## 디렉토리 구조

```
OpenFx-runtime/
├── Makefile
├── list.yml              # 런타임 목록 및 핸들러 정의
├── go/
│   ├── Dockerfile
│   └── src/
│       └── handler.go
├── python3/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── src/
│       └── handler.py
├── python2/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── src/
│       └── handler.py
├── nodejs/
│   ├── Dockerfile
│   └── src/
│       └── handler.js
├── java/
│   ├── Dockerfile
│   └── src/
│       └── Handler.java
├── cpp/
│   ├── Dockerfile
│   ├── CMakeLists.txt
│   ├── config.yaml
│   ├── cmake/
│   └── src/
│       └── handler.cc
├── csharp/
│   ├── Dockerfile
│   ├── fxServer.csproj
│   └── src/
│       └── handler.cs
└── ruby/
    ├── Dockerfile
    └── src/
        └── handler.rb
```

---

## 핸들러 작성 방법

각 언어별 핸들러는 입력을 받아 결과를 반환하는 단순한 함수 형태입니다.

### Go
```go
package main

import sdk "github.com/keti-openfx/openfx/executor/go/pb"

func Handler(req sdk.Request) string {
    return "[Go] " + string(req.Input)
}
```

### Python 3
```python
def Handler(req):
    return str.encode("[Python] ") + req.input
```

### Node.js
```javascript
function Handler(argStr) {
    return '[NodeJS] ' + argStr;
}
module.exports = Handler;
```

### Java
```java
package io.grpc.fxwatcher;

import com.google.protobuf.ByteString;

public class Handler {
    public static String reply(ByteString input) {
        return "[Java] " + input.toStringUtf8();
    }
}
```

### C++
```cpp
#include <iostream>
using namespace std;

string Handler(const string req) {
    return "[CPP] " + req;
}
```

### C#
```csharp
using System.Text;
using System;

namespace Fx {
    class Function {
        public byte[] Handler(byte[] Input) {
            string runtime = "[CSharp] ";
            byte[] strbyte = Encoding.UTF8.GetBytes(runtime);
            byte[] res = new byte[strbyte.Length + Input.Length];
            Array.Copy(strbyte, 0, res, 0, strbyte.Length);
            Array.Copy(Input, 0, res, strbyte.Length, Input.Length);
            return res;
        }
    }
}
```

### Ruby
```ruby
module FxWatcher
  def FxWatcher.Handler(argStr)
    return "[Ruby] " + argStr
  end
end
```

---

## Mesh Call (함수 간 호출)

다른 함수를 호출하는 Mesh Call 기능을 각 런타임에서 사용할 수 있습니다.

### Go 예시
```go
import mesh "github.com/keti-openfx/openfx/executor/go/mesh"

func Handler(req sdk.Request) string {
    functionName := "<FUNCTIONNAME>"
    result := mesh.MeshCall(functionName, req.Input)
    return result
}
```

### Python 예시
```python
import mesh

def Handler(req):
    functionName = "<FUNCTIONNAME>"
    result = mesh.mesh_call(functionName, req.input)
    return result
```

---

## Docker 빌드

각 런타임은 `fxwatcher` 베이스 이미지를 기반으로 빌드됩니다.

```bash
# 예시: Python3 런타임 빌드
cd python3
docker build \
  --build-arg REGISTRY=<YOUR_REGISTRY> \
  --build-arg WATCHER_VERSION=1.0 \
  -t <YOUR_REGISTRY>/python3-runtime:latest .
```

### 빌드 인자 (Build Args)

| 인자 | 기본값 | 설명 |
|------|--------|------|
| `REGISTRY` | - | 사설 Docker 레지스트리 주소 |
| `WATCHER_VERSION` | `1.0` | fxwatcher 버전 |
| `handler_file` | 언어별 상이 | 핸들러 파일명 |
| `handler_name` | `Handler` | 핸들러 함수명 |

---

## Health Check

모든 런타임 컨테이너는 헬스체크를 포함하고 있습니다:

```dockerfile
HEALTHCHECK --interval=5s CMD [ -e /tmp/.lock ] || exit 1
```

`/tmp/.lock` 파일의 존재 여부로 서비스 상태를 확인합니다.

---

## 관련 프로젝트

- [OpenFx](https://github.com/keti-openfx/openfx) - OpenFx 메인 플랫폼
- [fxwatcher](https://github.com/keti-openfx/fxwatcher) - gRPC 기반 함수 실행 에이전트

---

## 라이선스

본 프로젝트는 KETI(한국전자기술연구원)에서 개발 및 관리합니다.
