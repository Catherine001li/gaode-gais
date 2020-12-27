<template>
  <div class="app-container">
    <!-- <el-button type="primary" @click="seeBatteryHistory()">点击查看历史轨迹</el-button> -->
    <el-button type="primary" @click="addMarker()">新增标记点</el-button>
    <el-button type="primary" @click="clearMarker()">删除标记点</el-button>
    <el-button type="primary" @click="updateMarker()">更改标记点</el-button>
    <el-button type="primary" @click="updateContent()">更新标记点内容</el-button>
    <el-button type="primary" @click="playcustomLine()">将点连线线</el-button>
    <el-button type="primary" @click="drawPolyline()">绘制折线</el-button>
    <el-button type="primary" @click="editPolyline()">开始编辑折线</el-button>
    <el-button type="primary" @click="finishPolyline()">结束编辑折线</el-button>
    <!-- 查看电池历史轨迹 -->
      <div id="map-container" style="width:100%; height:500px"></div>
  </div>
</template>

<!--
使用说明
index.html引入 
‘<script type="text/javascript" src="https://webapi.amap.com/maps?v=1.4.15&key=872d315a8f55a1da909c08dc50e36a4f"></script>’

vue.config.js引入
module.exports = {
  configureWebpack: {
    externals: {
      'AMap': 'AMap' // 高德地图配置
    }
  }
}

官方实例https://lbs.amap.com/api/javascript-api/example/marker/replaying-historical-running-data/?sug_index=6
-->
<script>
import markData from '@/assets/data/mark.js'

