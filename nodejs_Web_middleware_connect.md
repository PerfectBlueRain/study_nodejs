# Web : Middleware
## Connect
- 인증 미들웨어 : authentication.js
```javascript
var connect = require('connect');

function logger(req, res, next) {
  console.log('%s %s', req.method, req.url);
  next();
}

function hello(req, res) {
  res.setHeader('Content-Type', 'text/plain');
  res.end('hello world');
}

function authenticateWithDatabase(user, pass, callback) {
  var err;
  if (user != 'tobi' || pass != 'ferret') {
    err = new Error('Unauthorized');
  }
  callback(err);
}

function restrict(req, res, next) {
  var authorization = req.headers.authorization;
  if (!authorization) return next(new Error('Unauthorized'));

  var parts = authorization.split(' ');
  var scheme = parts[0];
  var auth = new Buffer(parts[1], 'base64').toString().split(':');
  var user = auth[0];
  var pass = auth[1];

  authenticateWithDatabase(user, pass, function (err) {
    if (err) return next(err);
    next();
  });
}

function admin(req, res, next) {
  switch (req.url) {
    case '/':
      res.end('try /users');
      break;
    case '/users':
      res.setHeader('Content-Type', 'application/json');
      res.end(JSON.stringify(['tobi', 'loki', 'jane']));
      break;
  }
}

connect()
  .use(logger)
  .use('/admin', restrict)
  .use('/admin', admin)
  .use(hello)
  .listen(3000);
```

- router 미들웨어 컴포넌트
   - router_example.js
```javascript
var connect = require('connect');
var router = require('./middleware/router');
var routes = {
  GET: {
    '/users': function(req, res){
      res.end('tobi, loki, ferret');
    },
    '/user/:id': function(req, res, id){
      res.end('user ' + id);
    }
  },
  DELETE: {
    '/user/:id': function(req, res, id){
      res.end('deleted user ' + id);
    }
  }
};

connect()
  .use(router(routes))
  .listen(3000);
```
   - middleware/router.js
```javascript
var parse = require('url').parse;

module.exports = function route(obj) {
  return function(req, res, next){
    if (!obj[req.method]) {
      next();
      return;
    }
    var routes = obj[req.method];
    var url = parse(req.url);
    var paths = Object.keys(routes);

    for (var i = 0; i < paths.length; i++) {
      var path = paths[i];
      var fn = routes[path];
      path = path
        .replace(/\//g, '\\/')
        .replace(/:(\w+)/g, '([^\\/]+)');
      var re = new RegExp('^' + path + '$');
      var captures = url.pathname.match(re);
      if (captures) {
        var args = [req, res].concat(captures.slice(1));
        fn.apply(null, args);
        return;
      }
    }
    next();
  }
};
```

- Error Handling 미들웨어
```javascript
var connect = require('connect');

function badMiddleware(req, res, next) {
  next(new Error('Bad middleware makes error'));
}

function errorHandler() {
  var env = process.env.NODE_ENV || 'development';
  return function(err, req, res, next) {
    res.statusCode = 500;
    switch (env) {
      case 'development':
        res.setHeader('Content-Type', 'application/json');
        res.end(JSON.stringify(err));
        break;
      default:
        res.end('Server error');
    }
  }
}

connect()
  .use(badMiddleware)
  .use(errorHandler)
  .listen(3000);
```
