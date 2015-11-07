## Web
- basic HTTP server
```
var http = require('http');  // HTTP server
var server = http.createServer();

server.on('request', function(res, req){
   res.writeHead(200, {'Content-Type' : 'text/plain'});
   res.end('Hello world\n');
})

server.listen(3000); //port
console.log('Server running at http://localhost:3000/');
```

- Streaming Data
```
var http = require('http');  // HTTP server
var fs = require('fs'); // Streaming Data

var server = http.createServer();
server.on('request', function(res, req){
   res.writeHead(200, {'Content-Type' : 'image/png'});
   fs.createReadStream('./image.png').pipe(res); // read stream -> write stream으로 파이프연결
})

server.listen(3000); //port
console.log('Server running at http://localhost:3000/');
```
