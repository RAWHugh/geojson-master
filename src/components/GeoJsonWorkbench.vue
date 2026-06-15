<template>
  <div class="shell">
    <header class="topbar">
      <div class="brand">
        <span class="brand__title">geojson.studio</span>
        <span class="brand__subtitle">在线 GeoJSON 绘制与转换</span>
      </div>

      <div class="topbar__actions">
        <label class="file-button ghost-button">
          <input
            ref="fileInputRef"
            type="file"
            multiple
            @change="onFileInputChange"
          />
          导入
        </label>

        <select
          v-model="exportFormat"
          class="toolbar-select toolbar-select--top"
        >
          <option value="geojson">导出 GeoJSON</option>
          <option value="kml">导出 KML</option>
          <option value="csv">导出 CSV</option>
        </select>

        <button class="ghost-button" @click="exportData">导出</button>
        <button class="primary-button" @click="copyShareLink">分享链接</button>
      </div>
    </header>

    <main class="workspace">
      <aside class="panel panel--left">
        <div class="card side-card">
          <div class="side-card__section">
            <h2>绘图工具</h2>
            <div class="tool-grid">
              <button
                v-for="tool in drawTools"
                :key="tool.key"
                class="tool-button"
                :class="{ 'tool-button--active': activeTool === tool.key }"
                @click="setTool(tool.key)"
              >
                <span class="tool-button__label">{{ tool.label }}</span>
                <span class="tool-button__hint">{{ tool.hint }}</span>
              </button>
            </div>
          </div>

          <div class="side-card__section side-card__section--compact">
            <label class="field">
              <span class="field__label">Label 字段</span>
              <select v-model="labelField" class="toolbar-select">
                <option value="">不显示</option>
                <option
                  v-for="field in availablePropertyKeys"
                  :key="field"
                  :value="field"
                >
                  {{ field }}
                </option>
              </select>
            </label>

            <label class="field">
              <span class="field__label">底图</span>
              <select v-model="basemapKey" class="toolbar-select">
                <option value="gaode-street">高德街道</option>
                <option value="arcgis-image">ArcGIS 影像</option>
                <option value="osm">OSM</option>
              </select>
            </label>
          </div>

          <div class="side-card__section side-card__section--compact">
            <button class="secondary-button" @click="fitToData">
              定位到数据
            </button>
            <button
              class="secondary-button"
              :disabled="!selectedFeatureId"
              @click="removeSelectedFeature"
            >
              删除选中要素
            </button>
            <button class="danger-button" @click="clearAll">
              清空全部
            </button>
          </div>

          <div class="side-card__section">
            <p class="guide-text">
              用上方工具直接画点、线、面，或导入 GeoJSON、KML、GPX、CSV、
              SHP Zip 文件。地图、右侧 GeoJSON 代码、属性表会实时同步。
            </p>

            <label
              class="import-dropzone"
              @dragover.prevent
              @drop.prevent="onDropFiles"
            >
              <input
                ref="dropInputRef"
                type="file"
                multiple
                @change="onFileInputChange"
              />
              <span class="import-dropzone__title">+ Import</span>
              <span class="import-dropzone__text">
                拖拽文件到这里，或点击上传
              </span>
              <span class="import-dropzone__types">
                Supported: GeoJSON / TopoJSON / KML / GPX / CSV / SHP(.zip)
              </span>
            </label>
          </div>

          <div class="side-card__section">
            <div class="meta-list">
              <span>要素数量：{{ featureCount }}</span>
              <span>选中要素：{{ selectedFeatureId || "无" }}</span>
              <span>状态：{{ syncStatusText }}</span>
            </div>
            <p v-if="messageText" class="status-text">{{ messageText }}</p>
            <p v-if="jsonError" class="error-text">{{ jsonError }}</p>
          </div>
        </div>
      </aside>

      <section class="panel panel--center">
        <div class="card map-card">
          <div ref="mapRef" class="map-view"></div>

          <div class="map-toolbar">
            <div class="map-toolbar__pan">
              <button class="icon-button" @click="locateUser">◎</button>
            </div>

            <div class="map-toolbar__search">
              <input
                v-model.trim="searchKeyword"
                class="search-input"
                placeholder="搜索地名、地址或输入 lng,lat"
                @keydown.enter.prevent="searchLocation"
              />
              <button class="secondary-button" @click="searchLocation">
                搜索
              </button>
            </div>
          </div>
        </div>
      </section>

      <aside class="panel panel--right">
        <div class="card editor-card">
          <div class="tabbar">
            <button
              class="tab-button"
              :class="{ 'tab-button--active': activeRightTab === 'json' }"
              @click="activeRightTab = 'json'"
            >
              JSON
            </button>
            <button
              class="tab-button"
              :class="{ 'tab-button--active': activeRightTab === 'table' }"
              @click="activeRightTab = 'table'"
            >
              Table
            </button>
          </div>

          <div v-if="activeRightTab === 'json'" class="editor-panel">
            <textarea
              v-model="jsonText"
              class="json-editor"
              spellcheck="false"
              @input="onJsonEditorInput"
            ></textarea>
          </div>

          <div v-else class="editor-panel editor-panel--table">
            <div class="table-toolbar">
              <input
                v-model.trim="newPropertyName"
                class="text-input"
                placeholder="新增属性字段，例如 name"
                @keydown.enter.prevent="addPropertyColumn"
              />
              <button class="secondary-button" @click="addPropertyColumn">
                添加字段
              </button>
            </div>

            <div class="table-scroll">
              <table class="feature-table">
                <thead>
                  <tr>
                    <th>id</th>
                    <th>geometry</th>
                    <th
                      v-for="field in editablePropertyKeys"
                      :key="field"
                    >
                      <div class="table-heading">
                        <span>{{ field }}</span>
                        <button
                          class="text-button"
                          @click="removePropertyColumn(field)"
                        >
                          删除
                        </button>
                      </div>
                    </th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    v-for="row in tableRows"
                    :key="row.id"
                    :class="{ 'feature-row--active': row.id === selectedFeatureId }"
                    @click="selectFeatureById(row.id)"
                  >
                    <td>{{ row.id }}</td>
                    <td>{{ row.geometryType }}</td>
                    <td v-for="field in editablePropertyKeys" :key="field">
                      <input
                        :value="row.properties[field] ?? ''"
                        class="table-input"
                        @input="updateFeatureProperty(row.id, field, $event.target.value)"
                      />
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </aside>
    </main>
  </div>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, ref, watch } from "vue";
