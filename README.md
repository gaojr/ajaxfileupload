# ajaxfileupload

基于 ajaxfileupload.js 文件的增强版 ajaxfileupload.js

## 问题 & 原因 & 解决

1. 运行时报 `jQuery.handleError is not a function` 错误[^1]

    * 原因：ajaxfileupload.js 是在 jQuery 1.4.2 版本之前写的，之后的版本已经没有了 handleError 方法

    * 解决：将 1.4.2 版本中的该方法复制到 js 文件中

        ```javascript
        jQuery.extend({
            // 手动添加在 jQuery 1.4.2 之前的版本才有的 handlerError 方法
            handleError: function (s, xhr, status, e) {
                // If a local callback was specified, fire it
                if (s.error)
                    s.error.call(s.context || s, xhr, status, e);
                // Fire the global callback
                if (s.global)
                    (s.context ? jQuery(s.context) : jQuery.event).trigger("ajaxError", [xhr, s, e]);
            },
            ...
        })
        ```

2. 执行成功后，始终指向 error 方法处理，无法执行 sucess 方法

    * 原因：`dataType` 为 `json` 时，若返回 `data` 为 `json` 格式的字符串时，会出现问题

    * 解决1：将 `uploadHttpData` 方法中 if(type == "json") 里的局部变量改为方法内的变量

        ```javascript
        uploadHttpData: function (r, type) {
            var data = !type;
            ...
            if (type == "json") {
                data = r.responseText;// 去掉前面的 var
                var rx = new RegExp("<pre.*?>(.*?)</pre>", "i");
                var am = rx.exec(data);
                data = (am) ? am[1] : "";// 去掉前面的 var
                eval("data = " + data);// 返回 json 对象，注释掉可返回 json 格式的字符串
            }
            ...
            return data;
        }
        ```

    * 解决2：将上传方法 $.ajaxFileUpload() 中的 `dataType` 设置为 `text`，直接获取字符串[^2]

3. 无法带参数提交，只能上传文件

    * 原因：原作者只完成了文件提交功能……

    * 解决：修改 createUploadForm 方法及其调用位置[^1]

        ```javascript
        createUploadForm: function (id, fileElementId, data) {// 添加 data 参数
            ...
            $(oldElement).appendTo(form);

            // 增加参数的支持
            if (data) {
                for (var i in data)
                    $('<input type="hidden" name="' + i + '" value="' + data[i] + '" />').appendTo(form);
            }

            // set attributes
            ...
        },
        ...
        ajaxFileUpload: function (s) {
            ...
            if (s.data) form = jQuery.addOtherRequestsToForm(form, s.data);
            var io = jQuery.createUploadIframe(id, s.secureuri, s.data);// 添加传入参数 s.data
            var frameId = 'jUploadFrame' + id;
            ...
        }
        ```

4. 造成 input[type=file] 的 chang 事件只能触发一次[^3]

    * 原因：ajaxfileupload.js 会将原 file 元素替换成新的 file 元素，且替换时未绑定事件

    * 解决：在 createUploadForm 方法 `$(oldElement).clone()` 处添加 true 参数

        ```javascript
        createUploadForm: function (id, fileElementId, data) {
            ...
            var newElement = $(oldElement).clone(true);// true：复制元素的同时复制事件
            // newElement.val("");// 清空原有数据
            $(oldElement).attr('id', fileId);
            $(oldElement).before(newElement);
            $(oldElement).appendTo(form);
            ...
        }
        ```

## 使用方法

1. 页面部分

    通过设置 input[type=file] 的 `accept` 属性，可限制上传文件的类型[^4]，多个类型用英文逗号隔开。

    ```html
    <div class="upload_div">
        <input type="file" id="fileId" name="fileName" accept="application/vnd.ms-excel,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" />
        <label for="fileId">浏览</label>
    </div>
    ```

    **注意：input[type=file] 的 `name` 属性为必填**

2. 样式部分

    * 原因：一般来说，默认的 input[type=file] 太丑了，且不同浏览器下差距较大，因此会对其进行隐藏，使用其他标签替代
    * 问题：由于 IE 安全限制问题，没有点击到 input[type=file] 的浏览按钮就不允许上传
    * 解决方案：让 file 标签盖在替代标签上，但 file 是透明的，这样用户看到的是替代标签的外观，实际点击是 file 标签[^5]

    ```css
    input {
        position: absolute;
        width: 0px;
        opacity: 0;
    }
    lable {
        position: absolute;
        display: inline;
    }
    ```

3. 前端交互部分

    ```javascript
    function ajaxFileUpload() {
        // 封装参数
        var data = { "key1": "value1", "key2": "value2" };
        // 开始上传
        $.ajaxFileUpload({
            secureuri: false,// 是否启用安全提交，默认为 false
            type: "POST",
            url: getUrl(),
            fileElementId: "fileId",// input[type=file] 的 id
            dataType: "json",// 返回值类型
            data: data,// 添加参数，无参数时注释掉
            success: function (data, status) {
                // console.log(data);
            },
            error: function (data, status, e) {
                // console.log(data);
            }
        });
    }
    function getUrl() {
        var url = "../ajaxfileupload.do";// 后台方法的 url
        var jsessionid = getSessionId();
        if (jsessionid)
            url += ";jsessionid=" + jsessionid;
        return url;
        function getSessionId() {
            var cookie = document.cookie;
            if (cookie.length > 0) {
                c_start = cookie.indexOf("JSESSIONID=");
                if (c_start != -1) {
                    c_start += "JSESSIONID=".length;
                    c_end = cookie.indexOf(";", c_start);
                    if (c_end == -1)
                        c_end = cookie.length;
                    return unescape(cookie.substring(c_start, c_end));
                }
            }
        }
    }
    ```

4. 后端接收与返回

    ```java
    @RequestMapping(value = "ajaxfileupload.do", method = RequestMethod.POST)
    public void excelImport(HttpServletResponse response,
            @RequestParam MultipartFile fileName,
            @RequestParam(required = false) String key1,
            @RequestParam(required = false) String key2) {
        // @RequestParam(required = false) 表示该参数可以为 null
        // 参数不为 null 时：
        //     key1.euqals("value1") 结果为 true
        //     key2.euqals("value2") 结果为 true
    }
    ```

    **注意：**
    1. `fileName` 参数名称要与 html 中 input[type=file] 的 `name` 属性相同
    2. `key1`、`key2` 参数名称要与 js 中封装参数的键相同

[^1]: [ajaxfileupload.js问题汇总及解决 附修复版下载](https://blog.yadgen.com/?p=970)
[^2]: [关于AjaxFileUpload后台返回Json的处理](https://blog.csdn.net/gisredevelopment/article/details/29869109)
[^3]: [关于ajaxFileUpload造成 input[type=file] change事件只能触发一次的问题](https://blog.csdn.net/sinat_34930640/article/details/77368681)
[^4]: [MIME 参考手册](http://www.w3school.com.cn/media/media_mimeref.asp)
[^5]: [IE input file隐藏不能上传文件解决方法](http://www.qttc.net/201305334.html)