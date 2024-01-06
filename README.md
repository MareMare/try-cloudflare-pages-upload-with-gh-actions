# try-cloudflare-pages-upload-with-gh-actions
GitHub Actions で `Cloudflare Pages` のアップロードを試してみます。
* [cloudflare/pages\-action](https://github.com/cloudflare/pages-action)
* [Use Direct Upload with continuous integration · Cloudflare Pages docs](https://developers.cloudflare.com/pages/how-to/use-direct-upload-with-continuous-integration/)
* [ASP\.NET Core Blazor WebAssembly のホストと展開 \| Microsoft Learn](https://learn.microsoft.com/ja-jp/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-7.0)

## GitHub Actions Secrets
![image](https://user-images.githubusercontent.com/807378/227702276-d8ee9ca2-f44d-4a97-a233-9d5c1205776c.png)

## Cloudflare API Token
![image](https://user-images.githubusercontent.com/807378/227702407-6236e320-a1d0-4db2-a565-751530a794d4.png)

## Cloudflare Account ID
![image](https://user-images.githubusercontent.com/807378/227702537-bb41d82b-87fa-4be2-a3e7-a8c068c44504.png)

## Ahead-Of-Time (AOT) compilation

* `*.csproj`
  ```xml
    <PropertyGroup>
      <TargetFramework>net8.0</TargetFramework>
      <RunAOTCompilation>true</RunAOTCompilation>
      <WasmEnableExceptionHandling>true</WasmEnableExceptionHandling>
    </PropertyGroup>
  ```
* `cloudflare-pages.yml`
  ```yml
      - name: 🧰 Setup .NET workloads
        run: dotnet workload install wasm-tools
  ```

## Brotli Compression
* `wwwroot\js\decode.js`／`wwwroot\js\decode.min.js`
  * [google/brotli · GitHub](https://github.com/google/brotli/tree/master/js) から `decode.js` と `decode.min.js` をダウンロード
  * `wwwroot\js` へコピー
* `*.csproj`
  ```xml
    <PropertyGroup>
      <BlazorEnableCompression>true</BlazorEnableCompression>
    </PropertyGroup>

    <ItemGroup>
      <Content Update="wwwroot\js\decode.js">
        <CopyToOutputDirectory>Always</CopyToOutputDirectory>
      </Content>
    </ItemGroup>
  ```
* `wwwroot\index.html`
  ```html
  <body>

      <!-- 👇 ここから -->
      <!-- NOTE: https://learn.microsoft.com/ja-jp/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-8.0#compression -->
      <script src="_framework/blazor.webassembly.js" autostart="false"></script>
      <script type="module">
          import { BrotliDecode } from './js/decode.min.js';
          Blazor.start({
            loadBootResource: function (type, name, defaultUri, integrity) {
              if (type !== 'dotnetjs' && location.hostname !== 'localhost' && type !== 'configuration') {
                return (async function () {
                  const response = await fetch(defaultUri + '.br', { cache: 'no-cache' });
                  if (!response.ok) {
                    throw new Error(response.statusText);
                  }
                  const originalResponseBuffer = await response.arrayBuffer();
                  const originalResponseArray = new Int8Array(originalResponseBuffer);
                  const decompressedResponseArray = BrotliDecode(originalResponseArray);
                  const contentType = type === 
                    'dotnetwasm' ? 'application/wasm' : 'application/octet-stream';
                  return new Response(decompressedResponseArray, 
                    { headers: { 'content-type': contentType } });
                })();
              }
            }
          });
      </script>
      <!-- 👆 ここまで -->
  </body>
  ```

  ## Run Blazor Web Assembly locally
* [natemcmaster/dotnet\-serve: Simple command\-line HTTPS server for the \.NET Core CLI](https://github.com/natemcmaster/dotnet-serve)
  ```ps1
  dotnet publish src/try-cloudflarepages -c release -o output
  dotnet serve -o -S -p:7286 -b -d:output/wwwroot
  ```