import GeoJSON from "ol/format/GeoJSON";
import KML from "ol/format/KML";
import GPX from "ol/format/GPX";
import Map from "ol/Map";
import View from "ol/View";
import Feature from "ol/Feature";
import Point from "ol/geom/Point";
import TileLayer from "ol/layer/Tile";
import VectorLayer from "ol/layer/Vector";
import VectorSource from "ol/source/Vector";
import XYZ from "ol/source/XYZ";
import OSM from "ol/source/OSM";
import { Fill, Stroke, Style, Circle as CircleStyle, Text } from "ol/style";
import { defaults as defaultControls, ScaleLine } from "ol/control";
import {
  defaults as defaultInteractions,
  Draw,
  Modify,
  Select,
  Snap
} from "ol/interaction";
import { click } from "ol/events/condition";
import { createBox, createRegularPolygon } from "ol/interaction/Draw";
import { fromLonLat } from "ol/proj";
import { createEmpty, extend } from "ol/extent";
import Papa from "papaparse";
import shp from "shpjs";
import { feature as topojsonFeature } from "topojson-client";

const defaultGeoJson = {
  type: "FeatureCollection",
  features: []
};

const defaultMapCenter = [108.923611, 34.540833];
const defaultMapZoom = 4;

const geoJsonFormat = new GeoJSON({
  featureProjection: "EPSG:3857",
  dataProjection: "EPSG:4326"
});

const drawTools = [
  { key: "select", label: "选择", hint: "移动与修改" },
  { key: "point", label: "点", hint: "创建点位" },
  { key: "line", label: "线", hint: "绘制折线" },
  { key: "polygon", label: "面", hint: "绘制多边形" },
  { key: "rectangle", label: "矩形", hint: "快速框选" },
  { key: "circle", label: "圆", hint: "生成圆面" }
];

const mapRef = ref(null);

