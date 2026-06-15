# GeoJSON 在线绘制工具

基于 `Vue 3 + Vite + OpenLayers` 实现的轻量在线空间数据编辑器，按 `需求.md` 的技术路线完成以下能力：

- 点、线、面、矩形、圆面绘制
- 要素选择、拖拽修改、删除、清空
- GeoJSON 源码与地图双向实时同步
- 属性表格批量编辑与字段增删
- 导入 `GeoJSON / TopoJSON / KML / GPX / CSV / SHP(.zip)`
- 导出 `GeoJSON / KML / CSV`
- 底图切换：高德、天地图矢量、天地图影像、OSM
- 分享链接：当前数据写入地址栏 hash，可复制协作

## 启动

```bash
npm install
npm run dev
```

如果在 PowerShell 下遇到 `npm.ps1` 执行策略限制，可改用：

```bash
npm.cmd run dev
```

## 构建

```bash
npm.cmd run build
```

## Docker

构建镜像：

```bash
docker build -t geojson-master:latest .
```

运行容器：

```bash
docker run --rm -p 8080:80 geojson-master:latest
```

启动后访问：

```text
http://localhost:8080
```

![界面](/images/界面.png )
![加载GeoJSON](/images/加载geojson.png )
![绘制](/images/绘制.png )


## 说明

- 天地图底图需要自行填写可用 `Token`，否则会自动回退到 `OSM`
- 搜索功能支持输入 `lng,lat` 直接定位，也支持通过浏览器请求 `Nominatim` 进行地名检索
- Shapefile 导出未实现；当前按需求重点实现了多格式导入和 GeoJSON/KML/CSV 导出
