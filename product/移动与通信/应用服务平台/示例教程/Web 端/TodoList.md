本文档介绍如何无需域名、无需服务器快速搭建一个Todo List 应用，该应用可记录您的待办事项并可为待办事项添加附件。最终成型的应用展示如下：
![](https://main.qcloudimg.com/raw/38cb96d9cf25f433a16857229a81dc84.png)

## 准备工作

1. 请 [注册腾讯云账号](https://cloud.tencent.com/register?s_url=https%3A%2F%2Fcloud.tencent.com%2F)，注册成功后即可使用腾讯云服务，已注册可忽略此步骤。
2. 登录腾讯云 [云开发控制台](https://console.cloud.tencent.com/tcb)，您在填写环境名称之后，单击**下一步**，直接提交授权并使用云开发服务。
   ![](https://main.qcloudimg.com/raw/a93584a3b38cbd035501b0f3b8ac2d56.png)
   如果您之前创建过环境，可以继续使用已创建的**按量计费环境**，或者再次**新建环境**。
   ![](https://main.qcloudimg.com/raw/68f9e9836035f548aa840ad1c2a17a77.png)
3. <span id="step1.3"></span>开通成功之后，单击环境名称，进入 [环境总览](https://console.cloud.tencent.com/tcb/env/overview) 页面，如下所示：
![](https://qcloudimg.tencent-cloud.cn/raw/a2860d0f52afff9800cb66d5a60d4d03.jpg)
> !请记住您的环境 ID，这个 ID 在后续步骤将被使用。您可单击**环境Id**右侧的<img src="https://main.qcloudimg.com/raw/a06f957521023a64e977041f9181f251.jpg"  style="margin:0;">图标进行复制。

## 步骤1：开启匿名登录

搭建 Todo List 应用，需开启**匿名登录**功能，请前往 [登录授权](https://console.cloud.tencent.com/tcb/env/login) 控制台，打开**匿名登录**开关，如下图所示：
![](https://qcloudimg.tencent-cloud.cn/raw/f83fad336b179e9457e5b05d95e0a32c.jpg)

## 步骤2：建立本地 Todo List 文件

1. 在本地新建文本文件（Mac 用户推荐使用无格式文本编辑），在文件中填入如下内容，并将第17行代码中的`${envId}`替换为 [准备工作](#step1.3) 中复制的**环境Id**：
```js
<html>
<head>
  <meta charset="UTF-8">
  <script src="https://upload-dianshi-1255598498.file.myqcloud.com/todo-698d26c543e7f35729c121106f55ef1ffaeca67d.js" charset="utf-8"></script>
  <script src="https://imgcache.qq.com/qcloud/tcbjs/1.10.8/tcb.js"></script>
</head>
<body>
  <div id="model">
    <input id="text-in" type="text" placeholder="写下您的待办事项…">
    <label id="file-in" for="file-input">上传附件</label>
    <input id="file-input" type="file" onchange="TODO.filechange(this)">
    <ul id="todo-list"></ul>
  </div>
  <script>
    let uid = null;
    const app = tcb.init({
      env: "${envId}"
    })
    const auth = app.auth({
      persistence: "local"
    });
    const db = app.database();
    window.onload = function () {
      sign();
      TODO.init();
    }
    function sign() {
      auth.anonymousAuthProvider().signIn().then(() => {
        uid = auth.hasLoginState().user.uid;
        db.collection('todo').doc(uid).get().then(res => {
          if (res.data.length == 0) {
            db.collection('todo').add({
              _id: uid,
              list: TODO.todo,
              time: new Date()
            }).then(res => {
              console.log(res);
              watchtodo();
            })
          }
          else {
            console.log(res);
            TODO.todo = res.data[0].list;
            TODO.todoinit();
            watchtodo();
          }
        });
        app.callFunction({
          name:'todo_getNumber'
        }).then(res=>{
          document.getElementById('model').innerHTML+=`<p class='bottom-des'>共${res.result}人使用云开发TODO</p>`
        })
      })
    }
    TODO.itemChange = function (id, type, des) {
      if (type === 'add') {
        if (des != null) {
          app.uploadFile({
            cloudPath: `todo/${uid}/${TODO.todo[id].file}`,
            filePath: des
          }).then((result) => {
            console.log(result)
            TODO.todo[id].file = result.fileID
            updatetodo()
          });
        } else {
          updatetodo()
        }
      } else if (type === 'delete') {
        if (TODO.todo[id].file != null) {
          app.deleteFile({
            fileList: [TODO.todo[id].file]
          }).then((result) => {
            delete TODO.todo[id]
            console.log(result)
            updatetodo()
          });
        } else {
          delete TODO.todo[id]
          updatetodo()
        }
      } else {
        updatetodo()
      }
    }
    TODO.downLoadfile = function (file) {
      app.downloadFile({
        fileID: file
      })
    }
    function updatetodo() {
      db.collection('todo').doc(uid).update({
        list: db.command.set(TODO.todo),
        time: new Date()
      }).then(res => {
      }).catch(e => {
        console.log(e);
      })
    }
    function watchtodo() {
      db.collection('todo').where({ _id: uid }).watch({
        onChange: (snapshot) => {
          if (snapshot.msgType != "INIT_EVENT") {
            TODO.todo = snapshot.docs[0].list;
            TODO.todoinit();
          }
        },
        onError: (error) => {
          alert('远端数据库监听失败！');
        }
      });
    }
  </script>
</body>
</html>
```
2. 保存文本文件，并将后缀改为 `html`，命名为 `index.html`。

## 步骤3：托管静态文件

该步骤中您将使用云开发**静态网站托管**功能。为了让更多人可以访问 Todo List 应用，云开发提供默认域名，可通过公网访问应用。

1. 登录 [云开发控制台](https://console.cloud.tencent.com/tcb/env/index)，进入在 [准备工作](#.E5.87.86.E5.A4.87.E5.B7.A5.E4.BD.9C) 中已经创建好的**按量计费**环境。
2. 进入云开发控制台的 [静态网站托管](https://console.cloud.tencent.com/tcb/hosting)，单击**上传文件**，上传 [步骤2](#.E6.AD.A5.E9.AA.A42.EF.BC.9A.E5.BB.BA.E7.AB.8B.E6.9C.AC.E5.9C.B0-todo-list-.E6.96.87.E4.BB.B6) 中的 `index.html` 文件。
![](https://qcloudimg.tencent-cloud.cn/raw/444c00ee3f854be1092efb5bed1ad952.jpg)
3. 上传完毕后，单击**配置信息**中的**默认域名**，在浏览器中打开该链接，即可在公网环境下访问 Todo List 网站。
![](https://qcloudimg.tencent-cloud.cn/raw/199ae73a33116fecad9b99452715e743.jpg)
>?默认域名可供您快速验证业务，如您需要对外正式提供网站服务，请前往**基础配置**绑定您已备案的自定义域名。

## 步骤4：创建数据库

该步骤中您将使用云开发的**数据库**功能，将 Todo List 应用内的数据存储在云数据库中。

进入云开发控制台的 [数据库](https://console.cloud.tencent.com/tcb/db) 中，单击**新建集合**，新建一个名称为`todo`的集合，如下图所示：
![](https://qcloudimg.tencent-cloud.cn/raw/f5a1b9452a70509da2b2dac442d1c3a0.jpg)

之后 Todo List 内的数据便会存储在这个集合中。

## 步骤5：创建云函数

该步骤中您将使用云开发的**云函数**功能，以统计共有多少名用户在使用 Todo List 应用。

1. 进入云开发控制台的 [云函数](https://console.cloud.tencent.com/tcb/scf)，单击**新建云函数**，开始创建云函数
	1. 在基础信息中，将函数名命名为`todo_getNumber`。
![](https://qcloudimg.tencent-cloud.cn/raw/c070c2fed288e9d6950c24bed6158e18.jpg)
	2. 在函数配置中，均选择默认配置。
![](https://qcloudimg.tencent-cloud.cn/raw/46867882b9408154c9b3bbb3fbfbe1ce.jpg)
	3. 单击**确定**，即可创建云函数。
2. 上传云函数代码包。
 1. [单击下载](https://upload-dianshi-1255598498.file.myqcloud.com/todo_getNumber-844b59708eb69a064741989333abcd8bc3a67ffa.zip) 云函数代码 ZIP 包（需要注册 Gitee 才可进行下载）。
 2. 进入创建好的 `todo_getNumber` 函数，单击**函数代码**页签，进入函数代码页面。
 3. 在**提交方法**选项中，选择**本地上传ZIP包**，并单击**上传**，将下载的云函数代码包 ZIP上传至云开发中。
 4. 单击**保存**，即可完成上传云函数代码包。
![](https://qcloudimg.tencent-cloud.cn/raw/f4507d292f8cd9abd9546663a7039037.jpg)

## 步骤6：开始使用

至此，您已经成功创建了一个 Todo List 网页应用，打开 [静态网站托管](https://console.cloud.tencent.com/tcb/hosting) 中的默认域名链接即可访问应用：
![](https://main.qcloudimg.com/raw/38cb96d9cf25f433a16857229a81dc84.png)

## 总结
该版本的 Todo List 应用为入门教学案例，通过使用云开发中的登录授权、云函数、数据库、云存储、静态托管等功能，快速搭建了一个简易 Todo List 应用并开放给用户使用。