const activeTool = ref("select");
const activeRightTab = ref("json");
const exportFormat = ref("geojson");
const searchKeyword = ref("");
const labelField = ref("");
const basemapKey = ref("gaode-street");
const jsonText = ref(JSON.stringify(defaultGeoJson, null, 2));
const jsonError = ref("");
const messageText = ref("");
const newPropertyName = ref("");
const selectedFeatureId = ref("");
const syncStatusText = ref("已同步");

let map;
let vectorSource;
let vectorLayer;
let baseLayer;
let selectInteraction;
let modifyInteraction;
let snapInteraction;
let drawInteraction;
let messageTimer;
let applyingJson = false;

const featuresState = ref(structuredClone(defaultGeoJson));

const editablePropertyKeys = computed(() => {
  const keys = new Set();
  for (const feature of featuresState.value.features) {
    const properties = feature.properties || {};
    for (const key of Object.keys(properties)) {
      if (key !== "__featureId") {
        keys.add(key);
      }
    }
  }
  return Array.from(keys);
});

const availablePropertyKeys = computed(() => editablePropertyKeys.value);

const tableRows = computed(() =>
  featuresState.value.features.map((feature) => ({
    id: feature.id || feature.properties?.__featureId || "",
    geometryType: feature.geometry?.type || "",
    properties: feature.properties || {}
  }))
);

const featureCount = computed(() => featuresState.value.features.length);

watch(basemapKey, () => {
  updateBasemap();
});

watch(labelField, () => {
  if (vectorLayer) {
    vectorLayer.changed();
  }
});

watch(
  featuresState,
  (value) => {
    if (applyingJson) {
      return;
    }
    jsonText.value = JSON.stringify(value, null, 2);
    syncStatusText.value = "已同步";
  },
  { deep: true }
);

onMounted(() => {
  createMap();
  loadFromShareLink();
});

onBeforeUnmount(() => {
  window.clearTimeout(messageTimer);
  if (map) {
    map.setTarget(null);
  }
});

function createMap() {
  vectorSource = new VectorSource();
  baseLayer = new TileLayer({
    source: createBaseSource(basemapKey.value)
  });

  vectorLayer = new VectorLayer({
    source: vectorSource,
    style: (feature) => createFeatureStyle(feature)
  });

  map = new Map({
    target: mapRef.value,
    layers: [baseLayer, vectorLayer],
    controls: defaultControls({
      attribution: false,
      rotate: false
    }).extend([new ScaleLine()]),
    interactions: defaultInteractions({
      doubleClickZoom: false
    }),
    view: new View({
      center: fromLonLat(defaultMapCenter),
      zoom: defaultMapZoom
    })
  });

  selectInteraction = new Select({
    condition: click,
    layers: [vectorLayer],
    style: (feature) => createFeatureStyle(feature, true)
  });

  modifyInteraction = new Modify({
    features: selectInteraction.getFeatures()
  });

  snapInteraction = new Snap({
    source: vectorSource
  });

  selectInteraction.on("select", (event) => {
    const feature = event.selected[0];
    selectedFeatureId.value = feature ? ensureFeatureId(feature) : "";
  });

  modifyInteraction.on("modifyend", () => {
    syncFeaturesFromMap(true);
  });

  map.addInteraction(selectInteraction);
  map.addInteraction(modifyInteraction);
  map.addInteraction(snapInteraction);
  setTool(activeTool.value);
  syncMapFromState();
}

function createBaseSource(key) {
  if (key === "gaode-street") {
    return new XYZ({
      urls: [
        "http://webst01.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}",
        "http://webst02.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}",
        "http://webst03.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}",
        "http://webst04.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}"
      ]
    });
  }

  if (key === "arcgis-image") {
    return new XYZ({
      url: "https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}"
    });
  }

  return new OSM();
}

function updateBasemap() {
  if (!baseLayer) {
    return;
  }
  baseLayer.setSource(createBaseSource(basemapKey.value));
}

