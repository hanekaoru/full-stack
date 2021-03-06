```node.js``` 的 ```fs``` 模块并没有提供一个 ```copy``` 的方法，但我们可以很容易的实现一个，比如：

```js
var source = fs.readFileSync('/path/to/source', { encoding: 'utf8' });

fs.writeFileSync('/path/to/dest', source);
```

这种方式是把文件内容全部读入内存，然后再写入文件，对于小型的文本文件，这没有多大问题，比如 ```grunt-file-copy``` 就是这样实现的

但是对于体积较大的二进制文件，比如音频、视频文件，动辄几个GB大小，如果使用这种方法，很容易使内存“爆仓”

理想的方法应该是读一部分，写一部分，不管文件有多大，只要时间允许，总会处理完成，这里就需要用到流的概念

```js
var fs = require('fs');

var readStream = fs.createReadStream('/path/to/source');
var writeStream = fs.createWriteStream('/path/to/dest');

readStream.on('data', function (chunk) {  // 当有数据流出时，写入数据
    writeStream.write(chunk);
});

readStream.on('end', function () {  // 当没有数据时，关闭数据流
    writeStream.end();
});
```

上面的写法有一些问题，如果写入的速度跟不上读取的速度，有可能导致数据丢失

正常的情况应该是，写完一段，再读取下一段，如果没有写完的话，就让读取流先暂停，等写完再继续，于是代码可以修改为：

```js
var fs = require('fs');

var readStream = fs.createReadStream('/path/to/source');
var writeStream = fs.createWriteStream('/path/to/dest');

readStream.on('data', function (chunk) {  // 当有数据流出时，写入数据
    if (writeStream.write(chunk) === false) {  // 如果没有写完，暂停读取流
        readStream.pause();
    }
});

writeStream.on('drain', function () {  // 写完后，继续读取
    readStream.resume();
});

readStream.on('end', function () {  // 当没有数据时，关闭数据流
    writeStream.end();
});
```

或者使用更直接的 ```pipe```

```js
// pipe 自动调用了 data, end 等事件
fs.createReadStream('/path/to/source').pipe(fs.createWriteStream('/path/to/dest'));
```

下面是一个更加完整的复制文件的过程

```js
var fs = require('fs'),
    path = require('path'),
    out = process.stdout;

var filePath = '/Users/chen/Movies/Game.of.Thrones.S04E07.1080p.HDTV.x264-BATV.mkv';

var readStream = fs.createReadStream(filePath);
var writeStream = fs.createWriteStream('file.mkv');

var stat = fs.statSync(filePath);
var totalSize = stat.size;

var passedLength = 0;
var lastSize = 0;

var startTime = Date.now();

readStream.on('data', function (chunk) {
    passedLength += chunk.length;
    if (writeStream.write(chunk) === false) {
        readStream.pause();
    }
});

readStream.on('end', function () {
    writeStream.end();
});

writeStream.on('drain', function () {
    readStream.resume();
});

setTimeout(function show() {

    var percent = Math.ceil((passedLength / totalSize) * 100);
    var size = Math.ceil(passedLength / 1000000);
    var diff = size - lastSize;

    lastSize = size;
    out.clearLine();
    out.cursorTo(0);

    out.write('已完成' + size + 'MB, ' + percent + '%, 速度：' + diff * 2 + 'MB/s');

    if (passedLength < totalSize) {
        setTimeout(show, 500);
    } else {
        var endTime = Date.now();
        console.log();
        console.log('共用时：' + (endTime - startTime) / 1000 + '秒。');
    }
    
}, 500);
```
