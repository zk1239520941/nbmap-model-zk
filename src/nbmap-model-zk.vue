<template>
  <div id="myMap"></div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';
import { LineGeometry } from 'three/examples/jsm/lines/LineGeometry.js';
import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial.js';
import { Line2 } from 'three/examples/jsm/lines/Line2.js';
import * as d3 from "d3-geo";
import chengduGeo from './assets/geoJson/chengdu.json'
import myFont from './assets/fonts/YouSheBiaoTiHei_Regular.json'
export default {
  name: 'NbmapModelZk', // vue component name
  props:{
    options:{
      type:Object,
      default:()=>{
        return {}
      }
    }
  },
  data(){
    return{
      // Canvas
      initBgColor:0xb9d3ff,//地图背景色
      initBgOpacity:1,//地图背景透明度
      mapWidth:1000,//地图宽度
      mapHeight:500,//地图高度
      isBeFull:true,//是否铺满（铺满则mapWidth和mapHeight无效）
      //灯光
      lights:[
        {
          position:[0, 0, 800],
          color:0xffffff
        },
        {
          position:[0, 0, -800],
          color:0xffffff
        },
        {
          position:[800, 0, 0],
          color:0xffffff
        },
        {
          position:[800, 800, 0],
          color:0xffffff
        },
        {
          position:[0, 800, 0],
          color:0xffffff
        },
        {
          position:[-800, 800, 0],
          color:0xffffff
        },
        {
          position:[-800, 0, 0],
          color:0xffffff
        },
        {
          position:[-800, -800, 0],
          color:0xffffff
        },
        {
          position:[0, -800, 0],
          color:0xffffff
        },
        {
          position:[800, -800, 0],
          color:0xffffff
        },
      ],
      // 板块
      initMapThickness:30,// 地图厚度
      initMapColor:'#4189f2',// 地图默认颜色
      initMapOpacity:0.96,// 地图默认透明度
      initMapSideColor:'#184998',// 地图侧面默认颜色
      initMapSideOpacity:0.96,// 地图侧面默认透明度
      // 线条
      lineDiff:0.5, //线条与地图距离
      initLineColor:0x6becf5,// 线条默认颜色
      initLineOpacity:1,// 线条默认透明度
      chooseLineColor:0xffff00,// 选中模块线条颜色
      // 字体
      textDiff:0, // 字体与地图距离
      initTextHeight:7,// 字体厚度
      initTextSize:10,// 字体大小
      initTextRotateX:0,// 字体默认旋转
      initTextColor:0xffffff,// 字体默认颜色
      initTextSideColor:0x051b5f,// 字体侧面默认颜色
      //Three
      myMap:null,
      renderer:null,
      scene:null,
      camera:null,
      mapGroup:null,
      raycaster:null,
      pointer:null,
      intersects:[],
      chooseCounty:'',
      optionsKeys:['initBgColor','initBgOpacity','mapWidth','mapHeight','isBeFull','lights','initMapThickness','initMapColor','initMapOpacity','initMapSideColor','initMapSideOpacity',
      'lineDiff','initLineColor','initLineOpacity','chooseLineColor','textDiff','initTextHeight','initTextSize','initTextRotateX','initTextColor','initTextSideColor']
    }
  },
  mounted(){
    this.initOptions()
    this.initMap()
  },
  destroyed(){
    this.myMap.removeEventListener('mousemove',this.onMapMouseMove)
    this.myMap.removeEventListener('click',this.onMapClick)
  },
  methods:{
    initOptions(){
      this.optionsKeys.forEach(key=>{
        if(this.options[key]) this[key] = this.options[key]
      })
    },
    renderMap(){
      const render = ()=> {
          // 通过摄像机和鼠标位置更新射线
          this.raycaster.setFromCamera( this.pointer, this.camera );
          // 计算物体和射线的焦点
          this.intersects = this.raycaster.intersectObjects( this.mapGroup.children );
          this.mapGroup.children.forEach(object3D=>{
            const materials = object3D.children.filter(i=>i.name==="countyMap").map(i=>i.material[0])
            materials.forEach(item=>item.color.set(item.userData.initColor))
          })
          this.intersects.some((focus)=>{
            if(focus.object.name === "countyMap"){
              const materials = this.mapGroup.children.filter(i=>i.name === focus.object.parent.name)[0].children.filter(i=>i.name==="countyMap").map(i=>i.material[0])
              materials.forEach(item=>item.color.set(this.initLineColor))
              return true
            }
          })
          this.renderer.render(this.scene, this.camera);
          requestAnimationFrame(render);
      }
      render();
    },
    resizeHandle(){
      // 页面缩放事件监听
      window.addEventListener('resize', () => {
        if(this.isBeFull){
          this.mapWidth = this.myMap.offsetWidth
          this.mapHeight = this.myMap.offsetHeight
        }
        // 更新渲染
        this.renderer.setSize(this.mapWidth, this.mapHeight);
        // 更新相机
        this.camera.aspect = this.mapWidth / this.mapHeight;
        this.camera.updateProjectionMatrix();
      });
    },
    addTexts(texts){
      const fontBlob = new Blob([JSON.stringify(myFont)], {
        type: "application/json"
      })
      const loader = new FontLoader();
      const textsGroup = new THREE.Group()
      textsGroup.name='textsGroup'
      loader.load(URL.createObjectURL(fontBlob),  ( font ) =>{
        texts.forEach(text=>{
          const geometry = new TextGeometry( text[0], {
            font: font,
            size: this.initTextSize,
            height: this.initTextHeight,
            curveSegments: 20,
            bevelThickness: 2,
            bevelSize: 1.5,
            bevelEnalbed: true
          });
          const materials = [
            new THREE.MeshPhongMaterial({ color: this.initTextColor, flatShading: true }),
            new THREE.MeshPhongMaterial({ color:this.initTextSideColor }),
          ];
          const textMesh = new THREE.Mesh(geometry, materials);
          const box3 = new THREE.Box3();
          box3.expandByObject(textMesh)
          var center = new THREE.Vector3()
          box3.getCenter(center)
          textMesh.position.set(text[1]-center.x,text[2]-center.y,text[3])
          textMesh.rotateX(this.initTextRotateX*Math.PI/180)
          textsGroup.add(textMesh)
        })
        this.scene.add(textsGroup);
      });
    },
    addPointLights(){
      const lightsGroup = new THREE.Group()
      lightsGroup.name='lightsGroup'
      this.lights.forEach(i=>{
        var point = new THREE.SpotLight(i.color)
        point.position.set(...i.position);
        lightsGroup.add(point)
      })
      this.scene.add(lightsGroup);
    },
    initMap(){
      // 获取dom
      this.myMap = document.querySelector('#myMap');
      // 定义渲染尺寸
      if(this.isBeFull){
        this.mapWidth = this.myMap.offsetWidth
        this.mapHeight = this.myMap.offsetHeight
      }
      // 初始化渲染器
      this.renderer = new THREE.WebGLRenderer();
      this.renderer.setSize(this.mapWidth, this.mapHeight);
      this.renderer.setPixelRatio(window.devicePixelRatio);
      this.renderer.setClearColor(this.initBgColor, this.initBgOpacity);
      this.myMap.appendChild(this.renderer.domElement)
      // 初始化场景
      this.scene = new THREE.Scene();
      // 光源
      this.addPointLights()
      // 相机
      this.camera = new THREE.PerspectiveCamera(30, this.mapWidth / this.mapHeight, 1, 3000);
      this.camera.position.set(-240, -400, 1400);
      this.camera.lookAt(0, 0, 0)
      // 相机控件
      const control = new OrbitControls(this.camera, this.renderer.domElement);
      control.enablePan = false  // 平移
      control.minDistance = 1000  // 相机距离观察点最小距离（最大状态）
      control.maxDistance = 2000  // 相机距离观察点最大距离（最小状态）

      // 光线投射
      this.raycaster = new THREE.Raycaster();
      this.pointer = new THREE.Vector2(99999,99999);
      // 添加成都地图
      this.addMap()
      // 添加交互
      this.addHandle()
      // 渲染
      this.renderMap()
      // 缩放
      this.resizeHandle()
    },
    addMap(){
      const texts = []
      this.mapGroup = new THREE.Group()
      this.mapGroup.name='mapGroup'
      this.scene.add(this.mapGroup);
      const projection = d3.geoMercator().center([104.080989, 30.657689]).scale(20000).translate([0, 0]);
      chengduGeo.features.forEach(elem => {
        const scaleZ = elem.properties.scaleZ||1
        const color = elem.properties.color||this.initMapColor
        const sideColor = elem.properties.sideColor||this.initMapSideColor
        // 定一个区县3D对象
        const county = new THREE.Object3D();
        county.name = elem.properties.name
        // 每个的 坐标 数组
        const { coordinates } = elem.geometry;
        let [textX, textY] =projection(elem.properties.center)
        texts.push([elem.properties.name,textX,-textY,this.initMapThickness*scaleZ+this.textDiff])
        // 循环坐标数组
        coordinates.forEach(multiPolygon => {
            multiPolygon.forEach((polygon) => {
                const shape = new THREE.Shape();
                let path = []  // 轨迹路径
                let path1 = []  // 轨迹路径(选中)
                for (let i = 0; i < polygon.length; i++) {
                    let [x, y] = projection(polygon[i]);
                    path = path.concat([x,-y,this.initMapThickness*scaleZ+this.lineDiff])
                    path1 = path1.concat([x,-y,this.initMapThickness*scaleZ+this.lineDiff+2])
                    if (i === 0) {
                        shape.moveTo(x, -y);
                    }
                    shape.lineTo(x, -y);
                }
                path = path.concat([path[0],path[1],path[2]])
                path1 = path1.concat([path1[0],path1[1],path1[2]])
                const vertices = new Float32Array(path); // 顶点
                const attribue = new THREE.BufferAttribute(vertices, 3); 
                const geometryLine = new THREE.BufferGeometry();
                const geometryLine1 = new LineGeometry();
                geometryLine.attributes.position = attribue;
                geometryLine1.setPositions( path1);
                const extrudeSettings = {
                    depth: 30,
                    bevelEnabled: true,
                    bevelSegments: 1,
                    bevelThickness: 0.2
                };
                const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
                // 平面部分材质
                const material = new THREE.MeshPhongMaterial( {
                    color: color,
                    side: THREE.DoubleSide,// FrontSide默认只有正面可见 BackSide只有背面可见 DoubleSide两面可见
                    transparent:true,//开启透明
                    opacity:this.initMapOpacity//设置透明度 
                } );
                material.userData.initColor = color
                // 拉高部分材质
                const material1 = new THREE.MeshPhongMaterial( {
                    color: sideColor,
                    side: THREE.DoubleSide,// FrontSide默认只有正面可见 BackSide只有背面可见 DoubleSide两面可见
                    transparent:true,//开启透明
                    opacity:this.initMapSideOpacity//设置透明度 
                } );
                const mesh = new THREE.Mesh(geometry, [
                    material,
                    material1,
                ]);
                mesh.name='countyMap'
                // 线 (直接画线)
                const material2 = new THREE.LineBasicMaterial({
                    color: this.initLineColor, //线条颜色
                    linewidth: 1,
                    transparent:true,//开启透明
                    opacity:this.initLineOpacity//设置透明度 
                });
                const material3 = new LineMaterial({
                    color: this.chooseLineColor, //线条颜色
                    linewidth: 3,
                    visible:false
                });
                material3.resolution.set(this.mapWidth,this.mapHeight);
                const line = new THREE.LineLoop(geometryLine, material2);
                const line1 = new Line2(geometryLine1, material3);
                line1.name='countyline'
                // 设置高度将省区分开来
                mesh.scale.set(1, 1,scaleZ);
                mesh._color = color
                mesh.position.set(0,0,0)
                county.add(mesh);
                county.add(line);
                county.add(line1);
            })
        })
        this.mapGroup.add(county);
      })
      this.addTexts(texts)
    },
    onMapMouseMove(e){
      this.pointer.x = ( e.clientX / this.mapWidth ) * 2 - 1;
      this.pointer.y = - ( e.clientY /this.mapHeight ) * 2 + 1;
    },
    onMapClick(){
      if(!this.intersects.length) return
      const chooseCounty = this.intersects[0].object.parent.name
      // 控制给哪个模型描边
      if(this.chooseCounty===chooseCounty){
        const materialsChoose = this.mapGroup.children.filter(i=>i.name === chooseCounty)[0].children.filter(i=>i.name==="countyline").map(i=>i.material)
        materialsChoose.forEach(item=>item.visible=false)
        this.chooseCounty=''
      }else{
        // 改轮廓线
        this.mapGroup.children.forEach(object3D=>{
          const materials = object3D.children.filter(i=>i.name==="countyline").map(i=>i.material)
          materials.forEach(item=>item.visible=false)
        })
        const materialsChoose = this.mapGroup.children.filter(i=>i.name === chooseCounty)[0].children.filter(i=>i.name==="countyline").map(i=>i.material)
        materialsChoose.forEach(item=>item.visible=true)
        this.chooseCounty = chooseCounty
      }
    },
    addHandle(){
      this.myMap.addEventListener('mousemove',this.onMapMouseMove)
      this.myMap.addEventListener('click',this.onMapClick)
    }
  }
}
</script>

<style>
#myMap{
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
}
</style>