function createFeatureStyle(feature, selected = false) {
  const geometryType = feature.getGeometry()?.getType();
  const label = labelField.value
    ? String(feature.get(labelField.value) ?? "")
    : "";

  const stroke = new Stroke({
    color: selected ? "#f97316" : "#1d5fd0",
    width: selected ? 3 : 2
  });

  const fill = new Fill({
    color:
      geometryType === "Polygon" || geometryType === "MultiPolygon"
        ? selected
          ? "rgba(249, 115, 22, 0.22)"
          : "rgba(29, 95, 208, 0.18)"
        : "rgba(29, 95, 208, 0.08)"
  });

  return new Style({
    stroke,
    fill,
    image: new CircleStyle({
      radius: selected ? 8 : 6,
      fill: new Fill({
        color: selected ? "#f97316" : "#1d5fd0"
      }),
      stroke: new Stroke({
        color: "#ffffff",
        width: 2
      })
    }),
    text: label
      ? new Text({
          text: label,
          offsetY: -18,
          padding: [4, 6, 4, 6],
          backgroundFill: new Fill({
            color: "rgba(16, 33, 47, 0.88)"
          }),
          fill: new Fill({
            color: "#ffffff"
          })
        })
      : undefined
  });
}

function setTool(toolKey) {
  activeTool.value = toolKey;

  if (drawInteraction) {
    map.removeInteraction(drawInteraction);
    drawInteraction = undefined;
  }

  const selectedFeatures = selectInteraction?.getFeatures();
  if (selectedFeatures) {
    selectedFeatures.clear();
  }

  selectedFeatureId.value = "";

  if (toolKey === "select") {
    selectInteraction.setActive(true);
    modifyInteraction.setActive(true);
    return;
  }

  selectInteraction.setActive(false);
  modifyInteraction.setActive(false);

  const options = resolveDrawOptions(toolKey);
  drawInteraction = new Draw({
    source: vectorSource,
    ...options
  });

  drawInteraction.on("drawend", (event) => {
    ensureFeatureId(event.feature);
    if (!event.feature.get("name")) {
      event.feature.set("name", `${options.type}-${featureCount.value + 1}`);
    }
    setTool("select");
    window.requestAnimationFrame(() => {
      syncFeaturesFromMap(true);
      selectedFeatureId.value = ensureFeatureId(event.feature);
    });
  });

  map.addInteraction(drawInteraction);
}

function resolveDrawOptions(toolKey) {
  if (toolKey === "point") {
    return { type: "Point" };
  }
  if (toolKey === "line") {
    return { type: "LineString" };
  }
  if (toolKey === "rectangle") {
    return {
      type: "Circle",
      geometryFunction: createBox()
    };
  }
  if (toolKey === "circle") {
    return {
      type: "Circle",
      geometryFunction: createRegularPolygon(64)
    };
  }
  return { type: "Polygon" };
}

function ensureFeatureId(feature) {
  const existingId = feature.getId() || feature.get("__featureId");
  if (existingId) {
    feature.set("__featureId", String(existingId));
    feature.setId(String(existingId));
    return String(existingId);
  }

  const nextId = `feature-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
  feature.setId(nextId);
  feature.set("__featureId", nextId);
  return nextId;
}

function syncFeaturesFromMap(updateHash = false) {
  const features = vectorSource.getFeatures();
  for (const feature of features) {
    ensureFeatureId(feature);
  }

  const featureCollection = geoJsonFormat.writeFeaturesObject(features);
  featureCollection.type = "FeatureCollection";
  featureCollection.features = featureCollection.features.map((feature, index) => {
    const cloned = structuredClone(feature);
    const sourceFeature = features[index];
    const featureId = ensureFeatureId(sourceFeature);
    cloned.id = featureId;
    cloned.properties = {
      ...cloned.properties,
      __featureId: featureId
    };
    return cloned;
  });

  applyingJson = true;
  featuresState.value = featureCollection;
  jsonText.value = JSON.stringify(featureCollection, null, 2);
  applyingJson = false;
  jsonError.value = "";
  syncStatusText.value = "已同步";

  if (updateHash) {
    updateShareLinkHash();
  }
}

function syncMapFromState() {
  applyingJson = true;
  vectorSource.clear();
  const features = geoJsonFormat.readFeatures(featuresState.value);
  for (const feature of features) {
    ensureFeatureId(feature);
  }
  vectorSource.addFeatures(features);
  applyingJson = false;

  if (selectedFeatureId.value) {
    highlightFeatureById(selectedFeatureId.value);
  }

  if (featureCount.value > 0) {
    fitToData();
  }
}

function onJsonEditorInput() {
  try {
    const parsed = JSON.parse(jsonText.value);
    normalizeFeatureCollection(parsed);
    featuresState.value = parsed;
    jsonError.value = "";
    syncStatusText.value = "JSON 已应用";
    syncMapFromState();
    updateShareLinkHash();
  } catch (error) {
    jsonError.value = `JSON 解析失败：${error.message}`;
    syncStatusText.value = "JSON 有错误";
  }
}

function normalizeFeatureCollection(data) {
  if (!data || data.type !== "FeatureCollection" || !Array.isArray(data.features)) {
    throw new Error("必须是合法的 FeatureCollection。");
  }

  data.features = data.features.map((feature) => {
    if (!feature || feature.type !== "Feature") {
      throw new Error("features 中只允许 Feature。");
    }
    const nextFeature = structuredClone(feature);
    const featureId =
      String(nextFeature.id || nextFeature.properties?.__featureId || "") ||
      `feature-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
    nextFeature.id = featureId;
    nextFeature.properties = {
      ...(nextFeature.properties || {}),
      __featureId: featureId
    };
    return nextFeature;
  });
}

