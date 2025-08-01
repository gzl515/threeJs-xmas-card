<template>
  <div class="scenes" style="
    position: fixed; 
    top: 0; 
    left: 0; 
    z-index: 10; 
    pointer-events: none; 
    transition: all 1s;
  " :style="{
    transform: `translate3D(0, ${-index * 100}vh, 0)`,
  }">
    <div v-for="item in scenes" style="width: 100vw; height: 100vh;">
      <h1 style="padding: 100px 50px; font-size: 50px; color: #fff;">{{ item.text }}</h1>
    </div>
  </div>
</template>

<script setup lang="ts">
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';  // 加载模型
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';  // 解压模型
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';  // 加载hdr图片
import { Water } from 'three/examples/jsm/objects/Water2.js';  // 水面效果
import gsap from 'gsap';
import { ref } from 'vue';

// 初始化场景
const scene = new THREE.Scene();
// 初始化相机
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
camera.position.set(-3.23, 2.98, 4.06);
camera.updateProjectionMatrix();
// 初始化渲染器
const renderer = new THREE.WebGLRenderer({
  // 设置抗锯齿
  antialias: true,
});
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
// 设置色调映射
renderer.outputColorSpace = THREE.SRGBColorSpace; // 输出颜色空间
renderer.toneMapping = THREE.ACESFilmicToneMapping; // 色调映射
renderer.toneMappingExposure = 0.5; // 色调映射曝光度
renderer.shadowMap.enabled = true; // 开启阴影贴图

// 初始化控制器
const controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(-8, 2, 0); // 设置控制器目标点
controls.enableDamping = true; // 开启阻尼效果

// 初始化loader
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://cdn.jsdelivr.net/npm/three@0.152.2/examples/jsm/libs/draco/');
const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

// 加载环境纹理
let rgbeLoader = new RGBELoader();
rgbeLoader.load('./textures/sky.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping; // 设置纹理映射方式
  scene.background = texture; // 设置场景背景
  scene.environment = texture; // 设置场景环境光照
});

// 加载模型
gltfLoader.load('./model/scene.glb', (gltf) => {
  const model = gltf.scene;
  model.traverse((child) => {
    if (child.name === 'Plane') {
      child.visible = false;
    }
    // if (child.isMesh) {
    //   child.castShadow = true; // 开启模型投射阴影
    //   child.receiveShadow = true; // 开启模型接收阴影
    // }
  });
  scene.add(model);
});

// 创建水面
const waterGeometry = new THREE.CircleGeometry(300, 32);
const water = new Water(waterGeometry, {
  textureWidth: 1024,
  textureHeight: 1024,
  color: 0xeeeeff,
  flowDirection: new THREE.Vector2(1, 1),
  scale: 100,
});
water.rotation.x = -Math.PI / 2; // 旋转水面
water.position.y = -0.4 // 设置水面位置
scene.add(water);

// 添加平行光
const light = new THREE.DirectionalLight(0xffffff, 1);
light.position.set(0, 50, 0);
scene.add(light);

// 添加点光源
const pointLight = new THREE.PointLight(0xffffff, 50);
pointLight.position.set(0.1, 2.4, 0);
pointLight.castShadow = true; // 开启阴影
scene.add(pointLight);

// 创建点光源组
const pointLightGroup = new THREE.Group();
pointLightGroup.position.set(-8, 2.5, -1.5);
let radius = 3;
let pointLightArr = [];
for (let i = 0; i < 3; i++) {
  // 创建球体当灯泡
  const sphereGeometry = new THREE.SphereGeometry(0.2, 32, 32);
  const sphereMaterial = new THREE.MeshStandardMaterial({
    color: 0xffffff,
    emissive: 0xffffff, // 发光颜色
    emissiveIntensity: 1, // 发光强度
  });
  const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial);
  pointLightArr.push(sphere);
  sphere.position.set(
    radius * Math.cos((i * 2 * Math.PI) / 3),
    Math.cos((i * 2 * Math.PI) / 3),
    radius * Math.sin((i * 2 * Math.PI) / 3)
  );

  let pointLight = new THREE.PointLight(0xffffff, 50);
  sphere.add(pointLight);

  pointLightGroup.add(sphere);
}
scene.add(pointLightGroup);

// 使用补间函数，从0到2Π，使灯泡旋转
let options = {
  angle: 0,
};
gsap.to(options, {
  angle: 2 * Math.PI,
  duration: 10,
  repeat: -1, // 无限循环
  ease: 'linear', // 线性补间
  onUpdate: () => {
    pointLightGroup.rotation.y = options.angle; // 更新灯泡组的旋转角度
    pointLightArr.forEach((sphere, index) => {
      sphere.position.set(
        radius * Math.cos((index * 2 * Math.PI) / 3),
        Math.cos((index * 2 * Math.PI) / 3 + options.angle * 5),
        radius * Math.sin((index * 2 * Math.PI) / 3)
      );
    });
  },
});

function render() {
  // 下一帧循环调用
  requestAnimationFrame(render);
  renderer.render(scene, camera); // 渲染场景
  controls.update(); // 更新控制器
}
render();

// 使用补间动画移动相机
let timeLine1 = gsap.timeline();
let timeLine2 = gsap.timeline();

// 定义相机移动函数
function translateCamera(position: any, target: any) {
  timeLine1.to(camera.position, {
    x: position.x,
    y: position.y,
    z: position.z,
    duration: 1,
    ease: 'power2.inOut',
  });
  timeLine2.to(controls.target, {
    x: target.x,
    y: target.y,
    z: target.z,
    duration: 1,
    ease: 'power2.inOut',
  });
}

