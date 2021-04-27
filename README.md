# 使用 Threejs 和 D3 可视化全球新冠疫情

## 话不多说，整体效果是这样的，本文主要讲解的地球实现

![整体效果](/img1.png)

## 核心需求

- 地球半透明，可以看到背面
- 点阵式的全球地图
- 根据数据的经纬度生成对应的柱体
- 数值越大，柱体的颜色和高度就越深越长

### 引入Threejs和D3

```html
<script src="https://cdn.staticfile.org/three.js/r125/three.min.js"></script>
<script src="https://d3js.org/d3.v6.js"></script>
<script src="https://unpkg.com/three@0.125.0/examples/js/controls/OrbitControls.js"></script>
<script src="https://unpkg.com/three@0.125.0/examples/js/utils/BufferGeometryUtils.js"></script>
```

### 基本HTML结构
```html
<div id="box" style="width: 100%; height: 100%">
  <canvas id="canvas" style="width: 100%; height: 100%" />
</div>
```

### 定义必要的变量
```js
const box = document.getElementById("box");
const canvas = document.getElementById("canvas");

let glRender; // webgl渲染器
let camera; // 摄像机
let earthMesh; // 地球的Mesh
let scene; // 场景，一个大容器，可以理解为html中的body
let meshGroup; // 所有Mesh的容器，后面所有Mesh都会放在这里面，方便我们管理，可理解为一个div
let controls; // 轨道控制器，实现整体场景的控制
```

### 初始化相关变量
```js
// 创建webgl渲染器
glRender = new THREE.WebGLRenderer({ canvas, alpha: true });
glRender.setSize(canvas.clientWidth, canvas.clientHeight, false);
// 创建场景
scene = new THREE.Scene();

// 创建相机
const fov = 45;
const aspect = canvas.clientWidth / canvas.clientHeight;
const near = 1;
const far = 4000;
camera = new THREE.PerspectiveCamera(fov, aspect, near, far);
// 设置一个适当的位置
camera.position.z = 400;

// 轨道控制器
controls = new THREE.OrbitControls(camera, canvas);
controls.target.set(0, 0, 0);

// 创建容器
meshGroup = new THREE.Group();
// 把容器添加到场景中
scene.add(meshGroup);
```

### 创建地球
```js
// 首先创建一个球体
const globeRadius = 100; // 球体半径
const globeSegments = 64; // 球体面数，数量越大越光滑，性能消耗越大
const geometry = new THREE.SphereGeometry(
  globeRadius,
  globeSegments,
  globeSegments
);
// 创建球体材质，让球体有质感
const material = new THREE.MeshBasicMaterial({
  transparent: true, // 设置是否透明
  opacity: 0.5, // 透明度
  color: 0x000000, // 颜色
});
// 生成地球的网格对象
earthMesh = new THREE.Mesh(geometry, material);
// 把地球添加到mesh容器里
meshGroup.add(earthMesh);
```
### 基本的元素创建好了，但界面没有任何显示，我们还需要渲染场景
```js
function screenRender(){
  glRender.render(scene, camera);
  controls.update();
  requestAnimationFrame(screenRender);
}

screenRender()
```
### 这个时候你应该看到一个圆形的物体
![页面设置](/img2.jpg)
### 使用绘图处理工具绘制点阵墨卡托投影的贴图
![页面设置](/dots.jpg)
### 我们把地图所有点阵的坐标记录下，具体怎么样记录这里不详细展开了，因为这不是重点，最终结果保存在mapPoints.js
```js
export default {
  "points": [
    {
      "x": 1.5,
      "y": 2031.5
    },
    {
      "x": 1.5,
      "y": 2016.5
    },
    ...
  ]
}
```
### 创建点阵式地图
```js
// 引入点阵数据
import mapPoints from "./mapPoints.js";
/**
 * 生成点状世界地图方法
 */
function createMapPoints() {
  // 点的基本材质.
  const material = new THREE.MeshBasicMaterial({
    color: "#AAA",
  });

  const sphere = [];
  // 循环遍历所有点将2维坐标映射到3维坐标
  for (let point of mapPoints.points) {
    const pos = convertFlatCoordsToSphereCoords(point.x, point.y);
    if (pos.x && pos.y && pos.z) {
      // 生成点阵
      const pingGeometry = new THREE.SphereGeometry(0.4, 5, 5);
      pingGeometry.translate(pos.x, pos.y, pos.z);
      sphere.push(pingGeometry);
    }
  }
  // 合并所有点阵生成一个mesh对象
  const earthMapPoints = new THREE.Mesh(
    THREE.BufferGeometryUtils.mergeBufferGeometries(sphere),
    material
  );
  // 加入到mesh容器中
  meshGroup.add(earthMapPoints);
}

/**
 * 我们需要获取2维点的数组，循环遍历并将每个点转换为其3维位置。这是执行转换的功能。根据您创建的模板投影的大小，您可能需要调整前几个变量
 */
const globeWidth = 4098 / 2;
const globeHeight = 1968 / 2;

function convertFlatCoordsToSphereCoords(x, y) {
  let latitude = ((x - globeWidth) / globeWidth) * -180;
  let longitude = ((y - globeHeight) / globeHeight) * -90;
  latitude = (latitude * Math.PI) / 180;
  longitude = (longitude * Math.PI) / 180; 
  const radius = Math.cos(longitude) * globeRadius;
  const x = Math.cos(latitude) * radius;
  const y = Math.sin(longitude) * globeRadius;
  const z = Math.sin(latitude) * radius;
  return {
    x,
    y,
    z,
  };
}
```
### 在meshGroup.add(earthMesh)后面调用createMapPoints方法
```js
...
// 把地球添加到mesh容器里
meshGroup.add(earthMesh);
// 创建点阵图
createMapPoints();

// 渲染场景
screenRender();
...
```
### 漂亮！
![页面设置](/img3.jpg)

