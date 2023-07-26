## 项目中使用PWA的问题与经历

### 一、为什么选择使用PWA？
  目标：希望部分页面实现离线访问、应用程序本地化(基于浏览器的Web App)
  + 1、业务探索
  + 2、技术探索

### 二、PWA实现工具选择？
  为了实现上面两点，选择一下工具：
  + `vite-plugin-pwa` (由于项目是基于 `vite` 构建，所以选择了此插件。该插件作用：)
    -  生成`Web App Manifest` (`Web App Manifest`是一个JSON文件，用于定义Web应用的元数据，包括应用图标、名称、主题色等信息。vite-plugin-pwa会根据配置生成并添加这个文件，使得应用在安装到主屏幕时表现得更像原生应用。)
    - 配置`Server Workers`（`Service Workers`是PWA的核心技术，它是一段运行在浏览器后台的脚本，可以拦截网络请求、缓存资源以及实现离线访问。`vite-plugin-pwa`会帮助自动生成并配置`Service Workers`，简化`Service Workers`的使用和注册过程）
  +  `virtual:pwa-register` (`vite-plugin-pwa`并不会自动注册`Service Worker`，它生成的sw.js配置文件需要该插件注册到应用程序中)




### 三、使用以上插件后遇到的问题？
  使用以上两个插件实现了想要的功能，但经过一个月的线上数据统计，真正使用到以上功能用户很少，于是技术侧选择了下掉该工具。
  <br>
  **下架方案：**
  + 移除 `vite-plugin-pwa` 在 `vite-config.ts` 中的配置
  + 移除 `virtual:pwa-register` 在 `main.ts` 中的配置

  **结果：**
  + 发布新版本后，用户侧刷新浏览器后仍然是老版本、强制刷新仍然解决不了(范围：pc 、wap 所有浏览器)
  
  **解决方案1.0：**
  + 1、干掉历史CDN文件和CDN节点缓存（由于项目中静态资源是通过cdn访问，历史版本的cdn文件仍然保留，并且部分cdn已经分发到一些结点上！）
  + 2、更改`nginx`缓存策略（临时去掉了的nginx缓存控制）
  
  **结果1.0：**
  + 未解决问题
  
  **解决方案2.0：**
    分析：浏览器控制台发现`Server Worker` 的配置文件已经注册到本地了，在注册的`sw.js`配置文件中还发现：浏览器刷新页面时拦截所有网络请求，并且如果你构建的新版本如果没有产生新的`sw.js`配置文件，它可能继续拦截所有网络请求。
  + 1、恢复之前对PWA的配置
  + 2、在HTML加入清除PWA缓存的代码（基于`Caches Api`，触发时机：每次刷新）
  
  **结果2.0：**
  + 恢复正常

### 四、兼容性问题？
  在`Safari`遇到一个奇怪的问题。有时候刷新页面时会触发页面无限 reload 情况，多次`reload`导致Safari缓存暴增；
  分析认为`Safari`对`Service Workers`的配置有一些限制和实现差异导致的
  **解决方案：**
  + 移除Server Worker 的注册，即去掉 `virtual:pwa-register` （这地方继续保留了`vite-plugin-pwa`插件，每次构建仍然会`sw.js`配置文件， 防止出现上面的情况）

### 五、最终解决方案？
  综合以上经历和分析：可以得出结论，以上问题的产生原因是由于对 `Server Worker` 配置不合理导致的，因为PWA自从2015年被`Google`首次提出来后，`PWA`已经逐渐成为了`Web`开发的一种主流技术，也有很多大型公司都采用了`PWA`来提供更好的用户体验。所以在使用`PWA`时，要特别注意对它的配置，特别是对它的缓存控制策略控制。
  <br>
  **为了彻底下掉`Server Worker`，即让用户浏览器中的`sw.js`不再工作 ，做了一下事情：**
  
  + 1、移除 `vite-plugin-pwa` 和  `virtual:pwa-register`
  + 2、在HTML增加一下代码：
  ```js
    if ('serviceWorker' in navigator) {
      const domain = 'chato.cn';
      const storageKey = `clearedSW_${domain}`;
      if (!localStorage.getItem(storageKey)) {
        navigator.serviceWorker.ready
          .then((registration) => {
            registration.unregister()
              .then((isUnregistered) => {
                if (isUnregistered) {
                  console.log(`Service Worker on ${domain} unregistered.`);
                  localStorage.setItem(storageKey, true);
                } else {
                  console.error(`Service Worker unregistration on ${domain} failed.`);
                }
              })
              .catch((error) => {
                console.error(`Service Worker unregistration on ${domain} failed:`, error);
              });
          });
      } else {
        console.log(`Service Worker on ${domain} has already been unregistered.`);
      }
    }
  ```
  **结果：**
    运行一段时间，用户浏览器中之前注册的 `Server Worker` 已经不再工作，`sw.js` 注册文件也已经被移除。
  
### 六、总结？
  尽管我们移除了`PWA`支持，但在这个过程中，我们学到了许多关于`PWA`实现和兼容性的知识。虽然在某些情况下PWA可能不是最理想的选择，但它仍然是一个令人兴奋的技术，可以为用户带来非常好的体验。在未来，随着浏览器技术的发展，我们可能会重新评估PWA是否适合我们的项目，并再次尝试为用户提供更好的体验。