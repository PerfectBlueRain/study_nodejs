## node.js & Express install

0. 확인
```
$> node --version
$> git --version
```

1. node.js 다운 : https://nodejs.org/en/download/releases/
```
$> wget http://nodejs.org/dist/v0.10.22/node-v0.10.22.tar.gz
$> tar zxvf
```

2. 빌드 : node-v0...디렉터리에서
```
$> ./configure
$> make
$> make install
```

3. express설치
```
$> npm install -g express            //
$> express {프로젝트네임}               // Create the app
$> npm install                       // Install dependencies
```
4. 실행
```
$> npm start                         // Start the server
```
http://localhost:3000/

## npm

curl -L http://npmjs.org/install.sh | sh
