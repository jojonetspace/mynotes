npm install
npm run serve

启动 Vite 开发服务器（端口 81，根据 vite.config.ts 中的配置）。
Local:   http://localhost:81/
Network: http://0.0.0.0:81/
此时 G.Web 页面只要设置了 `js_local=1` 的 Cookie，刷新就会加载你本地修改后的 j-table 代码。