function selectFeatureById(featureId) {
  selectedFeatureId.value = featureId;
  highlightFeatureById(featureId);
}

function highlightFeatureById(featureId) {
  const target = vectorSource
    .getFeatures()
    .find((feature) => ensureFeatureId(feature) === featureId);

  selectInteraction.getFeatures().clear();
  if (target) {
    selectInteraction.getFeatures().push(target);
    const geometry = target.getGeometry();
    if (geometry) {
      const extent = geometry.getExtent();
      map.getView().fit(extent, {
        padding: [80, 80, 80, 80],
        maxZoom: 16,
        duration: 250
      });
    }
  }
}

function updateFeatureProperty(featureId, key, value) {
  const nextFeatures = featuresState.value.features.map((feature) => {
    const currentId = feature.id || feature.properties?.__featureId;
    if (currentId !== featureId) {
      return feature;
    }
    return {
      ...feature,
      properties: {
        ...(feature.properties || {}),
        [key]: value
      }
    };
  });

  featuresState.value = {
    ...featuresState.value,
    features: nextFeatures
  };

  syncMapFromState();
  updateShareLinkHash();
}

function addPropertyColumn() {
  const key = newPropertyName.value.trim();
  if (!key) {
    return;
  }

  const nextFeatures = featuresState.value.features.map((feature) => ({
    ...feature,
    properties: {
      ...(feature.properties || {}),
      [key]: feature.properties?.[key] ?? ""
    }
  }));

  featuresState.value = {
    ...featuresState.value,
    features: nextFeatures
  };
  newPropertyName.value = "";
  syncMapFromState();
  updateShareLinkHash();
}

function removePropertyColumn(key) {
  const nextFeatures = featuresState.value.features.map((feature) => {
    const properties = { ...(feature.properties || {}) };
    delete properties[key];
    return {
      ...feature,
      properties
    };
  });

  if (labelField.value === key) {
    labelField.value = "";
  }

  featuresState.value = {
    ...featuresState.value,
    features: nextFeatures
  };
  syncMapFromState();
  updateShareLinkHash();
}

function removeSelectedFeature() {
  if (!selectedFeatureId.value) {
    return;
  }

  const target = vectorSource
    .getFeatures()
    .find((feature) => ensureFeatureId(feature) === selectedFeatureId.value);

  if (target) {
    vectorSource.removeFeature(target);
    selectedFeatureId.value = "";
    syncFeaturesFromMap(true);
  }
}

function clearAll() {
  selectedFeatureId.value = "";
  featuresState.value = structuredClone(defaultGeoJson);
  syncMapFromState();
  jsonText.value = JSON.stringify(defaultGeoJson, null, 2);
  updateShareLinkHash();
}

function fitToData() {
  const features = vectorSource.getFeatures();
  if (!features.length) {
    map.getView().animate({
      center: fromLonLat(defaultMapCenter),
      zoom: defaultMapZoom,
      duration: 300
    });
    return;
  }

  const extent = createEmpty();
  for (const feature of features) {
    const geometry = feature.getGeometry();
    if (geometry) {
      extend(extent, geometry.getExtent());
    }
  }

  map.getView().fit(extent, {
    padding: [80, 80, 80, 80],
    duration: 300,
    maxZoom: 17
  });
}

