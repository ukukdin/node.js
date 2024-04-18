# Node.js - HTTPS 서버 구축하기

# 왜 굳이 HTTPS?
우리가 들어가는 대부분의 사이트는 HTTPS 프로토콜을 지원한다. HTTP 프로토콜은 프로토타입에서만 사용하고 끝내야 한다. 가장 큰 차이점은 SSL 인증서의 유무이다. SSL 인증서는 사용자가 사이트에 제공하는 정보를 암호화한다. HTTP와 HTTPS의 자세한 차이는 여기에 잘 나와있다. 이번에는 노드에서 어떻게 HTTPS 서버를 구축하는지 정리해보려 한다.

# HTTPS 서버 구축하기
인증서 선택하기 (무료/유료)
기본적으로 SSL 인증서는 유료이다. 하지만 https를 지향하는 기업에서 “https는 필수가 되어야 한다”는 목소리를 모아 Let’s Encrypt라는 무료 인증기관을 만들었다.

그러나 기본적으로 3달 주기로 인증서를 업데이트해줘야 하기 때문에 현재 도메인 서비스를 받고 있는 Namecheap에서 꽤 저렴한 SSL 인증서 제품을 팔고 있어서 이를 사서 쓰기로 했다. 1년에 4달러 정도인데, 이는 특정 한 도메인에만 사용할 수 있다. wildcard(*) 서브 도메인을 모두 인증서로 걸고 싶다면 Wildcard 제품을 별도로 구입해야 한다. (이건 비싸다)

# OpenSSL로 키 생성하기
여기에서 OpenSSL 설치 파일을 받아 설치한다. 이후 커맨드 창(터미널)을 관리자 권한으로 실행한다.

먼저 key를 생성한다.

openssl genrsa -des3 -out key.pem 2048
다음으로 csr을 생성한다.


openssl req -new -key key.pem -out csr.pem
나는 csr을 생성할 때 순서대로 이렇게 입력했고, 참고하면 된다.


Country Name (국가코드) : KR
State or Province Name (시/도) : Seoul
Locality Name (구/군) : (Enter로 넘어감)
Organization Name (회사명) : NemoBros
Organizational Unit Name (부서명) : Dev Team
Common Name (인증 받을 도메인 주소) : zini.work
Email Address : email@email.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password : (Enter로 넘어감)
An optional company name : (Enter로 넘어감)
이제 이 CSR 파일을 열어서 각자의 도메인 사이트에서 인증서 제품을 구입한 후 CSR 파일의 주석을 제외한 암호화 코드 부분을 넣으면 정상적으로 인증서 파일이 발급된다. 그리고 사이트가 본인의 것이 맞는지 검사하는 절차가 3가지정도 있는데, 마음대로 선택해서 인증을 진행한다.

(Let’s Encrypt 무료 인증서를 이용하면 스킵해도 된다.)

# Node.js HTTPS 서버 구축
이제 Node.js에서 인증서를 걸어서 https 서버를 구축해보자. http 서버만 정상적으로 구축했다면 어렵지 않다.


const https = require('https');
const options = require('./config/pem_config').options;
const httpPort = 80;
const httpsPort = 443;

// HTTPS 서버
https.createServer(options, app).listen(httpsPort, () => {
  console.log(`HTTPS: Express listening on port ${httpsPort}`);

// HTTP 서버
app.listen(httpPort, () => {
  console.log(`HTTP: Express listening on port ${httpPort}`);
});
## HTTPS 서버를 구축하는데 HTTP 서버까지 오픈하는 이유는 나중에 설명하겠다.

다음으로 options에 연결된 파일을 만드는데, pem키에 대한 config를 하드코딩 하는 것보다는 option 파일로 미리 빼서 사용하는 것이 보안적으로나 가독성으로나 낫다. 내 ./config/pem_config는 이렇게 작성되어 있는데,

const fs = require("fs");
const keys_dir = "config/secure/"; // 키 파일이 위치
const ca = fs.readFileSync(keys_dir + "ca.ca-bundle");
const key = fs.readFileSync(keys_dir + "key.pem");
const cert = fs.readFileSync(keys_dir + "cert.crt");

module.exports.options = {
  key,
  cert,
  ca,
};
이렇게 Let’s Encrypt로 발급한 인증서를 넣던지, 나처럼 기관에서 유료 인증서를 발급받고 다운로드받은 파일을 넣던지 하면 된다. 참고로 keys_dir는 키 파일이 위치한 폴더 경로인데, fs의 readFile 경로는 절대경로가 아님에 주의해서 작성하자.

# HTTPS 리다이렉션
이제 서버를 열면 HTTPS (443 포트), HTTP (80 포트) 모두에서 접속할 수 있다. 하지만 HTTPS 서버를 구축했는데 사용자가 임의로 HTTP로 프로토콜을 변경해서 들어오면 말짱도루묵이 될 것이다. 이제 HTTP 요청을 HTTPS로 리다이렉션하는 미들웨어만 작성하면 모든 작업이 끝난다.

나는 express를 class화하여 서버 인스턴스를 열고 있는데, 그렇지 않은 경우에는 앞에 this.만 지우면 된다. 모든 미들웨어의 최상단에 작성해야 한다.

this.app.use((req, res, next) => {
  if (req.secure) {
    next();
  } else {
    const to = `https://${req.hostname}${req.url}`;
    res.redirect(to);
  }
});
req.secure은 https 요청인지 아닌지를 반환하기에 이를 이용하여 https 리다이렉션을 구현할 수 있다. (구현하는 방법은 이것 말고도 많다)

우여곡절의 HTTPS 서버 구축이 끝났다. 사이트에 자물쇠 모양이 뜨는 것만으로도 인증서 4천원은 아까워하지 않기로 했다.
