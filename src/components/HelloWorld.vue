<template>
  <div id="container">

  </div>
</template>

<script>
  import * as THREE from "three";
  import {DRACOLoader} from 'three/examples/jsm/loaders/DRACOLoader';
  import {GLTFLoader} from 'three/examples/jsm/loaders/GLTFLoader';
  import TWEEN from "@tweenjs/tween.js"; // 动画

  const OrbitControls = require('three-orbit-controls')(THREE);

  export default {
    name: 'HelloWorld',
    data() {
      return {}
    },
    mounted() {
      this.init();
      // this.addSphere();
      // this.addObj();
      this.addModel();
      // this.animate();
    },
    beforeDestroy() {
      this.scene = null;
      this.light = null;
      this.camera = null;
      this.renderer = null;
      this.particleSystem = null;
      cancelAnimationFrame(this.req);
    },
    methods: {
      init() {

        this.scene = null;
        this.light = null;
        this.camera = null;
        this.renderer = null;
        this.particleSystem = [];
        this.scene = new THREE.Scene();

        this.fov = 60;
        // this.light = new THREE.AmbientLight(0xffffff);
        this.light = new THREE.AmbientLight(0x000000);
        this.scene.add(this.light);

        this.clock = new THREE.Clock();

        //初始化相机

        this.camera = new THREE.PerspectiveCamera(this.fov, window.innerWidth / window.innerHeight, 0.1, 10000);
        this.camera.position.set(100, 100, 100);


        //渲染
        this.renderer = new THREE.WebGLRenderer({
          alpha: true
        });//会在body里面生成一个canvas标签,
        this.renderer.setPixelRatio(window.devicePixelRatio);//为了兼容高清屏幕
        this.renderer.setClearColor(new THREE.Color(0x303030));
        this.renderer.setSize(window.innerWidth, window.innerHeight);


        // 参数设置坐标轴大小,150，编写程序的时候，可以根据相机参数调整为合适的值，如果太小可以无法显示出来
        //const axesHelper = new THREE.AxesHelper(2100);
        //this.scene.add(axesHelper);


        // 初始化控制器
        this.controls = new OrbitControls(this.camera, this.renderer.domElement); // 轨道控制器
        this.controls.target.set(0, 0, 0);
        // this.controls.minDistance = 80;  // 加了建筑物就非常小
        // this.controls.maxDistance = 600;
        this.controls.maxPolarAngle = Math.PI;
        this.controls.update();


        const container = document.getElementById('container');
        container.appendChild(this.renderer.domElement);


        window.addEventListener('resize', this.onWindowResize, false);//添加窗口监听事件（resize-onresize即窗口或框架被重新调整大小）

      },
      onWindowResize() {
        this.camera.aspect = window.innerWidth / window.innerHeight;
        this.camera.updateProjectionMatrix();
        this.renderer.setSize(window.innerWidth, window.innerHeight);
      },
      addSphere() {
        var geom = new THREE.SphereGeometry(10, 30, 20);
        var mate = new THREE.ShaderMaterial({
          vertexShader: `
  varying vec3 vNormal;
  void main() {
        //将attributes的normal通过varying赋值给了向量vNormal
    vNormal = normal;
        //projectionMatrix是投影变换矩阵 modelViewMatrix是相机坐标系的变换矩阵 最后我们将y值乘以1.4得到了一个形如鸡蛋的几何体
    gl_Position = projectionMatrix * modelViewMatrix * vec4( position.x, position.y * 1.4, position.z, 1.0 );
  }
  `,
          fragmentShader: `
    //片元着色器同样需要定义varying vec3 vNormal；
  varying vec3 vNormal;
  void main() {
        //vNormal是一个已经归一化的三维向量
    float pr = (vNormal.x + 1.0) / 2.0; //pr红色通道值范围为0~1
    float pg = (vNormal.y + 1.0) / 2.0; //pg绿色通道值范围为0~1
    float pb = (vNormal.z + 1.0) / 2.0; //pb蓝色通道值范围为0~1
    gl_FragColor=vec4(pr, pg, pb, 1.0); //最后设置顶点颜色，点与点之间会自动插值
  }
  `
        });
        let mesh = new THREE.Mesh(geom, mate);
        this.scene.add(mesh);
      },
      addModel() {
        let self = this;

        let uniforms = {
          // 顶点颜色
          color: {
            type: 'v3',
            // value: new THREE.Color(0x0000ff)
            value: new THREE.Color(0xffffff)
          },
          // 传递顶点贴图
          colorMap: {
            value: self.getTexture()
          },
          // 传递val值，用于shader计算顶点位置
          val: {
            value: 1.0
          }
        };

        // console.log(self.getTexture());

        // 着色器材料
        let shaderMaterial = new THREE.ShaderMaterial({
          uniforms: uniforms,

          vertexShader: `

           attribute float size;
           // 颜色透明度
           varying float opacity;
           varying vec3 vColor;
          void main(){
                // 开始产生模糊的z轴分界
                 float border = -150.0;
                // 最模糊的z轴分界
                float min_border = -160.0;
                // 最大透明度
                float max_opacity = 1.0;
                // 最小透明度
                float min_opacity = 0.03;
                // 模糊增加的粒子尺寸范围
                float sizeAdd = 4.0;

                vec4 mvPosition = modelViewMatrix * vec4(position, 1.0 );
                // z轴坐标越小越模糊，即越远越模糊
              if(mvPosition.z > border){
                    opacity = max_opacity;
                    gl_PointSize = size;
                }else if(mvPosition.z < min_border){
                   opacity = min_opacity;
                   gl_PointSize = size + sizeAdd;
                }else{
                      // 模糊程度随距离远近线性增长
                    float percent = (border - mvPosition.z)/(border - min_border);
                    opacity = (1.0-percent) * (max_opacity - min_opacity) + min_opacity;
                    gl_PointSize = percent * (sizeAdd) + size;
        }

        float positionY = position.y;

         //  根据y轴坐标计算传递的顶点颜色值
         vColor.x = abs(sin(positionY));
         vColor.y = abs(cos(positionY));
         vColor.z = abs(cos(positionY));

        gl_Position = projectionMatrix * mvPosition;
}

          `,
          //  vertexShader: `
          //   // 颜色透明度
          //    attribute float size;
          //    varying float opacity;
          //    void main() {
          //    // gl_PointSize = 1.;
          //    // gl_PointSize = size + 2. * sin(position.x / 4.);
          //     gl_PointSize = size;
          //     gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          //     if(gl_Position.x > 0.0){
          //         opacity = 0.1;
          //     }else{
          //        opacity= 1.0 ;
          //     }
          // }
          // `,
          fragmentShader: `
          uniform vec3 color;
          varying float opacity;
          uniform sampler2D colorMap;
          varying vec3 vColor;
          void main(){
               gl_FragColor = vec4(vColor * color, opacity);
             // 顶点颜色应用上2D纹理  不能使用text命名
           //  gl_FragColor =gl_FragColor * texture2D(colorMap, gl_PointCoord );
         }`,
          blending: THREE.AdditiveBlending,
          depthTest: false,// 这个不设置的话，会导致带透明色的贴图始终会有方块般的黑色背景
          transparent: true
        });


        //导入材质
        // let material = new THREE.PointsMaterial({
        //   color: 0xffffff,
        //   transparent: true,
        //   opacity: 1,
        //   size: 0.5,//可自由修改看看效果
        //   blending: THREE.AdditiveBlending,
        //   map: self.generateSprite()//自定义画布图案来充当每一个粒子的材质
        // });

        const dracoLoader = new DRACOLoader();
        dracoLoader.setDecoderPath("../../../static/draco/"); // 最后要加/
        dracoLoader.preload();


        new GLTFLoader().setDRACOLoader(dracoLoader).load(
          // '/static/model/bootboot.glb',
          '/static/model/p1.glb',
          // '/static/model/ZDHGDD.glb',
          // '/static/model/shanghaishanghai.fbx.glb',
          // '/static/model/city.glb',
          (gltf) => {

            let model = gltf.scene;
            model.scale.set(0.1, 0.1, 0.1);

            let index = 0;
            model.traverse(item => {
              if (item instanceof THREE.Mesh) {

                // let geometry = new THREE.Geometry().fromBufferGeometry(item.geometry);
                let geometry = item.geometry;

                console.log(geometry);


                let loadGeometry = geometry.clone();//创建该geometry的克隆体，后面会用到
                // let points = new THREE.Points(geometry, material);
                // let points = new THREE.Points(geometry, shaderMaterial);
                // self.particleSystem[`${index}`] = new THREE.Points(geometry, shaderMaterial);
                let particleSystem = new THREE.Points(geometry, shaderMaterial);
                // console.log(loadGeometry);

                let posSrc = {pos: 1};//创建一个posSrc的对象，该对象里面有pos的属性，并初始化该属性为0
                let tween = new TWEEN.Tween(posSrc).to({pos: 0}, 2000);//创建tween的补间动画，使posSrc中的pos属性的值在5000ms内从0到1变化
                tween.easing(TWEEN.Easing.Sinusoidal.InOut).onComplete(completeCallBack)

                ;//配置缓动效果
                let tweenStand = new TWEEN.Tween(posSrc).to({pos: 1}, 2000);//让动画在pos的值变为1后停止一段时间，方便我们观察，所以再创建一个tween，让pos从1到1（即不变）
                tween.easing(TWEEN.Easing.Sinusoidal.InOut);//配置缓动效果
                let tweenBack = new TWEEN.Tween(posSrc).to({pos: 1}, 2000);//创建tweenBack的补间动画，和初始相反，使posSrc中的pos属性的值在5000ms内从1到0变化
                tweenBack.easing(TWEEN.Easing.Sinusoidal.InOut).onComplete(completeCallBack);//配置缓动效果
                //每一个补间动画之间使用chain()连接起来
                tween.chain(tweenStand);
                tweenStand.chain(tweenBack);
                tweenBack.chain(tween);


                let length = loadGeometry.attributes.position.array.length;
                let sizes = new Float32Array(length);
                for (let i = 0; i < length; i++) {
                  sizes[i] = 0.5;
                }

                // 挂载属性值
                geometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));
                let color, nextcolor;

                //在补间的过程中，让所有的粒子开始移动
                let onUpdate = function () {
                  let pos = posSrc.pos;//定义一个pos，赋值为posSrc对象的pos属性

                  if (color) {
                    let uColor = particleSystem.material.uniforms.color.value;

                    uColor.r = color.r + (nextcolor.r - color.r) * pos;
                    uColor.b = color.b + (nextcolor.b - color.b) * pos;
                    uColor.g = color.g + (nextcolor.g - color.g) * pos;
                  }


                  for (let i = 0; i < length; i++) {
                    // geometry.attributes.position.array[i] = loadGeometry.attributes.position.array[i] * pos;
                  }
                  geometry.attributes.position.needsUpdate = true;
                };
                //tween在每次更新后会执行tween.onUpdate()函数，里面的参数就是我们自定义要让它如果去运动的函数，即上面写的onUpdate
                tween.onUpdate(onUpdate);
                tweenStand.onUpdate(onUpdate);
                tweenBack.onUpdate(onUpdate);

                tween.start();//开启tween


                // 每轮动画完成时的回调函数

                function completeCallBack() {
                  let uColor = particleSystem.material.uniforms.color.value;

                  // 保存旧的粒子颜色
                  color = {
                    r: uColor.r,
                    b: uColor.b,
                    g: uColor.g
                  };

                  // 随机生成将要变换后的粒子颜色
                  nextcolor = {
                    r: Math.random(),
                    b: Math.random(),
                    g: Math.random()
                  }
                }

                self.particleSystem[`${index}`] = particleSystem;
                self.scene.add(self.particleSystem[`${index}`]);
                index++;

                // flag = true;
              }
            });

            // console.log(this.particleSystem);
            self.animate();


          },
          (progressEvent) => {


          },
          (error) => {
            // called when loading has errors
            console.error('An error happened', error);
          }
        );
      },
      animate() {

        let detal = this.clock.getDelta();
        this.controls.update(detal); // 更新控制器


        TWEEN.update(); // 补间动画
        let time = Date.now() * 0.005;
        this.particleSystem.forEach(item => {

          let bufferObj = item.geometry;
          // 粒子系统缓缓旋转
          item.rotation.y = 0.01 * time;
          let sizes = bufferObj.attributes.size.array;
          let len = sizes.length;
          for (let i = 0; i < len; i++) {
            sizes[i] = 1.5 * (2.0 + Math.sin(0.02 * i + time));
          }
          // 需指定属性需要被更新
          bufferObj.attributes.size.needsUpdate = true;

        });

        // console.log(TWEEN.getAll());

        this.render();
        this.req = requestAnimationFrame(this.animate);
      },
      render() {
        this.renderer.render(this.scene, this.camera);
      },
      //自定义渐变颜色的画布s
      generateSprite() {

        let canvas = document.createElement('canvas');
        canvas.width = 16;
        canvas.height = 16;

        let context = canvas.getContext('2d');
        let gradient = context.createRadialGradient(canvas.width / 2, canvas.height / 2, 0, canvas.width / 2, canvas.height / 2, canvas.width / 2);
        gradient.addColorStop(0, 'rgba(255,255,255,1)');
        gradient.addColorStop(0.2, 'rgba(0,255,255,1)');
        gradient.addColorStop(0.4, 'rgba(0,0,255,1)');
        gradient.addColorStop(1, 'rgba(0,0,0,1)');

        context.fillStyle = gradient;
        context.fillRect(0, 0, canvas.width, canvas.height);

        let texture = new THREE.Texture(canvas);
        texture.needsUpdate = true;
        return texture;

      },
      getTexture() {
        let canvasSize = 64;
        let canvas = document.createElement('canvas');
        canvas.width = canvasSize;
        canvas.height = canvasSize;
        canvas.style.background = "transparent";
        let context = canvas.getContext('2d');
        let gradient = context.createRadialGradient(canvas.width / 2, canvas.height / 2, canvas.width / 8, canvas.width / 2, canvas.height / 2, canvas.width / 2);
        gradient.addColorStop(0, '#fff');
        gradient.addColorStop(1, 'transparent');
        context.fillStyle = gradient;
        context.beginPath();
        context.arc(canvas.width / 2, canvas.height / 2, canvas.width / 2, 0, Math.PI * 2, true);
        context.fill();
        let texture = new THREE.Texture(canvas);
        texture.needsUpdate = true;
        return texture;
      }

    }
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  #container >>> canvas {
    display: block;
  }
</style>
