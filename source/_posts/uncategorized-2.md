---
title: 阿里云视频点播sdk扩展
date: 2021-12-22 22:00:00
---
SDK：Javascript SDK   version-1.5.2
[阿里云视频点播Javascript SDK文档](https://help.aliyun.com/document_detail/52204.html)

问题：该版本不支持指定单个文件暂停上传 or 指定目标文件开始上传。sdk提供的几个操作单文件的api支持不到这么细节的操作，deleteFile/cancelFile/resumeFile

源码解析：
> 提取sdk代码中的一些关键枚举值
```javascript
const VODSTATE = {
  INIT: "Init",
  START: "Start",
  STOP: "Stop",
  FAILURE: "Failure",
  EXPIRE: "Expire",
  END: "End",
};
const UPLOADSTATE = {
  INIT: "Ready",
  UPLOADING: "Uploading",
  SUCCESS: "Success",
  FAIlURE: "Failure",
  CANCELED: "Canceled",
  STOPED: "Stoped",
};

```
> uploader.cancelFile(index)  取消单个文件上传
```javascript
{
  key: "cancelFile",
  value: function (e) {
    this.options;
    if (e < 0 || e >= this._uploadList.length) return !1;
    var t = this._uploadList[e];
    // 判断索引是否和当前操作的上传文件一致 & 索引对应文件的上传状态是否是上传中
    if (
      e == this._curIndex &&
      t.state == a.UPLOADSTATE.UPLOADING
    ) {
      t.state = a.UPLOADSTATE.CANCELED;
      // 获取已成功上传的分片数据
      var n = this._getCheckoutpoint(t);
      n && n.checkpoint && (n = n.checkpoint),
      // 分片数据存在则停止停止上传分片
        n && this._ossUpload.abort(t),
      // 清楚分片数据缓存
        this._removeCheckoutpoint(t),
      // 继续开始下一个上传任务
        this.nextUpload();
    } else
      // 上传状态不等于SUCCESS则直接修改为CANCELED
      t.state != a.UPLOADSTATE.SUCCESS &&
        (t.state = a.UPLOADSTATE.CANCELED);
    return this._reportLog("20008", t), !0;
  }
}
```
abort函数中调用了abortMultipartUpload方法
[AbortMultipartUpload API文档](https://help.aliyun.com/document_detail/31996.html)，调用这个相当于删除了已上传的分片数据并取消分片上传事件
说明这个方法不支持暂停上传，在这里我遇到个问题，恢复该文件上传后，异常提示报错：The specified upload does not exist. The upload ID may be invalid, or the upload may have been aborted or completed.
这个报错很好理解，结合代码调用，是因为该upload事件been aborted了
> 结论：cancelFile 是中断上传任务且无法恢复，不能用做暂停处理
```javascript
{
  key: "abort",
  value: function (e) {
    if (e.checkpoint) {
      var t = e.checkpoint.uploadId;
      this.oss.abortMultipartUpload(e.object, t);
    }
  },
}
```
> uploader.deleteFile(index); 删除上传文件
> 调用了cancelFile后从上传文件列表中删除该文件
```javascript
{
  key: "deleteFile",
  value: function (e) {
    return (
      !!this.cancelFile(e) && (this._uploadList.splice(e, 1), !0)
    );
  }
}
```
> uploader.resumeFile(index);  恢复单个文件上传        
```javascript
{
  key: "resumeFile",
  value: function (e) {
    this.options;
    if (e < 0 || e >= this._uploadList.length) return !1;
    var t = this._uploadList[e];
    return (
      t.state == a.UPLOADSTATE.CANCELED &&
      ((t.state = a.UPLOADSTATE.INIT), !0)
    );
  }
}
```
从代码中看只是简单的判断了下状态是否为CANCELED & 修改状态为INIT，没有其他处理，如何恢复上传呢
官方文档给的实例，在onUploadstarted回调事件中通过setUploadAuthAndAddress方法开始上传，其中调用了_uploadCore函数,
_uploadCore中创建了阿里云对象存储oss实例，并执行this._ossUpload.upload(e, u) 真正开始上传。其中的_complete回调中执行了_complete方法，表示上传完成后执行complete回调
```javascript
/* _uploadCore */
{
  key: "_uploadCore",
  value: function (e) {
    arguments.length > 1 &&
      void 0 !== arguments[1] &&
      arguments[1];
    if (
      !this._ossCreditor.accessKeyId ||
      !this._ossCreditor.accessKeySecret ||
      !this._ossCreditor.securityToken
    )
      throw new Error(
        "AccessKeyId、AccessKeySecret、securityToken should not be null"
      );
    if (((e.state = a.UPLOADSTATE.UPLOADING), !this._ossUpload)) {
      e.endpoint =
        e.endpoint || "http://oss-cn-hangzhou.aliyuncs.com";
      var t = this;
      this._ossUpload = new c.default(
        {
          bucket: e.bucket,
          endpoint: e.endpoint,
          AccessKeyId: this._ossCreditor.accessKeyId,
          AccessKeySecret: this._ossCreditor.accessKeySecret,
          SecurityToken: this._ossCreditor.securityToken,
          timeout: this.options.timeout,
          cname: this.options.cname,
        },
        {
          onerror: function (e, n) {
            t._error.call(t, e, n);
          },
          oncomplete: function (e, n) {
            t._complete.call(t, e, n);
          },
          onprogress: function (e, n, r) {
            t._progress.call(t, e, n, r);
          },
        }
      );
    }
    var n = y.default.getFileType(e.file.name),
      r = this._getCheckoutpoint(e),
      o = "",
      i = "";
    r &&
      r.checkpoint &&
      ((i = r.state), (o = r.videoId), (r = r.checkpoint)),
      r &&
        o == e.videoId &&
        i != a.UPLOADSTATE.UPLOADING &&
        ((r.file = e.file), (e.checkpoint = r), r.uploadId);
    var s = this._adjustPartSize(e);
    this._reportLog("20002", e, {
      ft: n,
      fs: e.file.size,
      bu: e.bucket,
      ok: e.object,
      vid: e.videoId || "",
      fn: e.file.name,
      fw: null,
      fh: null,
      ps: s,
    });
    var u = {
      headers: {
        "x-oss-notification": e.userData ? e.userData : "",
      },
      partSize: s,
      parallel: this.options.parallel,
    };
    this._ossUpload.upload(e, u);
  },
}
/* _complete */
{
  key: "_complete",
  value: function (e, t) {
    if (
      ((e.state = a.UPLOADSTATE.SUCCESS),
      this.options.onUploadSucceed)
    )
      try {
        this.options.onUploadSucceed(e);
      } catch (e) {
        console.log(e);
      }
    var n = 0;
    t &&
      t.res &&
      t.res.headers &&
      (n = t.res.headers["x-oss-request-id"]),
      this._removeCheckoutpoint(e);
    var r = this;
    setTimeout(function () {
      r.nextUpload();
    }, 100),
      (this._retryCount = 0),
      this._reportLog("20003", e, { requestId: n });
  },
}
```
注意到_complete中干了什么
1. 抛出onUploadSucceed回调到用户层，通知该文件上传成功
2. 移除本地保存分片数据
3. 100ms后执行nextUpload
```javascript
/* nextUpload */
{
  key: "nextUpload",
  value: function () {
    var e = this.options;
    if (this._state == a.VODSTATE.START)
      if (
        ((this._curIndex = this._findUploadIndex()),
        -1 != this._curIndex)
      ) {
        var t = this._uploadList[this._curIndex];
        (this._ossUpload = null), this._upload(t);
      } else {
        this._state = a.VODSTATE.END;
        try {
          e.onUploadEnd && e.onUploadEnd(t);
        } catch (e) {
          console.log(e);
        }
      }
  },
}
/* _findUploadIndex */
{
  key: "_findUploadIndex",
  value: function () {
    for (var e = -1, t = 0; t < this._uploadList.length; t++)
      if (this._uploadList[t].state == a.UPLOADSTATE.INIT) {
        e = t;
        break;
      }
    return e;
  },
}
```
nextUpload中执行逻辑
1. 判断uploader上传器状态是否为VODSTATE.START
2.查找上传文件（_findUploadIndex逻辑，查找第一个上传状态为INIT的文件）
3. 上一步中查找到文件，则开始上传该文件
4. 否则说明所有文件上传结束，标记uploader状态为VODSTATE.END & 抛出onUploadEnd回调

到这里就好理解了，为何resumeFile只是更改状态没有做其他操作，因为当有某个文件结束上传任务时会自动运行下一个INIT状态的上传任务

看到这里大概能理解为什么官方文档中推荐不要使用批量上传了，如果单个文件调用cancelFile后再调用resumeFile是不可能恢复 上传的，因为不会有其他文件上传结束触发nextUpload，而且cancelFile是取消上传job，resumeFile再恢复必报错。这里就比较疑惑了，因为resumeFile中判断CANCELED状态说明必然是跟在cancelFile后使用的


> 为了实现业务逻辑，扩展了几个方法实现
```javascript
if (AliyunUpload.Vod) {
  const VODSTATE = {
    INIT: 'Init',
    START: 'Start',
    STOP: 'Stop',
    FAILURE: 'Failure',
    EXPIRE: 'Expire',
    END: 'End'
  }
  const UPLOADSTATE = {
    INIT: 'Ready',
    UPLOADING: 'Uploading',
    SUCCESS: 'Success',
    FAIlURE: 'Failure',
    CANCELED: 'Canceled',
    STOPED: 'Stoped'
  }
  /**
   * 重启停止的文件上传
   */
  AliyunUpload.Vod.prototype.restartFile = function(index) {
    this._retryCount = 0
    this.options
    if (this._state == VODSTATE.START || this._state == VODSTATE.EXPIRE) {
      console.warn('already started or expired')
      return
    }
    if (index < 0 || index >= this._uploadList.length) {
      return
    }
    var t = this._uploadList[index]
    this._ossUpload = null
    this._initState()
    this._upload(t)
    this._state = VODSTATE.START
  }
  /**
   * 从索引位置开始查找待上传的文件索引
   * 如果不存在则从首个位置到索引位置中查找
   */
  AliyunUpload.Vod.prototype.findIndexOfWaiting = function(index = 0) {
    if (index < 0 || index + 1 >= this._uploadList.length) {
      return -1
    }
    let indexList = []
    for (let i = 0; i < this._uploadList.length; i++) {
      const item = this._uploadList[i]
      if (item.state === UPLOADSTATE.INIT) {
        if (i < index) indexList.push(i)
        if (i > index) {
          return i
        }
      }
    }
    return indexList.length > 0 ? indexList[0] : -1
  }
}
```
简单的使用场景流程DEMO，这里还有个问题在于，stopUpload会更改文件状态为STOP，自动开始下一个任务时不会执行STOP状态的任务，需要手动开始，不过问题不大，所以不做更多改造了
```javascript
/** 暂停单个文件上传 */
pause(index) {
  if(this.uploader == null) return 
  this.uploader.stopUpload()
  // 获取上传文件列表
  const uploadFiles = this.uploader.listFiles()
  if (index < 0 || index > uploadFiles.length) return
  // 获取下一个可执行上传的上传文件
  const waitingIndex = this.uploader.findIndexOfWaiting(index)
  // 自动开始下一个上传任务
  if (waitingIndex >= 0) {
    this.uploader.restartFile(waitingIndex)
  }
},
/** 删除单个上传文件 */
del(index) {
  if(this.uploader == null) return 
  this.uploader.deleteFile(index)
},
/** 恢复单个文件上传 */
resume(index) {
  this.uploader.restartFile(index)
},
```

stopUpload配合其他几个扩展方法为何能实现单个文件上传的暂停/恢复就不具体做源码解析了，说明下大概流程
1. stopUpload => 停止当前正常执行的任务，执行_changeState方法，执行ossUpload.cancel取消上传（和abort不同，可以理解为暂时停止了当前的oss上传）
2. _changeState中获取了当前已成功上传的分片数据，并保存到localStorage中
3. 这些保存的分片数据会在开始上传的时候回传到oss，具体逻辑可以参考[阿里云视频点播-断点续传](https://help.aliyun.com/document_detail/52204.html?spm=a2c4g.11186623.0.0.131d6515sHP4SQ#title-235-ywd-zkh)，[oss-分片上传和断点续传](https://help.aliyun.com/document_detail/31850.html?spm=5176.21213303.J_6028563670.19.3bde3edavQ9Y21&scm=20140722.S_help%40%40%E6%96%87%E6%A1%A3%40%4031850.S_hot.ID_31850-RL_%E6%96%AD%E7%82%B9-OR_s%2Bhelpmain-V_1-P0_2)
