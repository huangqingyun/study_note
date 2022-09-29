---
title: HttpClient
date: 2022-09-14 16:17:49
tags: [".net","C#","HttpClient"]
---

## HttpClient

用于发送 HTTP 请求并从 URI 标识的资源接收 HTTP 响应。

> [官方源码地址](https://source.dot.net/#System.Net.Http/System/Net/Http/HttpClient.cs)
>
> [官方文档](https://docs.microsoft.com/zh-cn/dotnet/api/system.net.http.httpclient?view=net-6.0)

### 使用准则

`System.Net.Http.HttpClient`类用于发送`HTTP`请求以及从`URI`所标识的资源接收`HTTP`响应。`HttpClient`实例是应用于该实例执行的所有请求的设置集合，每个实例使用自身的连接池，该池将其请求与其他请求隔离开来。 从`.NET Core 2.1`开始，`SocketsHttpHandler`类提供实现，使行为在所有平台上保持一致。

**注意事项**：

- `HttpClient`仅在创建连接时解析`DNS`条目
- 若要解决此问题，可以通过设置`PooledConnectionLifetime`属性来限制连接的生存期，以便在替换连接时要求执行`DNS`查找

 ```csharp
 var handler = new SocketsHttpHandler
 {
     PooledConnectionLifetime = TimeSpan.FromMinutes(15) // Recreate every 15 minutes
 };
 var client = new HttpClient(handler);
 ```

> [详细请看官方说明](https://docs.microsoft.com/zh-cn/dotnet/fundamentals/networking/http/httpclient-guidelines)

### Get请求

```csharp
var response = await client.GetAsync("http://www.contoso.com/");
response.EnsureSuccessStatusCode();
string responseBody = await response.Content.ReadAsStringAsync();
```

### Post请求

`Post`请求涉及到多种请求体媒体类型（`MIME`）,每种`MIME`类型都有对应的`Content-Type`。所以需要针对不同的`Content-Type`进行说明。

#### **application/x-www-form-urlencoded**

这应该是最常见的`POST`提交数据的方式了。浏览器的原生`form`表单，如果不设置`enctype`属性，那么最终就会以`application/x-www-form-urlencoded`方式提交数据。类似于下面这样请求报文（无关的请求头在本文中都省略掉了）：

```http
POST http://www.example.com HTTP/1.1
Content-Type: application/x-www-form-urlencoded;charset=utf-8
 
title=test&sub%5B%5D=1&sub%5B%5D=2&sub%5B%5D=3
```

在.net里对应的封装`application/x-www-form-urlencoded`请求内容类型的类为：[FormUrlEncodedContent](https://docs.microsoft.com/zh-cn/dotnet/api/system.net.http.formurlencodedcontent?view=net-6.0).

##### `FormUrlEncodedContent`示例

```csharp
var param = new Dictionary<string, string> 
{
    {"Id", "10" },
    {"Name", "Tom Post"},
    {"Age", "10"},
    {"Birthday", DateTime.Now.ToString("yyyyMMdd")}
};
using var content = new FormUrlEncodedContent(param);
var response = await client.PostAsync("http://www.contoso.com/",content);
response.EnsureSuccessStatusCode();
string responseBody = await response.Content.ReadAsStringAsync();

// 自定义请求,特别要注意URL编码
var requestStr = UrlEncoder.Default.Encode("Id=10&Name=Tom Post&Age=10&Birthday=20080520");
using var content = new StringContent(requestStr);
var mediaType = new MediaTypeHeaderValue("application/x-www-form-urlencoded"); // 添加内容头类型
mediaType.CharSet = "utf-8"; // 设置字符编码类型，和第一行对应
content.Headers.ContentType = mediaType; // 添加内容头类型
var response = await client.PostAsync("http://www.contoso.com/",content);
response.EnsureSuccessStatusCode();
string responseBody = await response.Content.ReadAsStringAsync();
```

在使用该类型进行自定义请求时，特别容易**忘记**将请求的数据**进行URL编码**。如果忘记URL解码情况下,还对请求数据`Base64`编码进行请求，服务端在收到请求多数据再进行`URL`解码时，会把`Base64`中 `+` 解码为空格。导致`Base64`解码失败。

> URL编码(URL encoding)适用于统一资源标识符(URI)的编码，也用于为"application/x-www-form-urlencoded"`MIME`数据.

#### **multipart/form-data**

在文件上传场景，是最常见的`POST`提交数据的方式了。浏览器的原生`form`表单，设置`enctype`属性值为`multipart/form-data`，那么最终就会以`multipart/form-data`方式提交数据。类似于下面这样请求报文（无关的请求头在本文中都省略掉了）：

```http
POST http://www.example.com HTTP/1.1
Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryrGKCBY7qhFd3TrwA
 
------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="text"
 
title
------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="file"; filename="chrome.png"
Content-Type: image/png
 
PNG ... content of chrome.png ...
------WebKitFormBoundaryrGKCBY7qhFd3TrwA--
```

这个例子稍微复杂点。首先生成了一个`boundary`用于分割不同的字段，为了避免与正文内容重复，`boundary`很长很复杂。然后 `Content-Type` 里指明了数据是以`mutipart/form-data`来编码，本次请求的`boundary`是什么内容。消息主体里按照字段个数又分为多个结构类似的部分，每部分都是以 `--boundary` 开始，紧接着内容描述信息，然后是回车，最后是字段具体内容（文本或二进制）。如果传输的是文件，还要包含文件名和文件类型信息。消息主体最后以`--boundary--`标示结束。

> 关于 `mutipart/form-data` 的详细定义，请前往 [rfc1867](http://www.ietf.org/rfc/rfc1867.txt) 查看。

在.net里对应的封装`mutipart/form-data`请求内容类型的类为：[HttpMultipartFormDataContent](https://docs.microsoft.com/zh-cn/dotnet/api/system.net.http.multipartformdatacontent?view=net-6.0).

##### `HttpMultipartFormDataContent`示例

```csharp
using var content = new MultipartFormDataContent(){
     { new StringContent("12"),  "Id"},
     { new StringContent("zhangs"),  "Name"},
};
var byteArrayContent = new ByteArrayContent(File.ReadAllBytes("your file path"));
byteArrayContent.Headers.ContentType = new MediaTypeHeaderValue("audio/wav");// 这里为文件MIME类型
content.Add(byteArrayContent, Constants.DefaultFormFields.File, filename);
content.Add(new StringContent("false"), Constants.DefaultFormFields.EnableWords);

HttpResponseMessage response = await _client.PostAsync("http://www.samples.com/", content, cancellationToken).ConfigureAwait(false);
response.EnsureSuccessStatusCode();
string responseBody = await response.Content.ReadAsStringAsync();
```

#### **application/json**

`RESTful`的接口最常用的提交数据的方式了。`application/json`方式提交数据。类似于下面这样请求报文（无关的请求头在本文中都省略掉了）：

```http
POST http://www.example.com HTTP/1.1
Content-Type: application/json;charset=utf-8
 
{"title":"test","sub":[1,2,3]}
```

在.net里对应的封装`application/json`请求内容类型的类为：[JsonContent](https://docs.microsoft.com/zh-cn/dotnet/api/system.net.http.json.jsoncontent?view=net-6.0).

##### `JsonContent`示例

```csharp
var user = new User { Name = "Steve Gordon" };
// 方式1
var content = new StringContent(JsonSerializer.Serialize(user));
content.Headers.ContentType = new MediaTypeHeaderValue("application/json");
var response = await client.PostAsync("http://www.samples.com/", content).ConfigureAwait(false);
response.EnsureSuccessStatusCode();

// 方式2
var postRequest = new HttpRequestMessage(HttpMethod.Post, uri)
{
    Content = JsonContent.Create(json);
};
var response = await httpClient.SendAsync(postRequest).ConfigureAwait(false);
response.EnsureSuccessStatusCode();

// 方式3
var response = await httpClient.PostAsJsonAsync(uri, json).ConfigureAwait(false);
response.EnsureSuccessStatusCode();
```

#### **text/xml**

`XML`作为编码方式的远程调用规范。典型的`XML-RPC`请求是这样的：

```http
POST http://www.example.com HTTP/1.1
Content-Type: text/xml
 
<!--?xml version="1.0"?-->
<methodcall>
<methodname>examples.getStateName</methodname>
  <params>
    <param>
      <value><i4>41</i4></value>
    </param>
  </params>
</methodcall>
```

这个现在不是很常用，至少目前我还没遇到过这样的。所以不再进行说明。

### **Url**拼接注意事项

在使用命名客户端等指定`BaseAddress`属性的情况下，特别要注意`Path`是相对路径还是绝对路径。

以下是`HttpClient`解析请求信息源码

```csharp
private void PrepareRequestMessage(HttpRequestMessage request)
{
    Uri? requestUri = null;
    if ((request.RequestUri == null) && (_baseAddress == null))
    {
        throw new InvalidOperationException(SR.net_http_client_invalid_requesturi);
    }
    if (request.RequestUri == null)
    {
        requestUri = _baseAddress;
    }
    else
    {
        // If the request Uri is an absolute Uri, just use it. Otherwise try to combine it with the base Uri.
        if (!request.RequestUri.IsAbsoluteUri)
        {
            if (_baseAddress == null)
            {
                throw new InvalidOperationException(SR.net_http_client_invalid_requesturi);
            }
            else
            {
                requestUri = new Uri(_baseAddress, request.RequestUri);
            }
        }
    }

    // We modified the original request Uri. Assign the new Uri to the request message.
    if (requestUri != null)
    {
        request.RequestUri = requestUri;
    }

    // Add default headers
    if (_defaultRequestHeaders != null)
    {
        request.Headers.AddHeaders(_defaultRequestHeaders);
    }
}
```

从源码中可知，在解析`HttpRequestMessage`时，其属性`RequestUri`不为空且是绝对路径时，会覆盖`HttpClient`的`BaseAddress`属性值。如果`HttpRequestMessage`的属性`RequestUri`不为空且是相对路径时，会将`HttpClient`的`BaseAddress`属性值和`HttpRequestMessage`的属性`RequestUri`值进行合并。所以要特别注意`Uri`合并规则。

**简单总结`Uri`网络路径的合并规则**：

`basePart`指代`HttpClient`的`BaseAddress`属性值；`relativePart`指代`HttpRequestMessage`的属性`RequestUri`。

- `relativePart`是以`//`或`\`开始的，为绝对路径，否则为相对路径。

 1. 绝对路径，将`basePart`端口号及其之前的部分和`relativePart`拼接
 2. 相对路径，将`basePart`直接和`relativePart`拼接

- `basePart`是以`\`结尾的，拼接时将取`basePart`最后的一个`\`之前的部分和`relativePart`直接拼接，若没有`\`结尾则取端口号及其之前的部分和`relativePart`拼接。

验证规则代码：

```csharp
var baseUrl = "https://user:password@www.contoso.com:80";
// var baseUrl = "https://user:password@www.contoso.com:80/v2/v1";
// var baseUrl = "https://user:password@www.contoso.com:80/v2";
var path = "xxxxx?a=q&b=p";
// var path = "/xxxxx?a=q&b=p";
var baseUri = new Uri(baseUrl);
var uri = new Uri(baseUri, path);
Console.WriteLine($"AbsolutePath: {uri.AbsolutePath}");
Console.WriteLine($"AbsoluteUri: {uri.AbsoluteUri}");

// 输出信息如下:
//  AbsolutePath: /xxxxx
//  AbsoluteUri: https://user:password@www.contoso.com:80/xxxxx?a=q&b=p
```

以下为`Uri`合并源码：

```csharp
private static string CombineUri(Uri basePart, string relativePart, UriFormat uriFormat)
{
    //NB: relativePart is ensured as not empty by the caller
    //    Another assumption is that basePart is an AbsoluteUri

    // This method was not optimized for efficiency
    // Means a relative Uri ctor may be relatively slow plus it increases the footprint of the baseUri

    char c1 = relativePart[0];

    //check a special case for the base as DOS path and a rooted relative string
    if (basePart.IsDosPath &&
        (c1 == '/' || c1 == '\\') &&
        (relativePart.Length == 1 || (relativePart[1] != '/' && relativePart[1] != '\\')))
    {
        // take relative part appended to the base string after the drive letter
        int idx = basePart.OriginalString.IndexOf(':');
        if (basePart.IsImplicitFile)
        {
            return string.Concat(basePart.OriginalString.AsSpan(0, idx + 1), relativePart);
        }

        // The basePart has explicit scheme (could be not file:), take the DOS drive ':' position
        idx = basePart.OriginalString.IndexOf(':', idx + 1);
        return string.Concat(basePart.OriginalString.AsSpan(0, idx + 1), relativePart);
    }

    // Check special case for Unc or absolute path in relativePart when base is FILE
    if (StaticIsFile(basePart.Syntax))
    {
        if (c1 == '\\' || c1 == '/')
        {
            if (relativePart.Length >= 2 && (relativePart[1] == '\\' || relativePart[1] == '/'))
            {
                //Assuming relative is a Unc path and base is a file uri.
                return basePart.IsImplicitFile ? relativePart : "file:" + relativePart;
            }

            // here we got an absolute path in relativePart,
            // For compatibility with V1.0 parser we restrict the compression scope to Unc Share, i.e. \\host\share\
            if (basePart.IsUnc)
            {
                ReadOnlySpan<char> share = basePart.GetParts(UriComponents.Path | UriComponents.KeepDelimiter, UriFormat.Unescaped);
                int i = share.Slice(1).IndexOf('/');
                if (i >= 0)
                {
                    share = share.Slice(0, i + 1);
                }

                if (basePart.IsImplicitFile)
                {
                    return string.Concat(@"\\", basePart.GetParts(UriComponents.Host, UriFormat.Unescaped), share, relativePart);
                }

                return string.Concat("file://", basePart.GetParts(UriComponents.Host, uriFormat), share, relativePart);
            }
            // It's not obvious but we've checked (for this relativePart format) that baseUti is nor UNC nor DOS path
            //
            // Means base is a Unix style path and, btw, IsImplicitFile cannot be the case either
            return "file://" + relativePart;
        }
    }

    // If we are here we did not recognize absolute DOS/UNC path for a file: base uri
    // Note that DOS path may still happen in the relativePart and if so it may override the base uri scheme.

    bool convBackSlashes = basePart.Syntax.InFact(UriSyntaxFlags.ConvertPathSlashes);

    string? left;

    // check for network or local absolute path
    if (c1 == '/' || (c1 == '\\' && convBackSlashes))
    {
        if (relativePart.Length >= 2 && relativePart[1] == '/')
        {
            // got an authority in relative path and the base scheme is not file (checked)
            return basePart.Scheme + ':' + relativePart;
        }

        // Got absolute relative path, and the base is not FILE nor a DOS path (checked at the method start)
        if (basePart.HostType == Flags.IPv6HostType)
        {
            left = $"{basePart.GetParts(UriComponents.Scheme | UriComponents.UserInfo, uriFormat)}[{basePart.DnsSafeHost}]{basePart.GetParts(UriComponents.KeepDelimiter | UriComponents.Port, uriFormat)}";
        }
        else
        {
            left = basePart.GetParts(UriComponents.SchemeAndServer | UriComponents.UserInfo, uriFormat);
        }

        return convBackSlashes && c1 == '\\' ?
            string.Concat(left, "/", relativePart.AsSpan(1)) :
            left + relativePart;
    }

    // Here we got a relative path
    // Need to run path Compression because this is how relative Uri combining works

    // Take the base part path up to and including the last slash
    left = basePart.GetParts(UriComponents.Path | UriComponents.KeepDelimiter,
        basePart.IsImplicitFile ? UriFormat.Unescaped : uriFormat);
    int length = left.Length;
    char[] path = new char[length + relativePart.Length];

    if (length > 0)
    {
        left.CopyTo(0, path, 0, length);
        while (length > 0)
        {
            if (path[--length] == '/')
            {
                ++length;
                break;
            }
        }
    }

    //Append relative path to the result
    relativePart.CopyTo(0, path, length, relativePart.Length);

    // Split relative on path and extra (for compression)
    c1 = basePart.Syntax.InFact(UriSyntaxFlags.MayHaveQuery) ? '?' : c_DummyChar;

    // The  implicit file check is to avoid a fragment in the implicit file combined uri.
    char c2 = (!basePart.IsImplicitFile && basePart.Syntax.InFact(UriSyntaxFlags.MayHaveFragment)) ? '#' :
        c_DummyChar;
    ReadOnlySpan<char> extra = string.Empty;

    // assuming c_DummyChar may not happen in an unicode uri string
    if (!(c1 == c_DummyChar && c2 == c_DummyChar))
    {
        int i = 0;
        for (; i < relativePart.Length; ++i)
        {
            if (path[length + i] == c1 || path[length + i] == c2)
            {
                break;
            }
        }
        if (i == 0)
        {
            extra = relativePart;
        }
        else if (i < relativePart.Length)
        {
            extra = relativePart.AsSpan(i);
        }
        length += i;
    }
    else
    {
        length += relativePart.Length;
    }

    // Take the base part up to the path
    if (basePart.HostType == Flags.IPv6HostType)
    {
        if (basePart.IsImplicitFile)
        {
            left = @"\\[" + basePart.DnsSafeHost + ']';
        }
        else
        {
            left = basePart.GetParts(UriComponents.Scheme | UriComponents.UserInfo, uriFormat)
                    + '[' + basePart.DnsSafeHost + ']'
                    + basePart.GetParts(UriComponents.KeepDelimiter | UriComponents.Port, uriFormat);
        }
    }
    else
    {
        if (basePart.IsImplicitFile)
        {
            if (basePart.IsDosPath)
            {
                // The FILE DOS path comes as /c:/path, we have to exclude first 3 chars from compression
                Compress(path, 3, ref length, basePart.Syntax);
                return string.Concat(path.AsSpan(1, length - 1), extra);
            }
            else if (!OperatingSystem.IsWindows() && basePart.IsUnixPath)
            {
                left = basePart.GetParts(UriComponents.Host, UriFormat.Unescaped);
            }
            else
            {
                left = @"\\" + basePart.GetParts(UriComponents.Host, UriFormat.Unescaped);
            }
        }
        else
        {
            left = basePart.GetParts(UriComponents.SchemeAndServer | UriComponents.UserInfo, uriFormat);
        }
    }
    //compress the path
    Compress(path, basePart.SecuredPathIndex, ref length, basePart.Syntax);
    return string.Concat(left, path.AsSpan(0, length), extra);
}
```

> [System.Uri源码](https://source.dot.net/#System.Private.Uri/System/Uri.cs,04da67e14624f9b6)