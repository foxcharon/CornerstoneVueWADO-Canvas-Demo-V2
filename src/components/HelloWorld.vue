<template>
  <div>
    <div class="image-canvas-wrapper"  oncontextmenu="return false" unselectable='on' onselectstart='return false;'
      onmousedown='return false;'>
      <!-- DICOM CANVAS -->
        <span id="loadProgress">Diocm加载: </span>
        <!-- <div> -->
          <div ref="canvas" class="image-canvas" oncontextmenu="return false" tabindex="-1"></div>
        <!-- </div> -->
    </div>

    <!-- fixed div -->
    <div class="fixed">
      <select v-model="selectValueString">
        <option value="0">无</option>
        <option value="1">多边形</option>
        <option value="2">矩形</option>
        <option value="3">椭圆</option>
      </select>
      <div>
        <button @click="clearButtonFunc">clear!!!</button>
      </div>
      <div>
        <p>图片放大缩小</p>
        <button @click="zoomPlusFunc"> + </button> <button @click="zoomSubFunc"> - </button>
        <p>移动单位<count @changeCount="changeCountFunc($event)" :count="directionStandandValueNumber"/></p>
        <p>图片位置控制</p>
        <div class="direction-control">
          <div></div>
          <div @click="directionTopFunc"></div>
          <div></div>
          <div @click="directionLeftFunc"></div>
          <div></div>
          <div @click="directionRightFunc"></div>
          <div></div>
          <div @click="directionBottomFunc"></div>
          <div></div>
        </div>
      </div>
      <!-- <button @click="submitTestFunc">selectValueString</button>
      <button @click="clearAllDataFunc">clear</button>
      <button @click="moveTestFunc">move</button> -->
      <p>颜色滤镜</p>
      <select v-model="selectColorFilterString">
        <option value="0">正常</option>
        <option value="1">反色</option>
        <!-- <option value="2">火红</option>
        <option value="3">彩虹色</option> -->
      </select>
      <!-- <button @click="addBrightFunc">变亮</button> <button @click="subBrightFunc">变暗</button> <button @click="resetBrightFunc">亮度复原</button>
      <br>
      <input type="radio" name="sex" value="" checked @click="brightRadioTypeNumberFunc(0)">一般 <input type="radio" name="sex" value="" @click="brightRadioTypeNumberFunc(1)">变亮 <input type="radio" name="sex" value="" @click="brightRadioTypeNumberFunc(2)">变暗
      <div>
        <p>此时视图素材宽高：{{ canvasSizeObject ? canvasSizeObject.width : ""}} {{ canvasSizeObject ? canvasSizeObject.height : "" }}</p>
        <p>此时放大级别：{{ zoomNumberArray[zoomIndexNumber] + "(" + zoomIndexNumber + ")"}}</p>
      </div> -->
    </div>

    <!-- 隐藏的canvas 放大缩小时需要用到 start -->
    <canvas id="hide-canvas" style="display:none"></canvas>
    <!-- end -->
  </div>

</template>

<script>

//引入 cornerstone,dicomParser,cornerstoneWADOImageLoader
import * as cornerstone from "cornerstone-core";
// import * as dicomParser from "dicom-parser";

// 不建议 npm 安装 cornerstoneWADOImageLoader 如果你做了 会很头疼
import * as cornerstoneWADOImageLoader from "../../static/dist/cornerstoneWADOImageLoader.js";

// Cornerstone 工具外部依赖
import Hammer from "hammerjs";
import * as cornerstoneMath from "cornerstone-math";
import * as cornerstoneTools from "cornerstone-tools";

// Specify external dependencies
cornerstoneTools.external.Hammer = Hammer;
cornerstoneTools.external.cornerstone = cornerstone;
cornerstoneTools.external.cornerstoneMath = cornerstoneMath;
cornerstoneWADOImageLoader.external.cornerstoneMath = cornerstoneMath;

//指定要注册加载程序的基石实例
cornerstoneWADOImageLoader.external.cornerstone = cornerstone;

cornerstone.registerImageLoader("http", cornerstoneWADOImageLoader.loadImage);
cornerstone.registerImageLoader("https", cornerstoneWADOImageLoader.loadImage);

//配置 webWorker (必须配置)
//注意这里的路径问题  如果路径不对 cornerstoneWADOImageLoaderWebWorker 会报错 index.html Uncaught SyntaxError: Unexpected token <
// 这个是 dev 环境的路径，如果在 production 环境项目不在服务器根目录，会找不到文件
var config = {
  webWorkerPath: "/static/dist/cornerstoneWADOImageLoaderWebWorker.js",
  taskConfiguration: {
    decodeTask: {
      codecsPath: "/static/dist/cornerstoneWADOImageLoaderCodecs.js"
    }
  }
};
// 这个是 production 环境的路径，请根据具体情况进行设置
// const path = require('path')
// var config = {
//   webWorkerPath: path.resolve('cornerstone-demo') + "/static/dist/cornerstoneWADOImageLoaderWebWorker.js",
//   taskConfiguration: {
//     decodeTask: {
//       codecsPath: path.resolve('cornerstone-demo') + "/static/dist/cornerstoneWADOImageLoaderCodecs.js"
//     }
//   }
// };
cornerstoneWADOImageLoader.webWorkerManager.initialize(config);
var _this = null

