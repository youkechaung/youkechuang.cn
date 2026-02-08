# 世界地图 · 国家猜图

这是一个交互式的地图游戏，点击地图上的国家，然后输入你猜的国家名称来测试你的地理知识！

**使用说明：**
- 点击地图上的任意国家
- 在弹出的对话框中输入你猜的国家名称
- 回答正确会显示绿色，回答错误会显示红色并显示正确答案

**注意：** 此游戏需要 `world.geojson` 文件才能正常运行。请将世界国家边界数据文件放在以下位置之一：
- `/source/static/maps/world.geojson` （推荐）
- 或当前章节目录下

你可以从以下来源获取 world.geojson 文件：
- [Natural Earth Data](https://www.naturalearthdata.com/downloads/) - 下载 countries 数据
- [GitHub - datasets](https://github.com/datasets/geo-boundaries-world-110m) - 开源的 GeoJSON 数据

<div style="position: relative; width: 100%; height: 600px; margin: 20px 0;">
  <div class="tip" style="position: absolute; top: 10px; left: 50%; transform: translateX(-50%); background: white; padding: 8px 16px; border-radius: 6px; font-size: 14px; box-shadow: 0 2px 8px rgba(0,0,0,.2); z-index: 999;">
    点击地图上的国家，输入你猜的国家名称
  </div>
  <div id="map" style="width: 100%; height: 100%;"></div>
</div>

<!-- Leaflet -->
<link
  rel="stylesheet"
  href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
/>
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<script>
  // 1. 创建地图（无文字底图）
  const map = L.map('map', {
    zoomControl: true
  }).setView([20, 0], 2);

  // 2. OSM 底图（不显示国家名）
  L.tileLayer(
    'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
    {
      attribution: '',
      opacity: 0.3
    }
  ).addTo(map);

  // 3. 国家默认样式
  function defaultStyle() {
    return {
      color: '#555',
      weight: 1,
      fillColor: '#ddd',
      fillOpacity: 0.6
    };
  }

  function correctStyle() {
    return {
      fillColor: '#4caf50',
      fillOpacity: 0.7
    };
  }

  function wrongStyle() {
    return {
      fillColor: '#f44336',
      fillOpacity: 0.6
    };
  }

  let geoLayer;

  // 4. 加载世界国家 GeoJSON
  // 按优先级尝试多个数据源
  const dataSources = [
    // 优先使用在线CDN（更可靠）
    'https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson',
    // 备用CDN
    'https://cdn.jsdelivr.net/gh/holtzy/D3-graph-gallery@master/DATA/world.geojson',
    // 本地文件（如果在线资源不可用）
    '/static/maps/world.geojson',
    // 相对路径（当前目录）
    './world.geojson'
  ];

  // 内联备用数据（至少保证有5个国家可以测试）
  const fallbackData = {
    "type": "FeatureCollection",
    "features": [
      {"type": "Feature", "properties": {"ADMIN": "中国"}, "geometry": {"type": "Polygon", "coordinates": [[[73.0, 18.0], [135.0, 18.0], [135.0, 54.0], [73.0, 54.0], [73.0, 18.0]]]}},
      {"type": "Feature", "properties": {"ADMIN": "美国"}, "geometry": {"type": "Polygon", "coordinates": [[[-125.0, 25.0], [-66.0, 25.0], [-66.0, 49.0], [-125.0, 49.0], [-125.0, 25.0]]]}},
      {"type": "Feature", "properties": {"ADMIN": "俄罗斯"}, "geometry": {"type": "Polygon", "coordinates": [[[19.0, 41.0], [180.0, 41.0], [180.0, 82.0], [19.0, 82.0], [19.0, 41.0]]]}},
      {"type": "Feature", "properties": {"ADMIN": "巴西"}, "geometry": {"type": "Polygon", "coordinates": [[[-75.0, -35.0], [-35.0, -35.0], [-35.0, 5.0], [-75.0, 5.0], [-75.0, -35.0]]]}},
      {"type": "Feature", "properties": {"ADMIN": "印度"}, "geometry": {"type": "Polygon", "coordinates": [[[68.0, 6.0], [97.0, 6.0], [97.0, 37.0], [68.0, 37.0], [68.0, 6.0]]]}}
    ]
  };

  function loadGeoJSONData(data) {
    console.log('✅ 成功加载 GeoJSON 数据，包含', data.features?.length || 0, '个国家');
    if (!data.features || data.features.length === 0) {
      throw new Error('GeoJSON 数据为空');
    }
    geoLayer = L.geoJSON(data, {
      style: defaultStyle,
      onEachFeature: onEachCountry
    }).addTo(map);
    console.log('地图已成功渲染');
  }

  function tryLoadGeoJSON(index) {
    if (index >= dataSources.length) {
      console.warn('所有外部数据源都加载失败，使用内联备用数据');
      // 使用内联备用数据
      try {
        loadGeoJSONData(fallbackData);
        console.log('⚠️ 使用的是简化版地图数据（仅5个国家），建议下载完整数据以获得更好的体验');
      } catch (err) {
        console.error('连备用数据都加载失败:', err);
        alert('无法加载地图数据。\n\n请检查网络连接，或手动下载 world.geojson 文件放在：\n/source/static/maps/world.geojson\n\n数据来源：\nhttps://github.com/holtzy/D3-graph-gallery/blob/master/DATA/world.geojson');
      }
      return;
    }

    const url = dataSources[index];
    console.log(`[${index + 1}/${dataSources.length}] 尝试加载: ${url}`);

    fetch(url)
      .then(res => {
        console.log(`响应状态: ${res.status} ${res.statusText}`);
        if (!res.ok) {
          throw new Error(`HTTP ${res.status}: ${res.statusText}`);
        }
        return res.json();
      })
      .then(data => {
        loadGeoJSONData(data);
      })
      .catch(err => {
        console.warn(`❌ 从 ${url} 加载失败:`, err.message || err);
        // 继续尝试下一个数据源
        tryLoadGeoJSON(index + 1);
      });
  }

  // 开始加载
  tryLoadGeoJSON(0);

  // 5. 国家点击交互
  function onEachCountry(feature, layer) {
    layer.on('click', () => {
      const answer = feature.properties.ADMIN;

      const userInput = prompt('你觉得这是哪个国家？');

      if (!userInput) return;

      if (normalize(userInput) === normalize(answer)) {
        layer.setStyle(correctStyle());
        alert('回答正确 🎉');
      } else {
        layer.setStyle(wrongStyle());
        alert(`回答错误 ❌ 正确答案是：${answer}`);
      }
    });
  }

  function normalize(str) {
    return str.trim().toLowerCase();
  }
</script>