function locateUser() {
  if (!navigator.geolocation) {
    pushMessage("当前浏览器不支持定位。");
    return;
  }

  navigator.geolocation.getCurrentPosition(
    (position) => {
      const center = fromLonLat([
        position.coords.longitude,
        position.coords.latitude
      ]);
      map.getView().animate({
        center,
        zoom: 14,
        duration: 400
      });
    },
    () => {
      pushMessage("定位失败，请检查浏览器权限。");
    }
  );
}

async function searchLocation() {
  const keyword = searchKeyword.value.trim();
  if (!keyword) {
    return;
  }

  const coordinateMatch = keyword.match(
    /^\s*(-?\d+(?:\.\d+)?)\s*[,， ]\s*(-?\d+(?:\.\d+)?)\s*$/
  );

  if (coordinateMatch) {
    const lon = Number(coordinateMatch[1]);
    const lat = Number(coordinateMatch[2]);
    map.getView().animate({
      center: fromLonLat([lon, lat]),
      zoom: 14,
      duration: 320
    });
    return;
  }

  try {
    const url = new URL("https://nominatim.openstreetmap.org/search");
    url.searchParams.set("format", "jsonv2");
    url.searchParams.set("limit", "1");
    url.searchParams.set("q", keyword);

    const response = await fetch(url.toString(), {
      headers: {
        Accept: "application/json"
      }
    });
    const results = await response.json();
    if (!Array.isArray(results) || !results.length) {
      pushMessage("未找到匹配位置。");
      return;
    }
    const result = results[0];
    map.getView().animate({
      center: fromLonLat([Number(result.lon), Number(result.lat)]),
      zoom: 14,
      duration: 320
    });
  } catch (error) {
    pushMessage(`搜索失败：${error.message}`);
  }
}

function onDropFiles(event) {
  const files = Array.from(event.dataTransfer?.files || []);
  if (files.length) {
    importFiles(files);
  }
}

function onFileInputChange(event) {
  const files = Array.from(event.target.files || []);
  if (files.length) {
    importFiles(files);
  }
  event.target.value = "";
}

async function importFiles(files) {
  try {
    let importedFeatures = [];
    for (const file of files) {
      const features = await parseFile(file);
      importedFeatures = importedFeatures.concat(features);
    }

    if (!importedFeatures.length) {
      pushMessage("未从文件中解析到可用要素。");
      return;
    }

    vectorSource.addFeatures(importedFeatures);
    syncFeaturesFromMap(true);
    fitToData();
    pushMessage(`已导入 ${importedFeatures.length} 个要素。`);
  } catch (error) {
    pushMessage(`导入失败：${error.message}`);
  }
}

async function parseFile(file) {
  const extension = file.name.split(".").pop()?.toLowerCase() || "";

  if (extension === "json" || extension === "geojson" || extension === "topojson") {
    const text = await file.text();
    const parsed = JSON.parse(text);
    if (parsed.type === "Topology") {
      return geoJsonFormat.readFeatures(convertTopoJsonToFeatureCollection(parsed));
    }
    return geoJsonFormat.readFeatures(parsed);
  }

  if (extension === "kml") {
    const text = await file.text();
    return new KML().readFeatures(text, {
      featureProjection: "EPSG:3857"
    });
  }

  if (extension === "gpx") {
    const text = await file.text();
    return new GPX().readFeatures(text, {
      featureProjection: "EPSG:3857"
    });
  }

  if (extension === "csv") {
    const text = await file.text();
    return parseCsvFeatures(text);
  }

  if (extension === "zip" || extension === "shp") {
    const arrayBuffer = await file.arrayBuffer();
    const parsed = await shp(arrayBuffer);
    const featureCollection = Array.isArray(parsed)
      ? mergeFeatureCollections(parsed)
      : parsed;
    return geoJsonFormat.readFeatures(featureCollection);
  }

  throw new Error(`不支持的文件格式：${file.name}`);
}

