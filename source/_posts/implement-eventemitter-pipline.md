---
title: 基于 EventEmitter 的双向数据 pipeline 实现
date: 2018-04-14 16:30:00
tags:
  - tech
  - eventemitter
  - pipeline
  - nodejs
---

# 基于 EventEmitter 的双向数据 pipeline 实现

## 想法来源

## Gulp

Gulp 是前端工具链中常用的流式任务执行器，适用于许多小型库的编译打包任务。它的设计思想其实很像 Linux 命令行里面的 Pipe（管道）：

```js
gulp.src(paths.scripts.src, { sourcemaps: true })
    .pipe(babel())
    .pipe(uglify())
    .pipe(concat('main.min.js'))
    .pipe(gulp.dest(paths.scripts.dest));
```

gulp 是单向的，即对于同一个 pipeline，数据一般不能被逆向还原。

## TCP/IP stack

我们知道，计算机网络协议是**分层设计**的，每层分别为数据赋予不同的含义、完成不同的使命。源主机采用网络协议栈将原始二进制流 **层层编码（encode）** 后送往目的主机，目的主机采用同样的协议栈将数据 **层层解码（decode）** 后得到原始数据。典型的 HTTP 协议将请求数据通过 TCP/IP 协议栈自上而下编码后送出，之后自下而上解码后得到响应数据：

```
                                                      +---------+
                                                      | "Hello" |
                                                      +---------+
                                        +-------------+---------+
                                        | HTTP header | PAYLOAD |
                                        +-------------+---------+   
                           +------------+-----------------------+
                           | TCP header |         PAYLOAD       |
                           +------------+-----------------------+
              +------------+------------------------------------+
              | IP header  |              PAYLOAD               |
              +------------+------------------------------------+
+------------+--------------------------------------------------+
| Eth header |                      PAYLOAD                     |
+------------+--------------------------------------------------+
```

TCP/IP 协议栈是双向的，即对于同一套协议，数据既可以被编码也可以被解码。

那么问题来了，是否可以抽象一种轻量的 Pipeline，实现类似网络协议栈双向数据流的处理能力，并且能够让用户定制化每层的处理逻辑？

## 数据流

设计之前，先根据数据流划分功能模块，这里 `PIPE` 是数据和各个数据处理单元的调度者，`PIPE_UNIT_x` 是每层数据的处理单元，可以有多个，并且按顺序前后**串联**。

```
                +------------------ PIPE -------------------+
[RAW_DATA] <==> | [PIPE_UNIT_1] <==> ... <==> [PIPE_UNIT_2] | <==> [ENCODED_DATA]
                +-------------------------------------------+
```

用户可以实现自己的 `PIPE_UNIT` 来达到定制化处理逻辑的功能，也可以任意调换 `PIPE_UNIT` 的顺序来达到不同的处理效果。

## Pipe 设计

`Pipe` 需要提供一个数据入口来启动链式处理流程：

```js
const EventEmitter = require('events');

const PIPE_TYPE_ENCODE = 'PIPE_TYPE_ENCODE';
const PIPE_TYPE_DECODE = 'PIPE_TYPE_DECODE';

class Pipe extends EventEmitter {

  // 构造 Pipe 时，传入的处理单元数组约定为 encode 顺序
  constructor(units) {
    super();
    this._encode_units = units;
    this._decode_units = [].concat(units).reverse();
  }

  // 数据处理入口
  feed(type, data) {
    const units = type === PIPE_TYPE_ENCODE ? this._encode_units : this._decode_units;
    if (units.length < 1) {
      return;
    }
    const first = units[0];
    if (first.listenerCount(type) < 1) {
      // 构建链式响应逻辑
      const last = units.reduce((prev, next) => {
        prev.on(type, (dt) => next._write(type, dt));
        return next;
      });
      last.on(type, (dt) => {
        // 最后一个 unit 完成之后 feed 的任务就结束了
        this.emit(type, dt);
      });
    }
    // 触发处理流程
    first._write(type, data);
  }

}
```

## PipeUnit 接口设计

`PipeUnit` 需要暴露编码（encode）和解码（decode）两个接口，考虑到处理单元可能异步执行，因此使用 `async` 黑膜法：

```js
class PipeUnit extends EventEmitter {

  async _write(type, data) {
    if (type === PIPE_TYPE_ENCODE) {
      this.emit(type, await this.encode(data));
    } else {
      this.emit(type, await this.decode(data));
    }
  }

  // 编码接口
  async encode(data) {
    return data;
  }

  // 解码接口
  async decode(data) {
    return data;
  }

}
```

## 实现 PipeUnit

首先实现一个提供压缩、解压缩功能的 `PipeUnit`：

```js
const zlib = require('zlib');

class ZipPipeUnit extends PipeUnit {

  async encode(data) {
    console.log('ZipPipeUnit::encode <-', data);
    return new Promise((resolve, reject) => {
      zlib.deflate(data, (err, buffer) => {
        if (err) {
          reject(err);
        } else {
          resolve(buffer);
        }
      });
    });
  }

  async decode(data) {
    console.log('ZipPipeUnit::decode <-', data);
    return new Promise((resolve, reject) => {
      zlib.unzip(data, (err, buffer) => {
        if (err) {
          reject(err);
        } else {
          resolve(buffer);
        }
      });
    });
  }

}
```