let scenes = [
  {
    text: '生日快乐',
    callback: () => {
      // 执行函数切换位置
      translateCamera(
        new THREE.Vector3(-3.23, 2.98, 4.06),
        new THREE.Vector3(-8, 2, 0)
      );
    },
  },
  {
    text: '永远热爱',
    callback: () => {
      // 执行函数切换位置
      translateCamera(
        new THREE.Vector3(7, 0, 23),
        new THREE.Vector3(0, 0, 0)
      );
    },
  },
  {
    text: '永远年轻',
    callback: () => {
      // 执行函数切换位置
      translateCamera(
        new THREE.Vector3(10, 3, 0),
        new THREE.Vector3(5, 2, 0)
      );
    },
  },
  {
    text: '永远爱你',
    callback: () => {
      // 执行函数切换位置
      translateCamera(
        new THREE.Vector3(7, 0, 23),
        new THREE.Vector3(0, 0, 0)
      );
      makeHeart(); // 执行爱心动画
    },
  },
  {
    text: '永远一起',
    callback: () => {
      // 执行函数切换位置
      translateCamera(
        new THREE.Vector3(-20, 1.3, 6.6),
        new THREE.Vector3(5, 2, 0)
      );
    },
  },
]
let index = ref(0);
let isAnimate = false;
// 监听鼠标滚轮事件
window.addEventListener(
  'wheel',
  (event) => {
    if (isAnimate) return; // 如果正在动画中，则不处理滚轮事件
    isAnimate = true; // 设置为正在动画中
    if (event.deltaY > 0) {
      index.value++;
      if (index.value > scenes.length - 1) {
        index.value = 0; // 如果超过了最后一个场景，则回到第一个
        restoreHeart(); // 恢复爱心动画
      }
    }
    scenes[index.value].callback(); // 执行对应的回调函数
    setTimeout(() => {
      isAnimate = false; // 动画结束后设置为可以处理滚轮事件
    }, 1000); // 设置动画时间与 gsap 动画时间一致
  },
  false,
);

// 实例化创建漫天星星
let statInstance = new THREE.InstancedMesh(
  new THREE.SphereGeometry(0.1, 32, 32),
  new THREE.MeshStandardMaterial({
    color: 0xffffff,
    emissive: 0xffffff,
    emissiveIntensity: 10
  }),
  100,
);

// 星星随机分布到天上
let starsArr: any = [];
let endArr: any = [];

for (let i = 0; i < 100; i++) {
  let x = Math.random() * 100 - 50;
  let y = Math.random() * 100 - 50;
  let z = Math.random() * 100 - 50;
  starsArr.push(new THREE.Vector3(x, y, z));

  let matrix = new THREE.Matrix4();
  matrix.setPosition(x, y, z);
  statInstance.setMatrixAt(i, matrix); // 设置每个星星的位置
}
scene.add(statInstance); // 将星星添加到场景中

// 创建爱心路径
let heartShape = new THREE.Shape();
heartShape.moveTo(25, 25);
heartShape.bezierCurveTo(25, 25, 20, 0, 0, 0);
heartShape.bezierCurveTo(-30, 0, -30, 35, -30, 35);
heartShape.bezierCurveTo(-30, 55, -10, 77, 25, 95);
heartShape.bezierCurveTo(60, 77, 80, 55, 80, 35);
heartShape.bezierCurveTo(80, 35, 80, 0, 50, 0);
heartShape.bezierCurveTo(30, 0, 25, 25, 25, 25);

// 根据爱心路径获取点
let center = new THREE.Vector3(0, 2, 10);
for (let i = 0; i < 100; i++) {
  let point = heartShape.getPointAt(i / 100);
  endArr.push(new THREE.Vector3(
    point.x * 0.1 + center.x, point.y * 0.1 + center.y, center.z
  ));
}

// 创建爱心动画
function makeHeart() {
  let params = {
    time: 0,
  };
  gsap.to(params, {
    time: 1,
    duration: 1,
    onUpdate: () => {
      for (let i = 0; i < 100; i++) {
        let x = starsArr[i].x + (endArr[i].x - starsArr[i].x) * params.time;
        let y = starsArr[i].y + (endArr[i].y - starsArr[i].y) * params.time;
        let z = starsArr[i].z + (endArr[i].z - starsArr[i].z) * params.time;
        let matrix = new THREE.Matrix4();
        matrix.setPosition(x, y, z);
        statInstance.setMatrixAt(i, matrix); // 更新每个星星的位置
      }
      statInstance.instanceMatrix.needsUpdate = true; // 更新实例矩阵
    },
  })
}

function restoreHeart() {
  let params = {
    time: 0,
  };
  gsap.to(params, {
    time: 1,
    duration: 1,
    onUpdate: () => {
      for (let i = 0; i < 100; i++) {
        let x = endArr[i].x + (starsArr[i].x - endArr[i].x) * params.time;
        let y = endArr[i].y + (starsArr[i].y - endArr[i].y) * params.time;
        let z = endArr[i].z + (starsArr[i].z - endArr[i].z) * params.time;
        let matrix = new THREE.Matrix4();
        matrix.setPosition(x, y, z);
        statInstance.setMatrixAt(i, matrix); // 更新每个星星的位置
      }
      statInstance.instanceMatrix.needsUpdate = true; // 更新实例矩阵
    },
  })
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
}

canvas {
  width: 100vw;
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
}
</style>