function parseCsvFeatures(text) {
  const result = Papa.parse(text, {
    header: true,
    skipEmptyLines: true
  });

  const rows = result.data || [];
  const features = [];

  for (const row of rows) {
    const lon =
      row.lon ?? row.lng ?? row.longitude ?? row.x ?? row.LON ?? row.LNG;
    const lat =
      row.lat ?? row.latitude ?? row.y ?? row.LAT ?? row.LATITUDE;

    if (lon == null || lat == null) {
      continue;
    }

    const point = new Point(fromLonLat([Number(lon), Number(lat)]));
    const feature = new Feature(point);

    for (const [key, value] of Object.entries(row)) {
      feature.set(key, value);
    }

    features.push(feature);
  }

  return features;
}

function mergeFeatureCollections(collections) {
  return {
    type: "FeatureCollection",
    features: collections.flatMap((item) => item.features || [])
  };
}

function convertTopoJsonToFeatureCollection(topology) {
  const objectEntries = Object.entries(topology.objects || {});
  if (!objectEntries.length) {
    throw new Error("TopoJSON 中没有可转换的对象。");
  }

  const collections = objectEntries.map(([, objectValue]) =>
    topojsonFeature(topology, objectValue)
  );

  return {
    type: "FeatureCollection",
    features: collections.flatMap((item) => {
      if (item.type === "FeatureCollection") {
        return item.features || [];
      }
      return item ? [item] : [];
    })
  };
}

function exportData() {
  const features = vectorSource.getFeatures();
  if (!features.length) {
    pushMessage("当前没有可导出的要素。");
    return;
  }

  let blob;
  let filename;

  if (exportFormat.value === "kml") {
    const content = new KML().writeFeatures(features, {
      featureProjection: "EPSG:3857",
      dataProjection: "EPSG:4326"
    });
    blob = new Blob([content], {
      type: "application/vnd.google-earth.kml+xml;charset=utf-8"
    });
    filename = "features.kml";
  } else if (exportFormat.value === "csv") {
    const rows = featuresState.value.features.map((feature) => {
      const row = {
        id: feature.id || feature.properties?.__featureId || "",
        geometryType: feature.geometry?.type || ""
      };

      if (feature.geometry?.type === "Point") {
        const [lon, lat] = feature.geometry.coordinates;
        row.lon = lon;
        row.lat = lat;
      }

      return {
        ...row,
        ...(feature.properties || {})
      };
    });
    const content = Papa.unparse(rows);
    blob = new Blob([content], {
      type: "text/csv;charset=utf-8"
    });
    filename = "features.csv";
  } else {
    const content = JSON.stringify(featuresState.value, null, 2);
    blob = new Blob([content], {
      type: "application/geo+json;charset=utf-8"
    });
    filename = "features.geojson";
  }

  const url = URL.createObjectURL(blob);
  const anchor = document.createElement("a");
  anchor.href = url;
  anchor.download = filename;
  anchor.click();
  URL.revokeObjectURL(url);
}

async function copyShareLink() {
  updateShareLinkHash();
  try {
    await navigator.clipboard.writeText(window.location.href);
    pushMessage("分享链接已复制到剪贴板。");
  } catch {
    pushMessage("分享链接已写入地址栏，可手动复制。");
  }
}

function updateShareLinkHash() {
  const json = JSON.stringify(featuresState.value);
  const encoded = btoa(unescape(encodeURIComponent(json)));
  const url = new URL(window.location.href);
  url.hash = `data=${encoded}`;
  window.history.replaceState({}, "", url);
}

function loadFromShareLink() {
  const hash = window.location.hash.replace(/^#/, "");
  const params = new URLSearchParams(hash);
  const data = params.get("data");
  if (!data) {
    syncMapFromState();
    return;
  }

  try {
    const decoded = decodeURIComponent(escape(atob(data)));
    const parsed = JSON.parse(decoded);
    normalizeFeatureCollection(parsed);
    featuresState.value = parsed;
    jsonText.value = JSON.stringify(parsed, null, 2);
    syncMapFromState();
    pushMessage("已从分享链接恢复数据。");
  } catch (error) {
    pushMessage(`分享链接解析失败：${error.message}`);
    syncMapFromState();
  }
}

function pushMessage(text) {
  messageText.value = text;
  window.clearTimeout(messageTimer);
  messageTimer = window.setTimeout(() => {
    messageText.value = "";
  }, 3600);
}
</script>
