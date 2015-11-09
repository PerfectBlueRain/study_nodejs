# 비동기 프로그래밍
- http://blog.naver.com/PostView.nhn?blogId=soener&logNo=40167093336
- callback / Event리스너
- Control Flow : 프로세스의 흐름 (Control Flow)을 관리하여 가독성 높은 코드를 작성
   - http://nodeqa.com/nodejs_ref/3
- nimble / Seq
- async / promises
   - http://edoli.tistory.com/57

---

## 콜백
- 일회성 응답에 대한 로직을 정의하는 데 사용
```
// callback_example.js
var http = require('http');
var fs = require('fs');

http.createServer(function(req, res) {
  if (req.url == '/') {
    fs.readFile('./titles.json', function(err, data) {
      if (err) {
        console.error(err);
        res.end('Server Error');
      }
      else {
        var titles = JSON.parse(data.toString());

        fs.readFile('./template.html', function(err, data) {
          if (err) {
            console.error(err);
            res.end('Server Error');
          }
          else {
            var tmpl = data.toString();

            var html = tmpl.replace('%', titles.join('</li><li>'));
            res.writeHead(200, {'Content-Type': 'text/html'});
            res.end(html);
          }
        });
      }
    });
  }
}).listen(8000, "127.0.0.1");
```

```
// reducing_nesting_by_returning_early.js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
  getTitles(res);
}).listen(8000, "127.0.0.1");

function getTitles(res) {
  fs.readFile('./titles.json', function (err, data) {
    if (err) return hadError(err, res);
    getTemplate(JSON.parse(data.toString()), res);
  })
}

function getTemplate(titles, res) {
  fs.readFile('./template.html', function (err, data) {
    if (err) return hadError(err, res);
    formatHtml(titles, data.toString(), res);
  });
}

function formatHtml(titles, tmpl, res) {
  var html = tmpl.replace('%', titles.join('</li><li>'));
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end(html);
}

function hadError(err, res) {
  console.error(err);
  res.end('Server Error');
}
```


## 이벤트 리스너
- 기본적으로 개념적인 요소(이벤트)에 연관된 콜백
- events모듈
```
// pubsub.js
var events = require('events')
  , net = require('net');

var channel = new events.EventEmitter();
channel.clients = {};
channel.subscriptions = {};

channel.on('join', function(id, client) {
  this.clients[id] = client;
  this.subscriptions[id] = function(senderId, message) {
    if (id != senderId) {
      this.clients[id].write(message);
    }
  }
  this.on('broadcast', this.subscriptions[id]);
});

channel.on('leave', function(id) {
  channel.removeListener('broadcast', this.subscriptions[id]);
  channel.emit('broadcast', id, id + " has left the chat.\n");
});

channel.on('shutdown', function() {
  channel.emit('broadcast', '', "Chat has shut down.\n");
  channel.removeAllListeners('broadcast');
});

var server = net.createServer(function (client) {
  var id = client.remoteAddress + ':' + client.remotePort;
  client.on('connect', function() {
    channel.emit('join', id, client);
  });
  client.on('data', function(data) {
    data = data.toString();
    if (data == "shutdown\r\n") {
      channel.emit('shutdown');
    }
    channel.emit('broadcast', id, data);
  });
  client.on('close', function() {
    channel.emit('leave', id);
  });
});
server.listen(8888);
```




## 비동기 로직의 순서 배열
- 직렬 흐름 제어
```
// random_story.js
var fs = require('fs');
var request = require('request');
var htmlparser = require('htmlparser');
var configFilename = './rss_feeds.txt';

function checkForRSSFile () {
  fs.exists(configFilename, function(exists) {
    if (!exists)
      return next(new Error('Missing RSS file: ' + configFilename));

    next(null, configFilename);
  });
}

function readRSSFile (configFilename) {
  fs.readFile(configFilename, function(err, feedList) {
    if (err) return next(err);

    feedList = feedList
               .toString()
               .replace(/^\s+|\s+$/g, '')
               .split("\n");
    var random = Math.floor(Math.random()*feedList.length);
    next(null, feedList[random]);
  });
}

function downloadRSSFeed (feedUrl) {
  request({uri: feedUrl}, function(err, res, body) {
    if (err) return next(err);
    if (res.statusCode != 200)
      return next(new Error('Abnormal response status code'))

    next(null, body);
  });
}

function parseRSSFeed (rss) {
  var handler = new htmlparser.RssHandler();
  var parser = new htmlparser.Parser(handler);
  parser.parseComplete(rss);

  if (!handler.dom.items.length)
    return next(new Error('No RSS items found'));

  var item = handler.dom.items.shift();
  console.log(item.title);
  console.log(item.link);
}

var tasks = [ checkForRSSFile,
              readRSSFile,
              downloadRSSFeed,
              parseRSSFeed ];

function next(err, result) {
  if (err) throw err;

  var currentTask = tasks.shift();

  if (currentTask) {
    currentTask(result);
  }
}

next();
```
- 병렬 흐름 제어
```
// word_count.js
var fs = require('fs');
var completedTasks = 0;
var tasks = [];
var wordCounts = {};
var filesDir = './text';

function checkIfComplete() {
  completedTasks++;
  if (completedTasks == tasks.length) {
    for(var index in wordCounts) {
      console.log(index +': ' + wordCounts[index]);
    }
  }
}

function countWordsInText(text) {
  var words = text
    .toString()
    .toLowerCase()
    .split(/\W+/)
    .sort();
  for(var index in words) {
    var word = words[index];
    if (word) {
      wordCounts[word] = (wordCounts[word]) ? wordCounts[word] + 1 : 1;
    }
  }
}

fs.readdir(filesDir, function(err, files) {
  if (err) throw err;
  for(var index in files) {
    var task = (function(file) {
      return function() {
        fs.readFile(file, function(err, text) {
          if (err) throw err;
          countWordsInText(text);
          checkIfComplete();
        });
      }
    })(filesDir + '/' + files[index]);
    tasks.push(task);
  }
  for(var task in tasks) {
    tasks[task]();
  }
});
```

- nimble
```
var flow = require('nimble');

flow.series([
  function (callback) {
    setTimeout(function() {
      console.log('I execute first.');
      callback();
    }, 1000);
  },
  function (callback) {
    setTimeout(function() {
      console.log('I execute next.');
      callback();
    }, 500);
  },
  function (callback) {
    setTimeout(function() {
      console.log('I execute last.');
      callback();
    }, 100);
  }
]);
```