//引入高德地图
import AMap from "AMap";
export default {
  data() {
    return {
      listLoading: true,
      dialogVisible: false, //电池轨迹dialog
      //地图
      map: null,
      marker: null,
      polyline: null,
      passedPolyline: null,
      selectedDate: [],
      marks: [],
      markerArr: [], //电池经纬度,也用于画线
      startIcon: new AMap.Icon({
        //起点图标
        // 图标尺寸
        size: new AMap.Size(25, 34),
        // 图标的取图地址
        image:
          "//a.amap.com/jsapi_demos/static/demo-center/icons/dir-marker.png",
        // 图标所用图片大小
        imageSize: new AMap.Size(135, 40),
        // 图标取图偏移量
        imageOffset: new AMap.Pixel(-9, -3)
      }),
      endIcon: new AMap.Icon({
        //终点图标
        size: new AMap.Size(25, 34),
        image:
          "//a.amap.com/jsapi_demos/static/demo-center/icons/dir-marker.png",
        imageSize: new AMap.Size(135, 40),
        imageOffset: new AMap.Pixel(-95, -3)
      }),
      infoWindow: null,
      curPolyline: null,
      polyEditor: null
    };
  },
  mounted() {
    this.seeBatteryHistory()
    
  },
  methods: {
    //点击查看历史轨迹
    seeBatteryHistory() {
      let that = this;
      that.dialogVisible = true;
      that.$nextTick(() => {
        //初始化地图
        that.initMap();
      });
      //初始化地图
    },
    //初始化地图
    initMap() {
      let that = this;
      that.map = new AMap.Map("map-container", {
        resizeEnable: true,
        zoom: 7,
        // mapStyle: 'amap://styles/e167856f806be09b1640fc7ca09ef8e4',
        center: [120.6641561, 31.3063851] //地图中心点
      });
      that.map.setMapStyle('amap://styles/darkblue')
      AMap.plugin(["AMap.ToolBar"], function() {
        //加载工具条
        var tool = new AMap.ToolBar();
        that.map.addControl(tool);
      });
      this.initLoadHistory()
      that.map.on('click', function(ev) {
        // 触发事件的对象
        let target = ev.target;
        // 触发事件的地理坐标，AMap.LngLat 类型
        let lnglat = ev.lnglat;
        // 触发事件的像素坐标，AMap.Pixel 类型
        let pixel = ev.pixel;
        // 触发事件类型
        var type = ev.type;
        console.log('mapClick', target, lnglat, pixel, type)
      });
      that.infoWindow = new AMap.InfoWindow({offset: new AMap.Pixel(0, -30)});
      
     
     // setTimeout(function() {
      //   that.$message({
      //     title: "提醒",
      //     message: "请选择时间范围！",
      //     type: "warning",
      //     duration: 2000
      //   });
      // }, 500);
    },
    //选择时间范围后
    changeDate() {
      //清除地图的所有覆盖物
      this.map.clearMap();
      //电池经纬度,也用于画线
      this.markerArr = [];
      //获取数据
      this.GetBatteryHistory();
    },
    // 加载地图
    initLoadHistory(){
      //清除地图的所有覆盖物
      this.map.clearMap();
      //电池经纬度,也用于画线
      this.markerArr = [];
      //获取数据
      this.GetBatteryHistory();
    },
    //获取轨迹数据
    GetBatteryHistory() {
      let that = this;
      this.listLoading = true;
      //接口获得数据后转换，高德接受的数据格式为[[111.11,122.22],[111.11,123.33]]
      setTimeout(() => {
        that.listLoading = false;
        //坐标点数组转换
        let temp = markData.trackData
        if (temp.length > 0) {
          temp.map(ele => {
            let tempArr = [];
            tempArr.push(ele.latitude);
            tempArr.push(ele.longitude);
            //格式转换后push进电池经纬度
            that.markerArr.push(tempArr);
          });
          that.$nextTick(() => {
            //画线
            that.playLine();
          });
        } else {
          that.$notify({
            title: "提醒",
            message: "此电池此时间范围内暂无轨迹！",
            type: "warning",
            duration: 4000
          });
        }
      }, 1000);
    },
    //画线
    playLine() {
      let that = this;
      //初始化起点终点
      that.initStartEnd();
      that.marker = new AMap.Marker({
        map: that.map,
        //小车出初始位置
        position: that.markerArr[0],
        icon: "https://webapi.amap.com/images/car.png",
        offset: new AMap.Pixel(-26, -13),
        autoRotation: true,
        angle: -90
      });
      // 绘制轨迹
      that.polyline = new AMap.Polyline({
        map: that.map,
        path: that.markerArr,
        showDir: true,
        strokeColor: "#28F", //线颜色
        // strokeOpacity: 1,     //线透明度
        strokeWeight: 6 //线宽
        // strokeStyle: "solid"  //线样式
      });
      //小车移动的轨迹线
      that.passedPolyline = new AMap.Polyline({
        map: that.map,
        // path: that.markerArr,
        strokeColor: "#AF5", //线颜色
        // strokeOpacity: 1,     //线透明度
        strokeWeight: 6 //线宽
        // strokeStyle: "solid"  //线样式
      });
      //marker移动时把经纬度传给小车移动的轨迹线
      that.marker.on("moving", function(e) {
        //设置组成该折线的节点数组
        that.passedPolyline.setPath(e.passedPath);
      });
      // marker开始移动
      that.marker.moveAlong(that.markerArr, 200);
      //自适应放大
      setTimeout(() => {
        that.map.setFitView();
      }, 500);
    },
    //初始化起点终点
    initStartEnd() {
      let that = this;
      //将icon添加进marker
      let startMarker = new AMap.Marker({
        position: new AMap.LngLat(that.markerArr[0][0], that.markerArr[0][1]),
        icon: that.startIcon,
        offset: new AMap.Pixel(-13, -30)
      });
      //将icon添加进marker
      let endMarker = new AMap.Marker({
        position: new AMap.LngLat(
          that.markerArr.pop()[0],
          that.markerArr.pop()[1]
        ),
        icon: that.endIcon,
        offset: new AMap.Pixel(-13, -30)
      });
      // 将 markers 添加到地图
      that.map.add([startMarker, endMarker]);
    },
    //关闭弹窗时
    closeBatteryHistroy() {
      this.selectedDate = [];
      this.markerArr = []; //电池经纬度,也用于画线
    },
    // 画线
    playcustomLine(){
      let that = this
      // 小车移动的轨迹线
      new AMap.Polyline({
        map: that.map,
        path: that.marks,
        strokeColor: "#28F", //线颜色
        // strokeOpacity: 1,     //线透明度
        strokeWeight: 6, //线宽
        strokeStyle: "solid"  //线样式
      });
    },
    // 添加标记点
    addMarker() {
      let that = this
      this.marks = [];
      let tempmarkers = markData.markers
        if (tempmarkers.length > 0) {
          tempmarkers.map(ele => {
            let tempArr = [];
            tempArr.push(ele.latitude);
            tempArr.push(ele.longitude);
            //格式转换后push进电池经纬度
            that.marks.push(tempArr);
          });
        }
      let markers = [{
        icon: '//a.amap.com/jsapi_demos/static/demo-center/icons/poi-marker-1.png',
        position: that.marks[0],
        title: '标记点1',
    }, {
        icon: '//a.amap.com/jsapi_demos/static/demo-center/icons/poi-marker-2.png',
        position: that.marks[1],
        title: '标记点2',
    }, {
        // icon: '//a.amap.com/jsapi_demos/static/demo-center/icons/poi-marker-3.png',
        icon: new AMap.Icon({
        //终点图标
        size: new AMap.Size(25, 34),
        image: require('../assets/img/1.png'),
        imageSize: new AMap.Size(135, 40),
        imageOffset: new AMap.Pixel(-95, -3)
      }),
        position: that.marks[2],
        title: '标记点3',
    }];
    // var markerContent = '' +
    //     '<div class="custom-content-marker">' +
    //     '   <img src="//a.amap.com/jsapi_demos/static/demo-center/icons/dir-via-marker.png">' +
    //     '   <div class="close-btn" onclick="this.clearMarker()">X</div>' +
    //     '</div>';

      markers.forEach((mark, index) => {
        let markval = new AMap.Marker({
          map: that.map,
          //小车出初始位置
          position: mark.position,
          icon: mark.icon,
          title: mark.title,
          // content: markerContent,
          offset: new AMap.Pixel(-26, -13),
        })
          markval.content = '我是第' + (index + 1) + '个Marker黄金时刻感受看过';
          markval.on('click', that.markerClick);
          // markval.emit('click', {target: markval});
        // markval.setLabel({
        // offset: new AMap.Pixel(20, 20),  //设置文本标注偏移量
        // content: `<div class='info'>${mark.title}</div>`, //设置文本标注内容
        // direction: 'right' //设置文本标注方位
        // });
      });
      // markers.forEach((text) => {
      //   let value = new AMap.Text({
      //     map: that.map,
      //     position: text.position,
      //     title: '纯文本标记11',
      //     // offset: new AMap.Pixel(-26, -13),
      //     anchor:'center', // 设置文本标记锚点
      //     cursor:'pointer',
      //     style:{
      //      'padding': '.75rem 1.25rem',
      //       'margin-bottom': '1rem',
      //       'border-radius': '.25rem',
      //       'background-color': 'white',
      //       'width': '15rem',
      //       'border-width': 0,
      //       'box-shadow': '0 2px 6px 0 rgba(114, 124, 245, .5)',
      //       'text-align': 'center',
      //       'font-size': '20px',
      //       'color': 'blue'
      //     },
      //   })
      //   console.log('text--', value)
      // });
      // that.marker.setMap(that.map);
    // new AMap.Text({
    //     text: markers[0].title,
    //     anchor:'center', // 设置文本标记锚点
    //     map: that.map,
    //     cursor:'pointer',
    //     style:{
    //         'padding': '.75rem 1.25rem',
    //         'margin-bottom': '1rem',
    //         'border-radius': '.25rem',
    //         'background-color': 'white',
    //         // 'width': '15rem',
    //         'border-width': 0,
    //         'box-shadow': '0 2px 6px 0 rgba(114, 124, 245, .5)',
    //         'text-align': 'center',
    //         'font-size': '20px',
    //         'color': 'blue'
    //     },
    //     position: that.marks[0]
    // });

    // text.setMap(that.map);
    },
    // 更改标记点
    updateMarker() {

    },
    // 删除标记点
    clearMarker() {
      alert('shanchu')
      // this.map.remove()
    },
    // 更新标记点内容
    updateContent() {

    },
    markerClick(e) {
      let that = this
      that.infoWindow.setContent(e.target.content);
      that.infoWindow.open(that.map, e.target.getPosition());
    },
    // 绘制折线
    drawPolyline() {
      let mouseTool = new AMap.MouseTool(this.map)
      this.curPolyline = mouseTool.polyline({
        strokeColor: "#3366FF", 
        strokeOpacity: 1,
        strokeWeight: 6,
        // 线样式还支持 'dashed'
        strokeStyle: "solid",
        // strokeStyle是dashed时有效
        // strokeDasharray: [10, 5],
      })
     
    },
    // 开始编辑折线
    editPolyline() {
      console.log('this.polyEditor')
      let that = this
      var path = [
        [116.362209, 39.887487],
        [116.422897, 39.878002],
        [116.372105, 39.90651],
        [116.428945, 39.89663]
    ];

    that.curPolyline = new AMap.Polyline({
        path: path,
        isOutline: true,
        outlineColor: '#ffeeff',
        borderWeight: 3,
        strokeColor: "#3366FF", 
        strokeOpacity: 1,
        strokeWeight: 6,
        // 折线样式还支持 'dashed'
        strokeStyle: "solid",
        // strokeStyle是dashed时有效
        strokeDasharray: [10, 5],
        lineJoin: 'round',
        lineCap: 'round',
        zIndex: 50,
    })

    that.curPolyline.setMap(that.map)
    // 缩放地图到合适的视野级别
    that.map.setFitView([ that.curPolyline ])
      
       that.polyEditor = new AMap.PolyEditor(that.map, that.curPolyline)
      console.log('that', that.polyEditor)
    that.polyEditor.on('addnode', function(event) {
      console.log('触发事件：addnode', event)
    })

    that.polyEditor.on('adjust', function(event) {
      console.log('触发事件：adjust', event)
    })

    that.polyEditor.on('removenode', function(event) {
      console.log('触发事件：removenode', event)
    })

    that.polyEditor.on('end', function(event) {
      console.log('触发事件： end', event)
        // event.target 即为编辑后的折线对象
    })
      this.polyEditor.open()
    },
    // 结束编辑折线
    finishPolyline() {
      this.polyEditor.close()
    },
  },
  // 销毁地图
  beforeDestroy() {
    this.map && this.map.destroy()
  }
};
</script>

<style scoped>
.selecte-date-tit {
  font-size: 15px;
  font-weight: 550;
}
</style>
<style>
.info {
  border: 1px solid red;
}
</style>