下面再实现一个提供 `AES` 对称加解密功能的 `PipeUnit`，这次采用同步执行：

```js
const crypto = require('crypto');

class CryptoPipeUnit extends PipeUnit {

  // 编码实现
  encode(plaintext) {
    console.log('CryptoPipeUnit::encode <-', plaintext);
    const cipher = crypto.createCipher('aes192', 'a password');
    const encrypted = cipher.update(plaintext);
    return Buffer.concat([encrypted, cipher.final()]);
  }

  // 解码实现
  decode(ciphertext) {
    console.log('CryptoPipeUnit::decode <-', ciphertext);
    const decipher = crypto.createDecipher('aes192', 'a password');
    const decrypted = decipher.update(ciphertext);
    return Buffer.concat([decrypted, decipher.final()]);
  }

}
```

## 实际运行

```js
// 自由组合处理单元
const units = [
  new ZipPipeUnit(),
  new CryptoPipeUnit(),
  // new CryptoPipeUnit(), // 再来一个也可以
];

// 来一个 pipe 对象
const pipe = new Pipe(units);

pipe.on(PIPE_TYPE_ENCODE, (data) => {
  console.log('encoded:', data);
  console.log('');
  // 解码
  pipe.feed(PIPE_TYPE_DECODE, data);
});
pipe.on(PIPE_TYPE_DECODE, (data) => console.log('decoded:', data.toString()));

// 编码
pipe.feed(PIPE_TYPE_ENCODE, Buffer.from('awesome nodejs'));

// 输出如下:
// ZipPipeUnit::encode <- <Buffer 61 77 65 73 6f 6d 65 20 6e 6f 64 65 6a 73>
// CryptoPipeUnit::encode <- <Buffer 78 9c 4b 2c 4f 2d ce cf 4d 55 c8 cb 4f 49 cd 2a 06 00 2a 0c 05 95>
// encoded: <Buffer a9 61 bc 37 1a 4c 41 e8 20 63 d2 90 86 94 7b 48 98 b1 91 16 84 66 58 9b 6d 88 53 da 9b b9 18 fb>

// CryptoPipeUnit::decode <- <Buffer a9 61 bc 37 1a 4c 41 e8 20 63 d2 90 86 94 7b 48 98 b1 91 16 84 66 58 9b 6d 88 53 da 9b b9 18 fb>
// ZipPipeUnit::decode <- <Buffer 78 9c 4b 2c 4f 2d ce cf 4d 55 c8 cb 4f 49 cd 2a 06 00 2a 0c 05 95>
// decoded: awesome nodejs
```

可以看到，通过对 EventEmitter 简单的封装就可以实现双向数据 pipeline，同时支持异步单元操作。

## 性能测试

功能实现了，性能又如何呢？抛开 `PipeUnit` 的业务实现，简单分析一下链式 EventEmitter 结构的性能影响因素，理论上很大程度取决于 EventEmitter 本身的性能，`Pipe::feed` 只在第一次被调用时构建响应链，之后的调用几乎不会有性能损失。

### 测试用例

Node.js 版本如下：

```
> process.versions
{ http_parser: '2.8.0',
  node: '9.11.1',
  v8: '6.2.414.46-node.23',
  uv: '1.19.2',
  zlib: '1.2.11',
  ares: '1.13.0',
  modules: '59',
  nghttp2: '1.29.0',
  napi: '3',
  openssl: '1.0.2o',
  icu: '61.1',
  unicode: '10.0',
  cldr: '33.0',
  tz: '2018c' }
```

下面分别考察 0 ~ 30000（每次递增 1000） 个 `PipeUnit` 实例的执行时间，来评估上述设计的性能表现：

```js
const { performance } = require('perf_hooks');

const payload = Buffer.alloc(4096);

for (let i = 0; i <= 30; i++) {
  const units = Array(i * 1000).fill().map(() => new PipeUnit());

  performance.mark('A_' + i);
  {
    const pipe = new Pipe(units);
    pipe.on(PIPE_TYPE_ENCODE, (data) => {
      pipe.feed(PIPE_TYPE_DECODE, data);
    });
    pipe.on(PIPE_TYPE_DECODE, () => null);
    pipe.feed(PIPE_TYPE_ENCODE, payload);
  }
  performance.mark('B_' + i);

  performance.measure(`${units.length} units`, 'A_' + i, 'B_' + i);
}

const entries = performance.getEntriesByType('measure');
for (const { name, duration } of entries) {
  console.log(`${name}: ${duration}ms`);
}
```

执行4次，可以将结果绘制到一张图中：

![](performance.png)

可以看到每次运行的结果高度一致，由上万个 `PipeUnit` 构成的链式 EventEmitter 能够以令人满意的效率完成运行。

不过出人意料的是，在特定数量的 `PipeUnit` 上总会出现尖峰，这可能和 V8 引擎的优化机制有关，作者能力有限，感兴趣的同学可以深挖原因。