### 接下来我们生成柱体，数据采集于https://disease.sh，并已转换方便我们使用的结构，保存在data.js

```js
// 转换好的数据
import data from "./data.js"
// 定义数据的颜色映射关系，可以根据实际数据动态计算
const colors = ["#ffdfe0","#ffc0c0","#FF0000","#ee7070","#c80200","#900000","#510000","#290000"];
const domain = [1000,3000,10000,50000,100000,500000,1000000,1000000];
```
### 创建生成柱体方法
```js
function createBar() {
  if (!data || data.length === 0) return;

  let color;
  // d3比例尺
  const scale = d3.scaleLinear().domain(domain).range(colors);

  // 循环遍历数据
  data.forEach(({ lat, lng, value: size }) => {
    // 通过比例尺获取数据对应的颜色
    color = scale(size);
    const pos = convertLatLngToSphereCoords(lat, lng, globeRadius);
    if (pos.x && pos.y && pos.z) {
      // 我们使用立方体来生成柱状图
      const geometry = new THREE.BoxGeometry(2, 2, 1);
      // 移动立方体Z使其立在地球表面
      geometry.applyMatrix4(
        new THREE.Matrix4().makeTranslation(0, 0, -0.5)
      );
      // 生成立方体mesh
      const barMesh = new THREE.Mesh(
        geometry,
        new THREE.MeshBasicMaterial({
          color,
        })
      );
      // 设置位置
      barMesh.position.set(pos.x, pos.y, pos.z);
      // 设置朝向
      barMesh.lookAt(earthMesh.position);

      // 根据数据设置柱的长度, 除20000主了为了防止柱体过长，可以根据实际情况调整，或做成参数
      barMesh.scale.z = Math.max(size/20000, 0.1);
      barMesh.updateMatrix();
      // 加入到mesh容器
      meshGroup.add(barMesh);
    }
  });
}

// 经纬度转成球体坐标
  function convertLatLngToSphereCoords(latitude, longitude, radius) {
    const phi = (latitude * Math.PI) / 180;
    const theta = ((longitude - 180) * Math.PI) / 180;
    const x = -(radius + -1) * Math.cos(phi) * Math.cos(theta);
    const y = (radius + -1) * Math.sin(phi);
    const z = (radius + -1) * Math.cos(phi) * Math.sin(theta);
    return {
      x,
      y,
      z,
    };
  }
```
### 在createMapPoints()后调用createBar()
```js
...
// 在这个方法后面调用createMapPoints方法
meshGroup.add(earthMesh);
// 创建点阵图
createMapPoints();
// 创建柱状图
createBar()

// 渲染场景
screenRender();
...
```

### 最终效果
![页面设置](/img4.jpg)

