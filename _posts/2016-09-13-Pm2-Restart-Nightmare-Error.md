---
layout: post
title: 关于PM2 restart nightmare错误
description: "Pm2 restart nightmare"
tags: [Node]
comments: true
share: true
---

线上服务nightmare使用pm2重启经常出现进程error，而使用stop && start的操作却可以正常重启，
希望从PM2的源码追踪问题原因。

<!--more-->

1.在 bin/pm2.js 中 restart 操作支持传入与进程相关的信息然后又CLI中restart进程。

<pre><code>
commander.command('restart <id|name|all|json|stdin...>')
  .option('--watch', 'Toggle watching folder for changes')
  .description('restart a process')
  .action(function(param) {
    // Commander.js patch 从param析取参数
    param = patchCommanderArg(param);

    async.forEachLimit(param, 1, function(script, next) {
      //调用CLI的restart方法，传入commander做回调
      CLI.restart(script, commander, next);

    }, function(err) {
      CLI.speedList(err ? 1 : 0);
    });
  });
</code></pre>

2.在 lib/CLI.js 中的restart操作

<pre><code>
CLI.restart = function(cmd, opts, cb) {
  //转存回调
  if (typeof(opts) == "function") {
    cb = opts;
    opts = {};
  }

  if (typeof(cmd) === 'number')
    cmd = cmd.toString();
  ....
  //调用restart
  CLI._restart(cmd, opts, cb);

};

CLI._restart = function(cmd, envs, cb) {
  CLI._operate('restartProcessId', cmd, envs, cb);
};

CLI._operate = function(action_name, process_name, envs, cb) {
  ........
  processIds([process_name], cb);

  function processIds(ids, cb) {
    async.eachLimit(ids, cst.CONCURRENT_ACTIONS, function(id, next) {
      var opts = id;
      if (action_name == 'restartProcessId') {
      	//运行环境拼装
        opts = {
          id  : id,
          env : util._extend(process.env, envs)
        };
      }
      //脚本都在这里面跑了
      Satan.executeRemote(action_name, opts, function(err, res) {
		.....
		//重启操作这么跑
        if (action_name == 'restartProcessId') {
          Satan.notifyGod('restart', id);
        } 
        .......
        .......
      });
    }, function(err) {....});
  }
}
</code></pre>

2.在 lib/Satan.js 中的 executeRemote、notifyGod 操作

<pre><code>
Satan.executeRemote = function executeRemote(method, env, fn) {
  ..........
  //最终到Satan中执行进程脚本
  return Satan.client.call(method, env, fn);
};

Satan.notifyGod = function(action_name, id, cb) {
  Satan.executeRemote('notifyByProcessId', {
    id : id,
    action_name : action_name, //'restartProcessId'
    manually : true
  }, function() {
    debug('God notified');
    return cb ? cb() : false;
  });
};
</code></pre>

3.pm2-axon-rpc/lib/client.js 中的实现

<pre><code>
var axon     = require('pm2-axon');
var rpc      = require('pm2-axon-rpc');
var req      = axon.socket('req');
Satan.client = new rpc.Client(req);

Client.prototype.call = function(name){
  var args = [].slice.call(arguments, 1, -1);
  var fn = arguments[arguments.length - 1];

  //this.sock 为req —— axon.socket('req')
  this.sock.send({
    type: 'call',
    method: name, //'restartProcessId'
    args: args
  }, function(msg){
    if ('error' in msg) {
      var err = new Error(msg.error);
      err.stack = msg.stack || err.stack;
      fn(err);
    } else {
      msg.args.unshift(null);
      fn.apply(null, msg.args);
    }
  });
};
</code></pre>