// 组件
import count from "./count";
export default {
  name: "HelloWorld",
  data() {
    return {
      baseUrl: "", 
      // 这里的DCM放在了后端服务器中，不是前端运行环境的那个localhost 
      exampleStudyImageIds: [
        'http://localhost/bbmri-53323851.dcm',
        // 'http://localhost/bbmri-53323707.dcm',
        // 'http://localhost/bbmri-53323851.dcm',
        // 'http://localhost/bbmri-53323707.dcm'
      ],
      isInitLoad: true,
      isShow: true,
      // EDIT ===>>> 20190109
      // obj
      canvasObject:null,    // 图元用CanvasRenderingContext2D
      canvasSizeObject:null,    // 图元用 canvas宽高
      canvasOriginSizeObject:null,    // 图元用 canvas宽高 origin
      canvasOriginDataObject:null,    // 图元用 canvasImageData origin
      canvasPaintingObject:null,   // 绘制用 CanvasRenderingContext2D
      canvasOriginFilterDataObject:null,   // CanvasRenderingContext2D-imgdata-origin 不带滤镜带亮度
      canvasOriginNormalDataObject:null,   // CanvasRenderingContext2D-imgdata-origin 不带滤镜不带亮度
      canvasOriginZoomDataObject:null,   // CanvasRenderingContext2D-imgdata-with-zoom-origin
      polygonMovingMouseStartedCoordinateObject:null,    // 鼠标移动多边形 前一次的坐标
      polygonMovingMouseContinueCoordinateObject:null,    // 鼠标移动多边形 现在的坐标
      polygonTransformingMouseStartedCoordinateObject:null,    // 鼠标拉伸多边形 前一次的坐标
      polygonTransformingMouseContinueCoordinateObject:null,    // 鼠标拉伸多边形 现在的坐标
      keyEventObject:null,    // 键盘事件对象
      // displayModel dataModel submitModel
      // XY坐标原型值（如果没有点任何方向键，此时XY坐标值是多少）
      prototypeXYCoordinateObject:{x:0,y:0},
      // 此时源图视窗起始点（左上角）的XY坐标值 - XY坐标原型值
      viewXYCoordinateObject:{x:0,y:0},
      // 此时源图视窗起始点（左上角）的XY坐标值
      scrollViewCoorObject:{x:0,y:0},
      // 记忆点过的方向键的对象
      directionMemoryObject:{x:0,y:0},
      // 记录此时滚轮值的对象
      scrollBaseObject:{
        deltaY:0,
        offsetX:0,
        offsetY:0
      },
      // EDIT END
      // string      
      selectValueString:"0",    // 选择模式
      selectColorFilterString:"0",    // 选择滤镜 0 为正常
      // number
      sqrtStandandNumber:10,  // 过近点计算标准值
      minPointNumNumber:3,  // 最少要几个点
      arcRNumber:4,    // 圆半径
      lineWidthNumber:1,  // 直线线宽
      arcWidthNumber:2,    // 圆线宽
      centerPointHoverJudgeNumber: Math.sqrt(20*20+20*20),     // 是否接近中心点判定值
      polygonPointHoverJudgeNumber: Math.sqrt(10*10+10*10),     // 是否接近多边形连接点判定值
      canTranslatePolygonIndex:-1,    // 此时有鼠标处于几号多边形的中心点附近
      brightRadioTypeNumber:0,   // 控制此时可否增减亮度
      zoomIndexNumber:0,    // 此时处于哪个放大倍数
      directionStandandValueNumber:10,    // 点1下移动几个像素
      // boolean
      isPointingBoolean:false,    // 是否正在绘制
      isPaintedBoolean:false,    // 是否已经完成过绘制
      isCanCopyOriginCanvasDataBoolean:true,  // 是否可以记录canvas原数据
      isMouseCanTranslatePolygonBoolean:false,    // true 此时有鼠标处于某个多边形的中心点附近
      isMouseTranslatingPolygonBoolean:false,    // true 此时正在拖动某个多边形，依赖上一个变量
      isRectPaintingBoolean:false,    // 是否正在绘制矩形
      isEllipsePaintingBoolean:false,    // 是否正在绘制椭圆
      isMouseCanTranslatePolygonBoolean:false,    // true 此时有鼠标处于某个多边形的中心点附近
      isMouseTranslatingPolygonBoolean:false,    // true 此时正在拖动某个多边形，依赖上一个变量
      isMouseCanTransformPolygonBoolean:false,    // true 此时有鼠标处于某个多边形的连接点附近
      isMouseTransformingPolygonBoolean:false,    // true 此时正在拉伸某个多边形，依赖上一个变量
      polygonLatingAndFormingIsDisconnectedBoolean:false,    // 鼠标拖动或拉伸多边形时与多边形失去连接
      renderAfterZoomChangeStatusBoolean:true,    // 在读取完成之前
      keyEventBaseBoolean:true,    // 键盘是否生效
      scrollBaseBoolean:true,   // 滚轮是否生效，防止事件叠加导致计算错误
      // array
      canvasMarkDataArray:[],    // 记录canvas标注数据的数组 视图模型
      // 三个地方  添加 | 修改 | 删除
      anotherCanvasMarkDataArray:[],    // 记录canvas标注数据的数组 数据模型
      zoomNumberArray:[1, 2, 3, 4, 5],    // 放大倍数数组
    };
  },
  methods: {
    show() {
      const _this = this;
      if (this.isShow === true) {
        this.isShow = false;
        // this.$http
        //   .get("http://10.0.0.5:90/DoctorService/Service.asmx/CS_Dicom")
          // .then(function(res) {
            // console.log("res:", res);

            // let Image = res.body.value;
            // console.log("res.body.value:", res.body.value);

            // _this.baseUrl = res.body.value.filmain;
            // console.log("res.body.value.filmain:", res.body.value.filmain);

            // _this.exampleStudyImageIds = res.body.value.testDate.testDate1;
            // console.log(
            //   "res.body.value.testDate.testDate1:",
            //   res.body.value.filmain
            // );
            
            // 找到要渲染的元素
            let canvas = this.$refs.canvas;
            // console.log(canvas);
            // 在 DOM 中将 canvas 元素注册到 cornerstone
            cornerstone.enable(canvas);
            // 拼接 url : cornerstoneWADOImageLoader 需要 wadouri 路径头
            const imageUrl = _this.baseUrl + _this.exampleStudyImageIds[0];
            // const imageUrl = 'http://localhost/bbmri-53323851.dcm';
            let imageId = "wadouri:" + imageUrl;

            //  Load & Display
            cornerstone.loadAndCacheImage(imageId).then(
              function(image) {
                // console.log(image);
                // 设置元素视口
                var viewport = cornerstone.getDefaultViewportForImage(
                  canvas,
                  image
                );
                // 显示图像
                cornerstone.displayImage(canvas, image, viewport);
                // 激活工具
                _this.initCanvasTools();

                // console.log("display")
              },
              function(err) {
                alert(err);
              }
            );

            // Dicom 加载 进度
            cornerstone.events.addEventListener(
              "cornerstoneimageloadprogress",
              function(event) {
                const eventData = event.detail;
                const loadProgress = document.getElementById("loadProgress");
                loadProgress.textContent = `Dicom加载: ${
                  eventData.percentComplete
                }%`;
                if (eventData.percentComplete === 100) {
                  // console.log("complete")
                }
              }
            );
          // });
      } else {
        this.isShow = true;
      }
    },
    initCanvasTools() {
      let _self = this;
      let canvas = this.$refs.canvas;
      this.isInitLoad = false;

      // 为 canvasStack 找到 imageIds
      let allImageIds = [];
      // this.exampleStudyImageIds.forEach(function(imageId) {
      //   let imageUrl = "wadouri:" + _self.baseUrl + imageId;
      //   allImageIds.push(imageUrl);
      // });
      // this.exampleStudyImageIds = [
      //   'http://localhost/bbmri-53323851.dcm',
      //   'http://localhost/bbmri-53323707.dcm',
      //   'http://localhost/bbmri-53323851.dcm',
      //   'http://localhost/bbmri-53323707.dcm'
      // ]
      this.exampleStudyImageIds.forEach(function(imageId) {
        let imageUrl = "wadouri:" + imageId;
        allImageIds.push(imageUrl);
      });

      // Create canvasStack
      let canvasStack = {
        currentImageIdIndex: 0,
        imageIds: allImageIds
      };

      // Enable Inputs
      // cornerstoneTools.mouseInput.enable(canvas);
      // cornerstoneTools.mouseWheelInput.enable(canvas);
      // cornerstoneTools.touchInput.enable(canvas);

      // Set the stack as tool state
      cornerstoneTools.addStackStateManager(canvas, ["stack"]);
      cornerstoneTools.addToolState(canvas, "stack", canvasStack);
      cornerstoneTools.stackScrollWheel.activate(canvas); // Mouse wheel
      cornerstoneTools.scrollIndicator.enable(canvas); // Position indicator

      // EDIT

      // Mouse
      // cornerstoneTools.wwwc.activate(canvas, 1); // left click
      cornerstoneTools.pan.activate(canvas, 2); // middle click
      // cornerstoneTools.zoom.activate(canvas, 4); // right click

      // Touch / Gesture
      // cornerstoneTools.wwwcTouchDrag.activate(canvas); // - Drag
      // cornerstoneTools.zoomTouchPinch.activate(canvas); // - Pinch
      // cornerstoneTools.panMultiTouch.activate(canvas); // - Multi (x2)

      this.extra()

      // EDIT END
    },
    /*
       * Window Resize
       *
       */
    listenForWindowResize() {
      this.$nextTick(function() {
        window.addEventListener(
          "resize",
          this.debounce(this.onWindowResize, 100)
        );
      });
    },
    onWindowResize() {
      cornerstone.resize(this.$refs.canvas, true);
    },
    /*
       * Utility Methods
       *
       */
    debounce(func, wait, immediate) {
      var timeout;
      return function() {
        var context = this;
        var args = arguments;
        var later = function() {
          timeout = null;
          if (!immediate) func.apply(context, args);
        };
        var callNow = immediate && !timeout;
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
        if (callNow) func.apply(context, args);
      };
    },
    // EDIT 
    extra(){
      // this.$refs.canvas.setAttribute("width", this.$ref.canvas.getAttribute("height"))
      // this.$refs.canvas.width = this.$refs.canvas.height
      const c = document.getElementsByTagName("canvas")[0]
      this.canvasObject = c.getContext('2d')
      this.canvasSizeObject = {
        width:c.width,
        height:c.height
      }
      this.canvasOriginSizeObject = {
        width:c.width,
        height:c.height
      }
      this.repeatGetCanvasDataFunc()
      // 滚轮事件
      this.$Tools.scrollCommonFunction(window, document)
      const _this = this
      window.addWheelListener(this.$refs.canvas, function( e ) {
        if (!_this.scrollBaseBoolean) {
          return
        }
        _this.scrollBaseObject = {
          deltaY:e.deltaY,
          offsetX:e.offsetX,
          offsetY:e.offsetY
        }
        _this.scrollBaseBoolean = false
        window.setTimeout(()=>{
          // deltaY < 0 放大 deltaY > 0 缩小
          if (_this.scrollBaseObject.deltaY < 0) {
            _this.scrollDriveCanvasZoomCalc(_this.scrollBaseObject.deltaY, _this.scrollBaseObject.offsetX, _this.scrollBaseObject.offsetY)
          } else {
            _this.scrollDriveCanvasZoomCalc(_this.scrollBaseObject.deltaY, _this.scrollBaseObject.offsetX, _this.scrollBaseObject.offsetY)
          }          
          _this.scrollBaseBoolean = true
        },100)        
        e.preventDefault();
      })
    },    
    repeatGetCanvasDataFunc(){
      const _this = this
      const p = (resolve, reject) => {
        _this.canvasOriginFilterDataObject = _this.canvasObject.getImageData(0, 0, width, height)
        _this.canvasOriginNormalDataObject = _this.canvasObject.getImageData(0, 0, width, height)
        _this.canvasOriginZoomDataObject = _this.canvasObject.getImageData(0, 0, width, height)
        resolve('200 OK');
      }
      const width = this.canvasSizeObject.width, 
        height = this.canvasSizeObject.height;
      this.canvasOriginDataObject = this.canvasObject.getImageData(0, 0, width, height)      
      if (this.canvasOriginDataObject.data[3] !== 0) {
        const p_run = new Promise(p).then((result) => {})
        this.addPaintingCanvas(width, height)
        return
      } else {
        setTimeout(()=>{
          this.repeatGetCanvasDataFunc()
        },10)
      }
    },
    // new func ::: 加一个canvas对象
    addPaintingCanvas(width, height){
      document.querySelector(".cornerstone-canvas").style.zIndex = 1
      let HTML = document.createElement("canvas");
      HTML.width = width,
        HTML.height = height,
        HTML.id = "id_painting"
      document.querySelector(".image-canvas").style.position = "relative"
      document.querySelector(".image-canvas").appendChild(HTML)
      this.updateDOMStyle(HTML, {
        'z-index':2,
        'position':'absolute',
        'left':0,
        'top':0
      })
      // const c = document.getElementsById('id_painting')
      this.canvasPaintingObject = HTML.getContext('2d')
    },
    updateDOMStyle(DOM, obj){
      Object.keys(obj).forEach(function(key){
        console.log(key,obj[key]);
        DOM["style"][key] = obj[key];
      });
    },
    clearPaintingFunc(){
      const w = this.canvasOriginSizeObject.width,
        h = this.canvasOriginSizeObject.height;
      const ctx = this.canvasPaintingObject
      ctx.clearRect(0, 0, w, h)
    },
    // 重绘
    reDrawFunc(){
      const mark_array = this.canvasMarkDataArray
      const line_width = 1
      const line_color = "red"
      mark_array.forEach((item, index) => {
        // 绘制多边形
        if (item.type === "polygon") {
          drawArcBeforeFunc(item.pointDataArray, item.pointActiveIndex, item.type)
        }
        if (item.completed && item.type === "polygon") {
          this.$Painting_tools.drawPointLineFunc(this.canvasPaintingObject, item.pointDataArray, line_color, line_width)
          drawCenterArcBeforeFunc(item.centerPointObject, item.centerPointActive)
        }
        if (item.completed && item.centerPointObject && item.type === "polygon") {
          drawCenterArcBeforeFunc(item.centerPointObject, item.centerPointActive)
        }
        // 绘制矩形
        if (item.completed && item.type === "rectangle") {
          // 如果中心点在第三象限就不绘制
          if (this.$Tools.checkCenterPointInQuadrantFunc(item)) {
            return
          }
          drawRectBeforeFunc(item)
          drawArcBeforeFunc(item.pointDataArray, item.pointActiveIndex, item.type)
          drawCenterArcBeforeFunc(item.centerPointObject, item.centerPointActive)
        }
        // 绘制椭圆
        if (item.completed && item.type === "ellipse") {
          // 如果中心点在第三象限就不绘制
          if (this.$Tools.checkCenterPointInQuadrantFunc(item)) {
            return
          }
          drawEllipseBeforeFunc(item)
          drawArcBeforeFunc(item.pointDataArray, item.pointActiveIndex, item.type)
          drawCenterArcBeforeFunc(item.centerPointObject, item.centerPointActive)
        }
      })
    },
    clearButtonFunc(){
      this.clearPaintingFunc()
      this.reDrawFunc()
    },
    // 计数器
    changeCountFunc($event){
      this.directionStandandValueNumber = $event
    },
    // 放大图片入口
    zoomPlusFunc(){
      const limit = this.zoomNumberArray.length - 1
      const plus = x => x + 1;
      const max = x => x > limit ? limit : x;
      this.zoomIndexNumber = max(plus(this.zoomIndexNumber)) 
      this.calcNowSourceWidthHeightFunc()
    },
    // 缩小图片入口
    zoomSubFunc(){
      const limit = 0
      const sub = x => x - 1;
      const min = x => x < limit ? limit : x;
      this.zoomIndexNumber = min(sub(this.zoomIndexNumber))
      this.calcNowSourceWidthHeightFunc()
    },
    // 上方向键
    directionTopFunc(event){
      const number = this.$Tools.directionCommonValueFunc(event, this.directionStandandValueNumber)
      const change = obj => ({y:obj.y - number, x:obj.x})
      this.viewXYCoordinateObject = this.checkViewCoorRangeFunc(change(this.viewXYCoordinateObject))
      this.calcNowSourceWidthHeightFunc('up')
    },
    // 左方向键
    directionLeftFunc(event){
      const number = this.$Tools.directionCommonValueFunc(event, this.directionStandandValueNumber)
      const change = obj => ({y:obj.y, x:obj.x - number})
      this.viewXYCoordinateObject = this.checkViewCoorRangeFunc(change(this.viewXYCoordinateObject))
      this.calcNowSourceWidthHeightFunc('left')
    },
    // 右方向键
    directionRightFunc(event){
      const number = this.$Tools.directionCommonValueFunc(event, this.directionStandandValueNumber)
      const change = obj => ({y:obj.y, x:obj.x + number})
      this.viewXYCoordinateObject = this.checkViewCoorRangeFunc(change(this.viewXYCoordinateObject))
      this.calcNowSourceWidthHeightFunc('right')
    },
    // 下方向键
    directionBottomFunc(event){
      const number = this.$Tools.directionCommonValueFunc(event, this.directionStandandValueNumber)
      const change = obj => ({y:obj.y + number, x:obj.x})
      this.viewXYCoordinateObject = this.checkViewCoorRangeFunc(change(this.viewXYCoordinateObject))
      this.calcNowSourceWidthHeightFunc('down')
    },
    // 计算相关值
    calcNowSourceWidthHeightFunc(type){
      const range_obj = this.getDirectionNumberRange()
      // 获取初始宽高 放大倍数
      // 求差值坐标
      const w = this.canvasOriginSizeObject.width,
        h = this.canvasOriginSizeObject.height,
        dmo = this.$Tools.directionMemoryObjectRangeCheckFunc(this.directionMemoryObject, range_obj),
        standand = this.directionStandandValueNumber,
        zoom_number = this.zoomNumberArray[this.zoomIndexNumber];        
      let prototypeXYCoordinateObject = this.prototypeXYCoordinateObject,
        viewXYCoordinateObject = this.viewXYCoordinateObject,
        scrollViewCoorObject = this.scrollViewCoorObject;
      // 计算现在用的源图片宽高 源图片初始点坐标
      let now_w = parseInt(w / zoom_number),
        now_h = parseInt(h / zoom_number);
      // x_max_stand:w - now_w, y_max_stand:h - now_h 两倍原型值 prototypeValue2X
      // (w - now_w) / 2,  (h - now_h) / 2 原型值：没有点过方向键时xy的值
      prototypeXYCoordinateObject.x = parseInt((w - now_w) / 2)
      prototypeXYCoordinateObject.y = parseInt((h - now_h) / 2)
      // viewXYCoordinateObject range change
      viewXYCoordinateObject = this.checkViewCoorRangeFunc(viewXYCoordinateObject)
      // x y
      let x = prototypeXYCoordinateObject.x + viewXYCoordinateObject.x,
        y = prototypeXYCoordinateObject.y + viewXYCoordinateObject.y;
      let x1 = x, 
        y1 = y;
      scrollViewCoorObject.x = prototypeXYCoordinateObject.x + viewXYCoordinateObject.x
      scrollViewCoorObject.y = prototypeXYCoordinateObject.y + viewXYCoordinateObject.y
      // console.log("view", this.viewXYCoordinateObject.x)
      // console.log("scroll", this.scrollViewCoorObject.x)
      // console.log("prototype", this.prototypeXYCoordinateObject.x)
      // console.log(x1, y1, w, h, now_w, now_h)
      this.renderAfterZoomChange(x1, y1, w, h, now_w, now_h, "button")
    },
    // 滚轮放大图片计算 放大用
    // deltaY 滚轮值 offsetX 鼠标横坐标 offsetY 鼠标纵坐标
    scrollDriveCanvasZoomCalc(deltaY, offsetX, offsetY){
      // deltaY < 0 放大 deltaY > 0 缩小
      // 计算now_zoom_number
      let now_zoom_number;
      const prev_zoom_number = this.zoomIndexNumber,
        zoom_min = 0,
        zoom_max = this.zoomNumberArray.length - 1;
      // 如果已到最小还想缩小，或者已到最大还想变大，就拒绝
      if ((deltaY > 0 && prev_zoom_number === zoom_min)||(deltaY < 0 && prev_zoom_number === zoom_max)) {
        return
      }
      const plus = x => x + 1;
      const sub = x => x - 1;
      const range_max_func = x => x > zoom_max ? zoom_max : x;
      const range_min_func = x => x < zoom_min ? zoom_min : x;
      deltaY < 0 ? now_zoom_number = range_max_func(plus(prev_zoom_number)) : '';
      deltaY > 0 ? now_zoom_number = range_min_func(sub(prev_zoom_number)) : '';
      // 计算之后的值赋值给zoomIndexNumber
      this.zoomIndexNumber = now_zoom_number
      const zoom = this.zoomNumberArray[this.zoomIndexNumber];
      // x10 相对于1倍率的横坐标 y10 相对于1倍率的纵坐标
      // x20 相对于将要发生的倍率的横坐标 y20 相对于将要发生的倍率的纵坐标
      const data_model_coor = this.$Tools.getDataModelValueInScroll({
        x:offsetX,
        y:offsetY
      }, this.scrollViewCoorObject, this.zoomNumberArray[prev_zoom_number])
      const x10 = data_model_coor.x,
        y10 = data_model_coor.y;      
      const x20 = parseInt(x10 / zoom),
        y20 = parseInt(y10 / zoom);
      const x00 = x10 - x20,
        y00 = y10 - y20;
      const w = this.canvasOriginSizeObject.width,
        h = this.canvasOriginSizeObject.height,
        standand = this.directionStandandValueNumber;
      const now_w = parseInt(w / zoom),
        now_h = parseInt(h / zoom);
      // check
      let {x1, y1, x_max_stand, y_max_stand} = this.$Tools.checkXYCurrentFunc(x00, y00, w - now_w, h - now_h)
      // 记录此时的初始点坐标，方便下次使用
      this.scrollViewCoorObject = {
        x:x1,
        y:y1
      }
      // 计算原型坐标
      this.$Tools.calcPrototypeXYCoordinateObjectFunc(w, h, zoom, this.prototypeXYCoordinateObject)
      // 计算原型坐标和现坐标的偏差值 viewXYCoordinateObject
      const dmoOrigin = this.$Tools.calcSubAboutPrototypeAndNowCoorFunc(this.prototypeXYCoordinateObject, this.scrollViewCoorObject, this.viewXYCoordinateObject)
      // console.log("view", this.viewXYCoordinateObject.x)
      // console.log("scroll", this.scrollViewCoorObject.x)
      // console.log("prototype", this.prototypeXYCoordinateObject.x)
      // 调用渲染
      this.renderAfterZoomChange(x1, y1, w, h, now_w, now_h, 'scroll')
    },
    // 获取方向键对象取值范围
    getDirectionNumberRange(){
      const zoom_number = this.zoomNumberArray[this.zoomIndexNumber],
        standand_number = this.directionStandandValueNumber,
        w = this.canvasOriginSizeObject.width,
        h = this.canvasOriginSizeObject.height;
      let now_w = w / zoom_number,
        now_h = h / zoom_number;
      let proto_x = (w - now_w) / 2,
        proto_y = (h - now_h) / 2;
      let range_x = proto_x / standand_number,
        range_y = proto_y / standand_number;
      return {
        x:[-range_x, range_x],
        y:[-range_y, range_y]
      }
    },
    // 获取差值坐标取值范围
    checkViewCoorRangeFunc(obj){
      // console.log(obj)
      // console.log(obj.x)
      const standand = {
        x:[-this.prototypeXYCoordinateObject.x,this.prototypeXYCoordinateObject.x],
        y:[-this.prototypeXYCoordinateObject.y,this.prototypeXYCoordinateObject.y]
      }
      // console.log(standand['x'][0])
      obj.x < standand['x'][0] ? obj.x = standand['x'][0] : ''
      obj.x > standand['x'][1] ? obj.x = standand['x'][1] : ''
      obj.y < standand['y'][0] ? obj.y = standand['y'][0] : ''
      obj.y > standand['y'][1] ? obj.y = standand['y'][1] : ''
      return obj
    },
    // 渲染
    renderAfterZoomChange(x, y, w, h, now_w, now_h, ui_type){
      if (!this.renderAfterZoomChangeStatusBoolean) {
        return
      }
      this.renderAfterZoomChangeStatusBoolean = false      
      const _this = this
      // canvas html element
      const hideCanvasHTML = document.getElementById("hide-canvas")
      hideCanvasHTML.width = w,
        hideCanvasHTML.height = h;
      const hideCanvasObject = hideCanvasHTML.getContext('2d')
      hideCanvasObject.putImageData(this.canvasOriginDataObject, 0, 0)
      hideCanvasHTML.toBlob(function (e) {
        const reader = new FileReader()
        reader.readAsDataURL(e)
        reader.onload = function(e){
          let img = new Image()
          img.src = e.target.result
          img.id = "abc"
          img.style.display = "none"
          // img注入进DOM
          document.body.appendChild(img)
          img.onload = function () {
            let canvasHTML = document.getElementsByTagName("canvas")[0]
            let ctx = canvasHTML.getContext("2d")
            ctx.drawImage(img, x, y, now_w, now_h, 0, 0, w, h)
            _this.canvasSizeObject = {
              width: now_w,
              height: now_h
            }
            // 记录zoomOriginData
            _this.canvasOriginZoomDataObject = ctx.getImageData(0, 0, w, h)
            // 重计算displayModel
            _this.resetDisplayModelFunc(ui_type)
            // 重绘
            _this.clearButtonFunc()
            // img从DOM中移除
            document.body.removeChild(img)
            _this.renderAfterZoomChangeStatusBoolean = true
            // 滤镜渲染
            _this.watchDriveFilterChangeFunc(_this.canvasOriginZoomDataObject, _this.selectColorFilterString)
          }
        }
      })
    },
    // 重设显示模型的数据
    resetDisplayModelFunc(ui_type){
      const range_obj = this.getDirectionNumberRange()
      const pXYc = this.prototypeXYCoordinateObject,
        vXYc = this.viewXYCoordinateObject,
        dmo = this.$Tools.directionMemoryObjectRangeCheckFunc(this.directionMemoryObject, range_obj),
        standand = this.directionStandandValueNumber,
        zoom_number = this.zoomNumberArray[this.zoomIndexNumber];
      let pXYcWithOffset = {
        x:pXYc.x + vXYc.x,
        y:pXYc.y + vXYc.y
      }
      this.resetDisplayModelCoreFunc(pXYcWithOffset, zoom_number)
    },
    // 重设显示模型的数据 核心
    // dataModel -> displayModel
    resetDisplayModelCoreFunc(pXYcWithOffset, zoom_number){
      let canvasMarkDataArray = this.canvasMarkDataArray
      this.anotherCanvasMarkDataArray.forEach((item, index, this_arr) => {
        // 中心点 center
        const centerPointObject = item.centerPointObject
        // dataModel中这个点与所知坐标的差值
        const no_mult_coor_center = {
          center_x:centerPointObject.center_x - pXYcWithOffset.x,
          center_y:centerPointObject.center_y - pXYcWithOffset.y
        }
        // 得到的坐标再乘以倍率
        const have_mult_coor_center = {
          center_x: parseInt(no_mult_coor_center.center_x * zoom_number),
          center_y: parseInt(no_mult_coor_center.center_y * zoom_number)
        }
        // set
        setCoreObjectArray(canvasMarkDataArray[index], "canvasMarkDataObject", "update", 0, "centerPointObject", have_mult_coor_center)
        // 链接点 connect
        let new_point_arr = []
        item.pointDataArray.forEach((child_item, child_index, child_this_arr) => {            
          const point = child_item
          const no_mult_coor_connect = {
            x:point.x - pXYcWithOffset.x,
            y:point.y - pXYcWithOffset.y
          }
          const have_mult_coor_connect = {
            x: parseInt(no_mult_coor_connect.x * zoom_number),
            y: parseInt(no_mult_coor_connect.y * zoom_number)
          }
          new_point_arr.push(have_mult_coor_connect)
        })
        // set
        setCoreObjectArray(canvasMarkDataArray[index], "canvasMarkDataObject", "update", 0, "pointDataArray", new_point_arr)
      })
    },
    // type "0" "1"
    // 更换滤镜选项 执行函数换色
    watchDriveFilterChangeFunc(cv_data_obj, type){
      function aaa(element, index, array) {
        return element >= 0;
      }
      let new_color_obj = {...cv_data_obj}
      let arr = new_color_obj.data
      let arr_2 = arr.filter(aaa)
      let color_array = new this.$Color_class(arr_2)
      const new_color_array = ((color_array) => {
        if (type === "0") {
          return color_array.get_normal
        } else if (type === "1") {
          return color_array.get_reverse
        } else if (type === "2") {
          return color_array.get_hot
        } else if (type === "3") {
          return color_array.get_rainbow
        }
      })(color_array)
      const width = this.canvasOriginSizeObject.width, 
        height = this.canvasOriginSizeObject.height;
      const c = this.canvasObject.createImageData(width, height)
      this.$Tools.updateArrayValue(c.data, new_color_array)
      this.canvasObject.putImageData(c, 0, 0)
    }
  },
  mounted() {
    _this = this
    this.show();

    // test
    // let arr = [0,1,2]
    // this.$delete(arr,0)
    // console.log(arr)
    // console.log($(this.$refs.canvas))
  },  
  watch:{
    // 用户操作选项
    selectValueString:function(new_value, old_value){
      if (new_value === "1" && this.isCanCopyOriginCanvasDataBoolean) {
        this.isCanCopyOriginCanvasDataBoolean = false
      }
    },
    // 色彩滤镜选项
    selectColorFilterString:function(new_value, old_value){
      this.watchDriveFilterChangeFunc(this.canvasOriginZoomDataObject, new_value)
    }
  },
  components: {
    count
  }
};
// event init
painting()
function painting(){
  // onclick
  $(document)
      .off("click", ".image-canvas")
      .on("click", ".image-canvas", function(event) {
        // 如果刚刚处于与多边形失去连接状态
        if (_this.polygonLatingAndFormingIsDisconnectedBoolean) {
          _this.polygonLatingAndFormingIsDisconnectedBoolean = false
          return
        }
        // 如果处于可移动或者拉伸某多边形状态就不启动绘制函数
        if (_this.isMouseCanTranslatePolygonBoolean || _this.isMouseTranslatingPolygonBoolean || _this.isMouseCanTransformPolygonBoolean || _this.isMouseTransformingPolygonBoolean) {
          return
        }
        // 如果不在绘制选项中就不绘制
        if (_this.selectValueString !== "1") {
          return
        }
        // 多边形
        if (_this.selectValueString === "1") {
          pointPaintingFunc(event)            
        }
      })
  // onmousemove 鼠标移动事件
  $(document)
      .off("mousemove", ".image-canvas")
      .on("mousemove", ".image-canvas", function(event) {
        // 移动 拉伸
        if (_this.isPaintedBoolean) {
          const x = event.offsetX, 
            y = event.offsetY;
          const judge_number = _this.centerPointHoverJudgeNumber
          let reDrawStatusBoolean = true
          let haveATranslatePolygon = false
          let haveATransformingPolygon = false
          _this.canvasMarkDataArray.forEach((item, index, this_arr) => {
            // 只计算已完成的多边形
            if (!item.completed) {
              return
            }
            // 计算中心点
            const x_ed = item.centerPointObject ? item.centerPointObject.center_x : -999999
            const y_ed = item.centerPointObject ? item.centerPointObject.center_y : -999999
            // 中心点与鼠标的距离
            const distance = _this.$Painting_tools.calcPointDistanceFunc(x, y, x_ed, y_ed)
            if (distance < judge_number) {
              setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "translateable", true)
              setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "centerPointActive", true)
              _this.canTranslatePolygonIndex = index
              _this.isMouseCanTranslatePolygonBoolean = true
              haveATranslatePolygon = true
              if (_this.isMouseTranslatingPolygonBoolean) {
                // 鼠标移动 mouseleave 记录坐标B
                _this.polygonMovingMouseContinueCoordinateObject = {
                  x:event.offsetX,
                  y:event.offsetY
                }
                // 计算两个坐标对象的差值 保留正负
                const obj_a = _this.polygonMovingMouseStartedCoordinateObject
                const obj_b = _this.polygonMovingMouseContinueCoordinateObject
                const coor_x_distance = obj_b.x - obj_a.x
                const coor_y_distance = obj_b.y - obj_a.y
                // 中心点重新赋值                    
                const new_center_x = item.centerPointObject.center_x + coor_x_distance
                const new_center_y = item.centerPointObject.center_y + coor_y_distance
                setCoreObjectArray(item, "canvasMarkDataObject", "update", 0, "centerPointObject", {
                  center_x:new_center_x,
                  center_y:new_center_y
                })
                // displayModel -> dataModel
                // 显示模型数据 - 数据模型数据
                const modelDataObject = getCurrentDataModelSinglePointValue({center_x:new_center_x, center_y:new_center_y}, "center")
                // item 对应 _this.anotherCanvasMarkDataArray[index]
                setCoreObjectArray(_this.anotherCanvasMarkDataArray[index], "canvasMarkDataObject", "update", 0, "centerPointObject", modelDataObject)
                item.pointDataArray.forEach((child_item, child_index, this_arr)=>{
                  const new_child_item_x = child_item.x + coor_x_distance
                  const new_child_item_y = child_item.y + coor_y_distance
                  setCoreObjectArray(this_arr, "pointDataArray", "update", child_index, null, {
                    x:new_child_item_x,
                    y:new_child_item_y
                  })
                  // displayModel -> dataModel
                  // 显示模型数据 - 数据模型数据
                  const modelDataObject_singlePoint = getCurrentDataModelSinglePointValue({x:new_child_item_x, y:new_child_item_y}, "connect")
                  setCoreObjectArray(_this.anotherCanvasMarkDataArray[index].pointDataArray, "pointDataArray", "update", child_index, null, modelDataObject_singlePoint)
                })
                // 修正polygonMovingMouseStartedCoordinateObject，保证下一次移动正常
                obj_a.x = obj_b.x
                obj_a.y = obj_b.y
              }
            } else {
              // 如果是数组最后一个，且没有可移动多边形
              if (!haveATranslatePolygon && (index === this_arr.length - 1)) {
                _this.canTranslatePolygonIndex = -1
                _this.isMouseCanTranslatePolygonBoolean = false
              }
              setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "translateable", false)
              setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "centerPointActive", false)
            }
            // 多边形的各个点与鼠标的距离
            if (!(distance < judge_number)) {
              const pointDataArray = item.pointDataArray
              const child_judge_number = _this.polygonPointHoverJudgeNumber
              let child_point_is_marked = false
              _this.isMouseCanTransformPolygonBoolean = true
              // foreach
              pointDataArray.forEach((child_item, child_index, this_arr) => {
                // x y 是鼠标此时的点 连接点需要别的标识符
                const point_x = child_item.x,
                  point_y = child_item.y;
                const child_distance = _this.$Painting_tools.calcPointDistanceFunc(x, y, point_x, point_y)
                if (child_distance < child_judge_number) {
                  setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "pointActiveIndex", child_index)
                  child_point_is_marked = true
                  haveATransformingPolygon = true
                  setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "transformable", true)
                } else if (!haveATransformingPolygon && (child_index === pointDataArray.length - 1)) {
                  _this.isMouseCanTransformPolygonBoolean = false
                  setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "transformable", false)
                  setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "pointActiveIndex", null)
                }
              })
              // 鼠标现坐标记录 多边形现坐标计算
              if (_this.isMouseTransformingPolygonBoolean) {
                // if item.pointActiveIndex === null
                if (item.pointActiveIndex === null) {
                  // 重设移动和拉伸布尔值
                  resetLatingAndFormingBooleanFunc()
                  // 激活重绘
                  _this.clearPaintingFunc()
                  _this.reDrawFunc()
                  return
                }
                // 鼠标移动 mouseleave 记录坐标B
                _this.polygonTransformingMouseContinueCoordinateObject = {
                  x:event.offsetX,
                  y:event.offsetY
                }
                // 计算两个坐标对象的差值 保留正负
                const obj_a = _this.polygonTransformingMouseStartedCoordinateObject
                const obj_b = _this.polygonTransformingMouseContinueCoordinateObject
                const coor_x_distance = obj_b.x - obj_a.x
                const coor_y_distance = obj_b.y - obj_a.y
                // 把新点坐标注入到对象中
                const transform_coor_obj = item.pointDataArray[item.pointActiveIndex]
                const new_transform_coor_obj_x = transform_coor_obj.x + coor_x_distance
                const new_transform_coor_obj_y = transform_coor_obj.y + coor_y_distance
                setCoreObjectArray(item.pointDataArray, "pointDataArray", "update", item.pointActiveIndex, null, {
                  x:new_transform_coor_obj_x,
                  y:new_transform_coor_obj_y
                })
                // displayModel -> dataModel
                // 显示模型数据 - 数据模型数据
                const modelDataObject = getCurrentDataModelSinglePointValue({x:new_transform_coor_obj_x, y:new_transform_coor_obj_y}, "connect")
                // item 对应 _this.anotherCanvasMarkDataArray[index]
                setCoreObjectArray(_this.anotherCanvasMarkDataArray[index].pointDataArray, "pointDataArray", "update", item.pointActiveIndex, null, modelDataObject)
                // 修正
                obj_a.x = obj_b.x
                obj_a.y = obj_b.y
                // 中心点坐标重设
                const new_center = _this.$Painting_tools.calcCenterInPolygon(item.pointDataArray, _this.$Tools.compareVal)
                setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "centerPointObject", new_center)
                // displayModel -> dataModel
                // 显示模型数据 - 数据模型数据
                const modelDataObject_center = getCurrentDataModelSinglePointValue(new_center, "center")
                // item 对应 _this.anotherCanvasMarkDataArray[index]
                setCoreObjectArray(_this.anotherCanvasMarkDataArray[index], "canvasMarkDataObject", "update", null, "centerPointObject", modelDataObject_center)
              }
            }
            // 如果找中心点，就不点亮连接点
            if (distance < judge_number) {
              item.pointActiveIndex = null
              setCoreObjectArray(item, "canvasMarkDataObject", "update", null, "pointActiveIndex", null)
            }
            // 重设移动和拉伸布尔值
            resetLatingAndFormingBooleanFunc()
            // 激活重绘
            _this.clearPaintingFunc()
            _this.reDrawFunc()
          })
        }
        // _this.selectValueString 模式选项 1 多边形 2 矩形 3 椭圆
        // 矩形绘制
        if (_this.isRectPaintingBoolean && _this.selectValueString === "2") {            
          rectanglePaintingFunc(event, 2)            
        }
        // 椭圆绘制
        if (_this.isEllipsePaintingBoolean && _this.selectValueString === "3") {            
          ellipsePaintingFunc(event, 2)
        }
      })
  // onmousedown
  $(document)
      .off("mousedown", ".image-canvas")
      .on("mousedown", ".image-canvas", function(event) {
        if (_this.isMouseCanTranslatePolygonBoolean && event.which === 1) {
          // 鼠标点击 mousedown 记录坐标A
          _this.polygonMovingMouseStartedCoordinateObject = {
            x:event.offsetX,
            y:event.offsetY
          }
          _this.isMouseTranslatingPolygonBoolean = true
        }
        // console.log(_this.isMouseCanTransformPolygonBoolean)
        if (_this.isMouseCanTransformPolygonBoolean && event.which === 1) {
          // 鼠标点击 mousedown 记录坐标A
          _this.polygonTransformingMouseStartedCoordinateObject = {
            x:event.offsetX,
            y:event.offsetY
          }
          _this.isMouseTransformingPolygonBoolean = true
        }
        // 删除点
        if (event.which === 3) {
          // brightRadioTypeNumber === 0 可以删除
          if (!_this.brightRadioTypeNumber) {
            pointDeleteFunc(event)
          }
        }
        // 矩形开始绘制
        if (_this.selectValueString === "2" && !_this.isMouseTranslatingPolygonBoolean && !_this.isMouseTransformingPolygonBoolean && event.which === 1) {
          rectanglePaintingFunc(event, 1)
        }
        // 椭圆开始绘制
        if (_this.selectValueString === "3" && !_this.isMouseTranslatingPolygonBoolean && !_this.isMouseTransformingPolygonBoolean && event.which === 1) {
          // 开始绘制
          ellipsePaintingFunc(event, 1)
        }
      })
  // onmouseup
  $(document)
      .off("mouseup", ".image-canvas")
      .on("mouseup", ".image-canvas", function(event) {
        if (_this.isMouseCanTranslatePolygonBoolean) {            
          _this.isMouseTranslatingPolygonBoolean = false
          _this.polygonMovingMouseStartedCoordinateObject = null
          _this.polygonMovingMouseContinueCoordinateObject = null
        }
        if (_this.isMouseCanTransformPolygonBoolean) {            
          _this.isMouseTransformingPolygonBoolean = false
          _this.polygonTransformingMouseStartedCoordinateObject = null
          _this.polygonTransformingMouseContinueCoordinateObject = null
        }
        // 矩形结束绘制
        if (_this.isRectPaintingBoolean && _this.selectValueString === "2") {
          rectanglePaintingFunc(event, 3)
        }
        // 椭圆结束绘制
        if (_this.isEllipsePaintingBoolean && _this.selectValueString === "3") {
          ellipsePaintingFunc(event, 3)
        }
      })
  // onkeydown 键盘事件
  $(document).keydown(function(event){
    // console.log(event)
    if (!_this.keyEventBaseBoolean) {
      return
    }
    _this.keyEventBaseBoolean = false
    _this.keyEventObject = event
    setTimeout(() => {
      keydownEventSwitchFunc(event)
      _this.keyEventBaseBoolean = true
    },30)
  })
}
// 键盘事件分发
function keydownEventSwitchFunc(event){
  // w 87 up
  if (event.keyCode === 87) {
    _this.directionTopFunc(event)
  }
  // s 83 down
  if (event.keyCode === 83) {
    _this.directionBottomFunc(event)
  }
  // d 68 right
  if (event.keyCode === 68) {
    _this.directionRightFunc(event)
  }
  // a 65 left
  if (event.keyCode === 65) {
    _this.directionLeftFunc(event)
  }
  // + 187 add zoom
  if (event.keyCode === 187) {
    _this.zoomPlusFunc()
  }
  // - 189 sub zoom 
  if (event.keyCode === 189) {
    _this.zoomSubFunc()
  }
}
// 删除点的函数
function pointDeleteFunc (event) {
  const x = event.offsetX,
    y = event.offsetY;
  let catch_item, catch_index
  try {
    let delete_index = -1
    _this.canvasMarkDataArray.forEach((item, index) => {
      // 计算中心点
      // result boolean
      const result = _this.$Tools.centerPointDeleteFunc(item, index, x, y, _this.centerPointHoverJudgeNumber, _this.$Painting_tools.calcPointDistanceFunc)
      // 如果返回true就跳出foreach
      if (result) {
        catch_item = item
        catch_index = index
        throw new Error("end_foreach");
      }
      // 计算连接点
      let delete_child_index = -1
      item.pointDataArray.forEach((child_item, child_index) => {
        let point_x = child_item.x
        let point_y = child_item.y
        // 连接点与鼠标的距离
        let distance = _this.$Painting_tools.calcPointDistanceFunc(x, y, point_x, point_y)
        let judge_number = _this.arcRNumber + _this.arcWidthNumber - 1  // 5
        // 只有没画完的点才能删除
        if (distance < judge_number && !item.completed) {
          delete_child_index = child_index
          return
        }
      })
      // 删除某个点
      setCoreObjectArray(item.pointDataArray, "pointDataArray", "delete", delete_child_index, null, null)
      // 对应的dataModelObject也要删除
      setCoreObjectArray(_this.anotherCanvasMarkDataArray[index].pointDataArray, "pointDataArray", "delete", delete_child_index, null, null)
      // 如果发现某个多边形已经没有点了 则删除这个多边形对象
      if (item.pointDataArray.length === 0) {
        delete_index = index
        // 关闭绘制状态
        _this.isPointingBoolean = false
        return
      }
    })
    // 如果发现某个多边形已经没有点了 则删除这个多边形对象
    if (delete_index > -1) {
      setCoreObjectArray(_this.canvasMarkDataArray, "canvasMarkDataArray", "delete", delete_index, null, null)
      // 对应的dataModelObject也要删除
      setCoreObjectArray(_this.anotherCanvasMarkDataArray, "canvasMarkDataArray", "delete", delete_index, null, null)
    }
  } catch (error) {
    if (error.message === "end_foreach") {
      // catch_index 可能是0 所以不能用
      if (!catch_item) {
        return
      }
      if (confirm("确定删除这个多边形？")) {
        setCoreObjectArray(_this.canvasMarkDataArray, "canvasMarkDataArray", "delete", catch_index, null, null)
        // 对应的dataModelObject也要删除
        setCoreObjectArray(_this.anotherCanvasMarkDataArray, "canvasMarkDataArray", "delete", catch_index, null, null)
        if (_this.canvasMarkDataArray.length === 0) {
          checkCanvasMarkDataArrayFunc()
        }
      }
    }
  }
  // 激活重绘
  _this.clearPaintingFunc()
  _this.reDrawFunc()
}
// 把lating和forming布尔值重设为正确的值
function resetLatingAndFormingBooleanFunc(){
  // 如果此时所有多边形都不处于移动和拉伸状态
  const allPolygonIsStaticBoolean = _this.$Tools.allPolygonLateAndFormCheckFunc(_this.canvasMarkDataArray)
  // 就 _this.isMouseTranslatingPolygonBoolean _this.isMouseTransformingPolygonBoolean 为 false
  if (allPolygonIsStaticBoolean) {
    let result = false
    if (_this.isMouseTranslatingPolygonBoolean) {
      _this.isMouseTranslatingPolygonBoolean = false
      result = true
    }
    if (_this.isMouseTransformingPolygonBoolean) {
      _this.isMouseTransformingPolygonBoolean = false
      result = true
    }
    if (result) {
      _this.polygonLatingAndFormingIsDisconnectedBoolean = true
    }                
  }
}
// 如果没有任何图形就重置状态
function checkCanvasMarkDataArrayFunc(){
  if (_this.canvasMarkDataArray.length === 0) {
    _this.isMouseCanTranslatePolygonBoolean = false
    _this.isMouseTranslatingPolygonBoolean = false
    _this.isMouseCanTransformPolygonBoolean = false
    _this.isMouseTransformingPolygonBoolean = false
  }
}
// 多边形 记录点的函数
function pointPaintingFunc (event) {
  const r = _this.arcRNumber
  const w = _this.arcWidthNumber
  const arc_color = "red"
  const arc_stroke_color = "black"
  if (!_this.isPointingBoolean) {
    const canvasMarkDataObject = new _this.$CanvasMarkDataObject_class(_this.$Tools.randomString(), 'polygon', event.offsetX, event.offsetY)
    setCoreObjectArray(_this.canvasMarkDataArray, "canvasMarkDataArray", "add", _this.canvasMarkDataArray.length, null, canvasMarkDataObject)
    // displayModel -> dataModel
    // 显示模型的数据 - 数据模型的数据
    // 用...运算符实现拷贝（不深）
    // const modelDataObject = getCurrentDataModelValue({...canvasMarkDataObject, isDataModel:true}, canvasMarkDataObject.type)
    const modelDataObject = getCurrentDataModelValue(_this.$Tools.objectCopy({...canvasMarkDataObject, isDataModel:true}), canvasMarkDataObject.type)
    setCoreObjectArray(_this.anotherCanvasMarkDataArray, "canvasMarkDataArray", "add", _this.anotherCanvasMarkDataArray.length, null, modelDataObject)
    // painting
    _this.isPointingBoolean = true    
    _this.$Painting_tools.drawArcFunc(_this.canvasPaintingObject, event.offsetX, event.offsetY, r, arc_color, w, arc_stroke_color)
  } else {
    const canvasMarkDataObject = _this.canvasMarkDataArray[_this.canvasMarkDataArray.length - 1]
    const pointDataArray = canvasMarkDataObject.pointDataArray
    let prevX,prevY
    if (pointDataArray[pointDataArray.length - 1]) {
      prevX = pointDataArray[pointDataArray.length - 1].x
      prevY = pointDataArray[pointDataArray.length - 1].y
    } else {
      prevX = prevY = null
    }
    // 检测这一点是否和上一点过近，过近就启动完成判定  
    if (_this.$Painting_tools.checkPointAndPointIsNearFunc(event.offsetX, event.offsetY, prevX, prevY, _this.sqrtStandandNumber)) {
      settlePointFunc(canvasMarkDataObject, pointDataArray, pointDataArray.length)
    } else {
      setCoreObjectArray(pointDataArray, "pointDataArray", "add", pointDataArray.length, null,
        {x: event.offsetX, y: event.offsetY})
      setCoreObjectArray(_this.canvasMarkDataArray, "canvasMarkDataArray", "update", _this.canvasMarkDataArray.length - 1, null, canvasMarkDataObject)
      // displayModel -> dataModel
      // 显示模型的数据 - 数据模型的数据
      const canvasMarkDataObject_another = _this.anotherCanvasMarkDataArray[_this.anotherCanvasMarkDataArray.length - 1]
      setCoreObjectArray(
        canvasMarkDataObject_another.pointDataArray,
        "pointDataArray",
        "add", 
        pointDataArray.length, 
        null,
        getCurrentDataModelSinglePointValue({x: event.offsetX, y: event.offsetY}, "connect")
      )
      setCoreObjectArray(_this.anotherCanvasMarkDataArray, "canvasMarkDataArray", "update", _this.anotherCanvasMarkDataArray.length - 1, null, canvasMarkDataObject_another)
      // painting
      _this.$Painting_tools.drawArcFunc(_this.canvasPaintingObject, event.offsetX, event.offsetY, r, arc_color, w,arc_stroke_color)
    }
  }
}
// 矩形绘制函数
// type 1 mousedown 2 mouseleave 3 mouseup
function rectanglePaintingFunc (event, type) {
  const r = _this.arcRNumber
  if (type === 1) {
    _this.isPaintedBoolean = false
    _this.isRectPaintingBoolean = true
    // 把数据对象放进实体类
    const canvasMarkDataObject = new _this.$CanvasMarkDataObject_class(_this.$Tools.randomString(), 'rectangle', event.offsetX, event.offsetY)
    setCoreObjectArray(_this.canvasMarkDataArray, "canvasMarkDataArray", "add", _this.canvasMarkDataArray.length, null, canvasMarkDataObject)
    // displayModel -> dataModel
    // 显示模型的数据 - 数据模型的数据
    // 用...运算符实现拷贝（不深）
    // const modelDataObject = getCurrentDataModelValue({...canvasMarkDataObject, isDataModel:true}, canvasMarkDataObject.type)
    const modelDataObject = getCurrentDataModelValue(_this.$Tools.objectCopy({...canvasMarkDataObject, isDataModel:true}), canvasMarkDataObject.type)
    setCoreObjectArray(_this.anotherCanvasMarkDataArray, "canvasMarkDataArray", "add", _this.anotherCanvasMarkDataArray.length, null, modelDataObject)
  } else if (type === 2) {
    // 拖动时重设右下角点的值
    const canvasMarkDataObject = _this.canvasMarkDataArray[_this.canvasMarkDataArray.length - 1]
    const pointDataArray = canvasMarkDataObject.pointDataArray
    // another
    let pointDataArray_2 = _this.anotherCanvasMarkDataArray[_this.anotherCanvasMarkDataArray.length - 1].pointDataArray
    // set
    setCoreObjectArray(pointDataArray, "pointDataArray", "update", 1, null, {
      x:event.offsetX,
      y:event.offsetY
    })
    // displayModel -> dataModel
    // 显示模型的数据 - 数据模型的数据
    const modelDataObject = getCurrentDataModelSinglePointValue({
      x:event.offsetX,
      y:event.offsetY
    }, "connect")
    setCoreObjectArray(pointDataArray_2, "pointDataArray", "update", 1, null, modelDataObject)
    // 重绘
    _this.clearPaintingFunc()
    _this.reDrawFunc()
    drawRectBeforeFunc(canvasMarkDataObject)
  } else if (type === 3) {
    const canvasMarkDataObject = _this.canvasMarkDataArray[_this.canvasMarkDataArray.length - 1]
    // another
    let canvasMarkDataObject_another = _this.anotherCanvasMarkDataArray[_this.anotherCanvasMarkDataArray.length - 1] 
    // 记录中心点
    const new_center = _this.$Painting_tools.calcCenterInPolygon(canvasMarkDataObject.pointDataArray, _this.$Tools.compareVal)
    setCoreObjectArray(canvasMarkDataObject, "canvasMarkDataObject", "update", null, "completed", true)
    setCoreObjectArray(canvasMarkDataObject, "canvasMarkDataObject", "update", null, "centerPointObject", new_center)
    // displayModel -> dataModel
    // 显示模型的数据 - 数据模型的数据
    const modelDataObject = getCurrentDataModelSinglePointValue({
      center_x:new_center.center_x,
      center_y:new_center.center_y,
    }, "center")
    setCoreObjectArray(canvasMarkDataObject_another, "canvasMarkDataObject", "update", null, "centerPointObject", modelDataObject)
    // boolean
    _this.isPaintedBoolean = true
    _this.isRectPaintingBoolean = false
    // 重绘
    _this.clearPaintingFunc()
    _this.reDrawFunc()
  }
}
// 椭圆绘制函数
// type 1 mousedown 2 mouseleave 3 mouseup
function ellipsePaintingFunc (event, type) {
  const r = _this.arcRNumber
  if (type === 1) {
    _this.isPaintedBoolean = false
    _this.isEllipsePaintingBoolean = true
    // 把数据对象放进实体类
    const canvasMarkDataObject = new _this.$CanvasMarkDataObject_class(_this.$Tools.randomString(), 'ellipse', event.offsetX, event.offsetY)
    setCoreObjectArray(_this.canvasMarkDataArray, "canvasMarkDataArray", "add", _this.canvasMarkDataArray.length, null, canvasMarkDataObject)
    // displayModel -> dataModel
    // 显示模型的数据 - 数据模型的数据
    // 用...运算符实现拷贝（不深）
    // const modelDataObject = getCurrentDataModelValue({...canvasMarkDataObject, isDataModel:true}, canvasMarkDataObject.type)
    const modelDataObject = getCurrentDataModelValue(_this.$Tools.objectCopy({...canvasMarkDataObject, isDataModel:true}), canvasMarkDataObject.type)
    setCoreObjectArray(_this.anotherCanvasMarkDataArray, "canvasMarkDataArray", "add", _this.anotherCanvasMarkDataArray.length, null, modelDataObject)
  } else if (type === 2) {
    // 拖动时重设右下角点的值
    const canvasMarkDataObject = _this.canvasMarkDataArray[_this.canvasMarkDataArray.length - 1]
    const pointDataArray = canvasMarkDataObject.pointDataArray
    // another
    let pointDataArray_2 = _this.anotherCanvasMarkDataArray[_this.anotherCanvasMarkDataArray.length - 1].pointDataArray
    // 椭圆只有一个控制点
    setCoreObjectArray(pointDataArray, "pointDataArray", "update", 0, null, {
      x:event.offsetX,
      y:event.offsetY
    })
    // displayModel -> dataModel
    // 显示模型的数据 - 数据模型的数据
    const modelDataObject = getCurrentDataModelSinglePointValue({
      x:event.offsetX,
      y:event.offsetY
    }, "connect")
    setCoreObjectArray(pointDataArray_2, "pointDataArray", "update", 0, null, modelDataObject)
    // 算中心点 和控制点0 的差值
    const hori = canvasMarkDataObject.centerPointObject.center_x - event.offsetX
    const vert = canvasMarkDataObject.centerPointObject.center_y - event.offsetY
    const second_point_x = canvasMarkDataObject.centerPointObject.center_x + hori
    const second_point_y = canvasMarkDataObject.centerPointObject.center_y + vert
    setCoreObjectArray(pointDataArray, "pointDataArray", "update", 1, null, {
      x:second_point_x,
      y:second_point_y
    })
    // displayModel -> dataModel
    // 显示模型的数据 - 数据模型的数据
    const modelDataObject_singlePoint = getCurrentDataModelSinglePointValue({
      x:second_point_x,
      y:second_point_y
    }, "connect")
    setCoreObjectArray(pointDataArray_2, "pointDataArray", "update", 1, null, modelDataObject_singlePoint)
    // 重绘
    _this.clearPaintingFunc()
    _this.reDrawFunc()
    drawEllipseBeforeFunc(canvasMarkDataObject)
  } else if (type === 3) {
    const canvasMarkDataObject = _this.canvasMarkDataArray[_this.canvasMarkDataArray.length - 1]
    setCoreObjectArray(canvasMarkDataObject, "canvasMarkDataObject", "update", null, "completed", true) 
    _this.isPaintedBoolean = true
    _this.isEllipsePaintingBoolean = false
    // 重绘
    _this.clearPaintingFunc()
    _this.reDrawFunc()
  }
}
// 结算入口，如果可以结算就绘制
function settlePointFunc (canvasMarkDataObject, pointDataArray, length) {
  const min = _this.minPointNumNumber
  const line_width = 1
  const line_color = "red"
  if (length < min) {
    alert("最少要画3个点")
    return
  } else {
    // normal - display model
    let canvasMarkDataObject = _this.canvasMarkDataArray[_this.canvasMarkDataArray.length - 1]
    let pointDataArray = canvasMarkDataObject.pointDataArray
    // another - data model
    let canvasMarkDataObject_another = _this.anotherCanvasMarkDataArray[_this.anotherCanvasMarkDataArray.length - 1]
    // 绘制
    _this.$Painting_tools.drawPointLineFunc(_this.canvasPaintingObject, pointDataArray, line_color, line_width)
    drawCenterArcFunc(pointDataArray, canvasMarkDataObject)
    // 数据模型相关对象注入中心点
    setAnotherCenterArc(canvasMarkDataObject_another.pointDataArray, canvasMarkDataObject_another)
    _this.isPointingBoolean = false    // 正在绘制
    _this.isPaintedBoolean = true    // 绘制过
    // 记点
    setCoreObjectArray(canvasMarkDataObject, "canvasMarkDataObject", "update", null, "completed", true)
  }
}
// 绘制多边形中心点
function drawCenterArcFunc(pointDataArray, canvasMarkDataObject){
  const r = _this.arcRNumber
  const w = _this.arcWidthNumber
  const arc_color = "red"
  const center_arc_stroke_color = "white"
  const obj = _this.$Painting_tools.calcCenterInPolygon(pointDataArray, _this.$Tools.compareVal)
  canvasMarkDataObject.centerPointObject = obj    // 把中心点坐标放进多边形对象
  _this.$Painting_tools.drawArcFunc(_this.canvasPaintingObject, obj.center_x, obj.center_y, r, arc_color, w, center_arc_stroke_color)
}
// 重绘用 画圆入口
function drawArcBeforeFunc(pointDataArray, pointActiveIndex, type){
  const r = _this.arcRNumber
  pointDataArray.forEach((item, index) => {
    // console.log(item)
    let arc_color
    const color_ex = "yellow"
    const color_nm = "red"
    const arc_width = 2
    const arc_stroke_color = "black"
    if (type === "polygon" || type === "rectangle" || type === "ellipse") {
      pointActiveIndex === index ? arc_color = color_ex : arc_color = color_nm
    } else {
      arc_color = color_nm
    }
    _this.$Painting_tools.drawArcFunc(_this.canvasPaintingObject, item.x, item.y, r, arc_color, arc_width, arc_stroke_color)
  })
}
// 重绘用 画中心点-圆入口
function drawCenterArcBeforeFunc(centerPointObject, centerAcitveBoolean){
  let arc_color
  const r = _this.arcRNumber
  const w = _this.arcWidthNumber
  const center_arc_stroke_color = "white"
  const color_ex = "yellow",
    color_nm = "red";
  centerAcitveBoolean ? arc_color = color_ex : arc_color = color_nm
  _this.$Painting_tools.drawArcFunc(_this.canvasPaintingObject, centerPointObject.center_x, centerPointObject.center_y, r, arc_color, w, center_arc_stroke_color)
}
// 重绘用 画矩形入口
function drawRectBeforeFunc(item){
  const rect_fill_color = "red",
    rect_width = 1,
    rect_stroke_color = "red";
  const x0 = item.pointDataArray[0].x,
    y0 = item.pointDataArray[0].y,
    x1 = item.pointDataArray[1].x,
    y1 = item.pointDataArray[1].y;
  _this.$Painting_tools.drawRectFunc(_this.canvasPaintingObject, x0, y0, x1, y1, rect_fill_color, rect_width, rect_stroke_color)
}
// 画椭圆入口
// item: canvasMarkDataObject
// x为椭圆中心横坐标，y为椭圆中心纵坐标，a为椭圆横半轴长，b为椭圆纵半轴长。
function drawEllipseBeforeFunc(item){
  const rect_fill_color = "red",
    rect_width = 1,
    rect_stroke_color = "red";
  const x = item.centerPointObject.center_x,
    y = item.centerPointObject.center_y,
    a = item.pointDataArray[0].x - item.centerPointObject.center_x, // 控制点坐标 - 中心点坐标
    b = item.pointDataArray[0].y - item.centerPointObject.center_y;
  _this.$Painting_tools.drawEllipseFunc(_this.canvasPaintingObject, x, y, a, b, rect_fill_color, rect_width, rect_stroke_color)
}
// canvasMarkDataArray 的所有增删改归于一个函数管理
// a target b name c status d index e key f value
// name: [canvasMarkDataArray, canvasMarkDataObject, centerPointObject, pointDataArray]
function setCoreObjectArray(target, name, status, index, key, value){
  // console.log("===" + name)
  // 总数组
  if (name === "canvasMarkDataArray") {
    if (status === "add") {
      _this.$set(target, target.length, value)
      return
    }
    if (status === "update") {
      _this.$set(target, index, value)
      return
    }
    if (status === "delete") {
      _this.$delete(target, index)
      return
    }
  }
  // 一个图形的对象
  if (name === "canvasMarkDataObject") {
    if (status === "update") {
      if (typeof value === "boolean" || typeof value === "number" || typeof value === "null" || typeof value === "object") {
        target[key] = value
        return
      }
    }
  }
  // 一个图形对象的点数组
  if (name === "pointDataArray") {
    if (status === "add") {
      _this.$set(target, target.length, value)
      return
    }
    if (status === "update") {
      _this.$set(target, index, value)
      return
    }
    if (status === "delete") {
      _this.$delete(target, index)
      return
    }
  }
}
// displayModel -> dataModel
// 把显示模型的数据转成数据模型
// type ellipse rectangle polygon
function getCurrentDataModelValue(canvasMarkDataObject, type){
  // prototype 原型 scroll 实际 view 差值
  const zoom_number = _this.zoomNumberArray[_this.zoomIndexNumber],
    proto_obj = _this.prototypeXYCoordinateObject,
    standand_number = _this.directionStandandValueNumber,
    view_coordinate = _this.viewXYCoordinateObject;
  if (type === "ellipse") {
    let {center_x, center_y} = canvasMarkDataObject.centerPointObject
    // A = (width - width / zoom比例) / 2
    // B = A + ([display]x / zoom比例)
    // C = B + (方向键值 * 方向值单位)
    const x_calc = (num) => proto_obj.x + num / zoom_number + view_coordinate.x
    const y_calc = (num) => proto_obj.y + num / zoom_number + view_coordinate.y
    const obj = {
      center_x: parseInt(x_calc(center_x)),
      center_y: parseInt(y_calc(center_y))
    }
    // 把修改后的值赋值给(dataModel)canvasMarkDataObject
    // centerPointObject
    canvasMarkDataObject.centerPointObject = obj
    // pointDataArray
    canvasMarkDataObject.pointDataArray = [
      {
        x:obj.center_x,
        y:obj.center_y
      },
      {
        x:obj.center_x,
        y:obj.center_y
      }
    ]      
    return canvasMarkDataObject
  } else if (type === "rectangle" || type === "polygon") {
    const x_calc = (num) => proto_obj.x + num / zoom_number + view_coordinate.x
    const y_calc = (num) => proto_obj.y + num / zoom_number + view_coordinate.y
    let arr = []
    let origin_arr = canvasMarkDataObject.pointDataArray
    let center_x, center_y
    let o

    if (canvasMarkDataObject.centerPointObject) {
      center_x = canvasMarkDataObject.centerPointObject.center_x
      center_y = canvasMarkDataObject.centerPointObject.center_y
      o = {
        center_x: center_x,
        center_y: center_y
      }
    }
    
    origin_arr.forEach((item, index) => {
      let obj = {}
      const x = item.x,
        y = item.y;
      obj.x = parseInt(x_calc(x))
      obj.y = parseInt(y_calc(y))
      arr.push(obj)
    })
    origin_arr = null
    // pointDataArray
    canvasMarkDataObject.pointDataArray = arr    
    // centerPointObject
    if (canvasMarkDataObject.centerPointObject) {
      canvasMarkDataObject.centerPointObject = o
    }    
    return canvasMarkDataObject
  }
}
// displayModel -> dataModel
// 把显示模型的数据转成数据模型
// 当数据是只有xy值的对象时用这个
// type connect 连接点 center 中心点
// prototype 原型 scroll 实际 view 差值
function getCurrentDataModelSinglePointValue(singlePointObject, type){
  if (!(type === "connect" || type === "center")) {
    console.warn("getCurrentDataModelSinglePointValue type ERROR: unknown type " + `=${type}=`)
  }
  const zoom_number = _this.zoomNumberArray[_this.zoomIndexNumber],
    proto_obj = _this.prototypeXYCoordinateObject,
    standand_number = _this.directionStandandValueNumber,
    view_coordinate = _this.viewXYCoordinateObject;
  // zoom为 1时
  if (zoom_number === 1) {
    let x, y
    type === "connect" ? x = singlePointObject.x : x = singlePointObject.center_x
    type === "connect" ? y = singlePointObject.y : y = singlePointObject.center_y
    let obj
    type === "connect" ? obj = {x:parseInt(x), y:parseInt(y)} : obj = {center_x:parseInt(x), center_y:parseInt(y)}
    return obj
  } else {
    let x, y
    type === "connect" ? x = singlePointObject.x : x = singlePointObject.center_x
    type === "connect" ? y = singlePointObject.y : y = singlePointObject.center_y
    const x_calc = (num) => proto_obj.x + num / zoom_number + view_coordinate.x
    const y_calc = (num) => proto_obj.y + num / zoom_number + view_coordinate.y
    let obj
    type === "connect" ? obj = {x:parseInt(x_calc(x)), y:parseInt(y_calc(y))} : obj = {center_x:parseInt(x_calc(x)), center_y:parseInt(y_calc(y))}
    return obj
  }
}
// 计算data model 中心点
function setAnotherCenterArc(pointDataArray, canvasMarkDataObject){
  let obj = _this.$Painting_tools.calcCenterInPolygon(pointDataArray, _this.$Tools.compareVal)
  canvasMarkDataObject.centerPointObject = obj    // 把中心点坐标放进多边形对象
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.image-canvas-wrapper {
  width: 80vh;
  height: 80vh;
  margin: 0 auto;
}

.image-canvas {
  width: 100%;
  height: 100%;
}

.fixed{
  position: fixed;
  right: 5px;
  top:5px;
  border: 1px solid #666;
  width:200px;
  height: 80vh;
  overflow: auto;
}
.direction-control{
  width: 90px;
  height: 90px;
  display: flex;
  flex-wrap: wrap;
  margin: 0 auto;
}
.direction-control div{
  width: 30px;
  height: 30px;
}
.direction-control div:nth-child(even){
  background: lightblue;
}
</style>
