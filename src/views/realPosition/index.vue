<template>
  <div id="outer-box">
    <!-- <div id="positions" v-loading="maploading" class="positioned"></div> -->
    <!-- <v-gaode :mapData='markerData' :mapCenter='mapCenterPoniter'></v-gaode> -->
    <gaode-map :mapData="markerData"></gaode-map>
    <div id="panel">
      <div class="panelTop" v-loading="loading">
        <div id="intro" class="intro">
          <h3>
            <span>{{titles}}</span>
            <el-button @click="showAllPionter" type="text" mini>{{$t('positions.lookAll')}}</el-button>
          </h3>
        </div>
        <ul class="list_warp">
          <li v-for="(item, index) in pointerArr" :class="[ devicelabel == item.deviceId ? 'selected': '', item.onlineStatus === 0? 'off': '', devicelabel == item.batteryId ? 'selected': '' ]" :key="item.deviceId" @click="checkItem(item, index)">
            <p><span class="listIndex">{{index + 1}}、</span>{{deviceShow? item.deviceId : item.batteryId}}</p>
            <el-badge :value="item.onLine" class="item">
              <el-button @click.prevent.stop="HistoryTrack(item.batteryId)" size="mini">{{$t('positions.track')}}</el-button>
            </el-badge>
          </li>
        </ul>
      </div>
      <div class="page">
        <el-pagination @current-change="pageChange" :current-page.sync="pageNum" small layout="prev, pager, next" :total="total">
        </el-pagination>
      </div>
    </div>
  </div>
</template>
<style scoped>
.list_warp {
  border-top: 1px solid #f0f0f0;
  min-height: 510px;
}
.listIndex {
  display: inline-block;
  width: 22px;
}
.list_warp li {
  position: relative;
  height: 50px;
  border-bottom: 1px solid #f0f0f0;
  line-height: 50px;
  font-size: 14px;
  color: #303133;
  cursor: pointer;
  padding-left: 10px;
}
.list_warp li.off {
  cursor: not-allowed;
}
.off .el-badge__content {
  background: #f0f0f0;
  color: #b3b2b2;
}
.list_warp .el-badge__content.is-fixed {
  top: 13px;
  right: 12px;
}
.list_warp li .item {
  position: absolute;
  top: 0;
  right: 34px;
}
.list_warp li p {
  width: 130px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 12px;
}
.list_warp .selected {
  background: rgb(112, 191, 255);
  color: #fff;
}
.positioned {
  width: 100%;
  height: calc(100vh - 110px);
}
.deviceList {
  font-size: 14px;
  max-height: 600px;
  overflow-y: auto;
}

.intro h3 {
  position: relative;
  padding-left: 8px;
  font-weight: normal;
  font-size: 16px;
  margin-bottom: 10px;
  height: 30px;
  line-height: 30px;
}
.intro h3 button {
  position: absolute;
  top: 0;
  right: 20px;
}
#outer-box {
  height: 100%;
  padding-right: 270px;
}
#panel {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 270px;
  box-sizing: border-box;
  padding: 10px 0;
  height: calc(100vh - 110px);
  background: #ffffff;
  border-left: 1px solid #f0f0f0;
  z-index: 999;
}
.panelTop {
  height: auto;
  padding: 0 5px;
  overflow-x: hidden;
  background: #ffffff;
}
.page {
  padding-top: 20px;
  text-align: right;
}
</style>
<script>
import { websockets, GetDeviceList } from "../../api/index.js";
import { trakTimeformat, nowDate } from "../../utils/transition.js";
import gaodeMap from "./map";

let batteryIdArr = {};
export default {
  components: {
    gaodeMap
  },
  data () {
    return {
      mapType: 0,
      dataForMap: [],
      loading: true,
      maploading: false,
      pointerArr: [],
      devicelabel: "",
      active: true,
      activeName: 1,
      pageNum: 1,
      total: 10,
      onLineData: [],
      titles: "",
      deviceShow: false,
      pathParams: "", // url 中设备id 参数
      mapCenterPoniter: "",
      markers: [],
      markerData: {}
    };
  },
  methods: {
    pageChange (val) {
      this.over();
      this.pageNum = val;
      // this.markers && map.remove(this.markers);
      this.getListData();
    },
    init () {
      // const lang = sessionStorage.getItem("locale") === "en" ? "en" : "zh_cn";
      // map = new AMap.Map("positions", {
      //   resizeEnable: true,
      //   lang: lang,
      //   zoom: 15
      // });
      // geocoder = new AMap.Geocoder({
      //   city: "", // 城市设为北京，默认：“全国”
      //   lang: lang,
      //   batch: false,
      //   radius: 500 // 范围，默认：500
      // });
      this.getListData();
    },
    // 获取列表数据
    getListData () {
      let pageObj = {
        pageNum: this.pageNum,
        pageSize: 10
      };
      if (this.deviceShow) {
        pageObj.bindingStatus = "";
      } else {
        pageObj.bindingStatus = 1;
      }
      this.loading = true;
      GetDeviceList(pageObj).then(res => {
        console.log(res.data);
        this.loading = false;
        if (res.data.code === 0) {
          this.pointerArr = [];
          let result = res.data.data.data;
          this.total = res.data.data.total;
          let sendData = { api: "bind", param: [] };
          // pointerObj = {};
          this.dataForMap = [];
          if (result.length > 0) {
            if (this.pathParams && this.pageNum === 1) {
              result.forEach((key, index) => {
                // pointerObj[key.deviceId] = `${key.longitude},${
                //   key.latitude
                // },${trakTimeformat(key.pushTime)},${key.batteryId},${
                //   key.onlineStatus
                // },0,${index + 1},${key.voltage}`; // pointerObj 对象。其key为设备id（唯一性），value为字符串、依次顺序为经度、纬度、时间、电池id、在线状态
                let obj = {
                  lngs: key.longitude,
                  lat: key.latitude,
                  batteryId: key.batteryId,
                  time: trakTimeformat(key.pushTime),
                  onlineStatus: key.onlineStatus,
                  pushStatus: 0,
                  voltage: key.voltage,
                  deviceId: key.deviceId,
                  index: index + 1
                };
                if (key.onlineStatus === 1) {
                  key.onLine = this.$t("positions.onLine");
                  // pathParams 路由传参。为设备id
                } else {
                  key.onLine = this.$t("positions.offline");
                }
                if (key.deviceId) {
                  sendData.param.push(key.deviceId);
                }
                if (key.batteryId) {
                  batteryIdArr[key.deviceId] = key.batteryId; // 制作电池id 字典。以设备id作为key，电池id作为value。
                }
                if (this.pathParams === key.deviceId) {
                  // let deviceId = this.pathParams;
                  this.checkItem(key, index);
                }
                this.dataForMap.push(obj);
                this.pointerArr.push(key);
              });
              this.sockets(JSON.stringify(sendData));
            } else {
              this.mapInit(result);
            }
          }
        }
      });
    },
    // websockets 请求
    sockets (data) {
      this.WX = websockets();
      this.WX.onopen = () => {
        console.log("open....");
        this.WX.send(data);
      };
      this.WX.onmessage = evt => {
        let data = JSON.parse(evt.data);
        console.log(data);

        if (data.code === 2) {
          // code 为 1时 既绑定成功，2时为 收到了数据
          let obj = data.data.split(",");
          let batteryId = batteryIdArr[obj[0]]; // 从电池id 字典中获取电池id，obj[0] 为设备id。
          // let pointerObjKeys = Object.keys(pointerObj);
          // let ponterIndexs = pointerObjKeys.indexOf(obj[0]);

          this.dataForMap.forEach((key, index) => {
            if (key.batteryId === batteryId) {
              this.dataForMap[index] = {
                lngs: obj[2],
                lat: obj[1],
                batteryId: batteryId,
                time: nowDate(),
                onlineStatus: 1,
                pushStatus: 1,
                voltage: obj[3],
                deviceId: obj[0],
                index: index + 1
              };
            }
          });
          // obj.forEach(() => {
          //   // pointerObj[obj[0]] = `${obj[2]},${
          //   //   obj[1]
          //   // },${nowDate()},${battery},1,1,${ponterIndexs + 1},${obj[3]}`;

          //   // pointerObj 对象。其key为设备id（唯一性），value为字符串、
          //   // 依次顺序为 经度、纬度、时间、电池id、在线状态、推送数据标志, 电压
          // });
          if ((this.deviceId || this.pathParams) && !this.viewAll) {
            // let keys = Object.keys(pointerObj);
            // let nextObj = {};
            this.dataForMap.forEach(item => {
              if (item === this.deviceId || item === this.pathParams) {
                this.markerData = {
                  data: item,
                  type: "click"
                };
              }
            });
            // this.GaoDeMap(nextObj, "fromClick");
          } else {
            // this.GaoDeMap(pointerObj, "fromWs");
            this.markerData = {
              data: this.dataForMap,
              type: "normal"
            };
          }
        }
      };
      this.WX.onerror = () => {
        console.log("onerror...");
        this.over();
      };
      this.WX.onclose = () => {
        console.log("closed...");
      };
    },
    over () {
      this.WX.close();
    },
    /*
     @params batteryIdArr 为电池ID对象 key为设备id，value为电池id
     @params pointerObj 电池坐标点对象，key为设备id，value为一个字符串，依次顺序为经度、纬度、时间、电池id。以逗号隔开
     */
    mapInit (data) {
      // console.log("data ===>>>", data);
      // pointerObj = {};
      this.dataForMap = [];
      let sendData = { api: "bind", param: [] };
      data.forEach((key, index) => {
        // pointerObj[key.deviceId] = `${key.longitude},${
        //   key.latitude
        // },${trakTimeformat(key.pushTime)},${key.batteryId},${
        //   key.onlineStatus
        // },0,${index + 1},${key.voltage}`;
        let obj = {
          lngs: key.longitude,
          lat: key.latitude,
          batteryId: key.batteryId,
          time: trakTimeformat(key.pushTime),
          onlineStatus: key.onlineStatus,
          pushStatus: 0,
          voltage: key.voltage,
          deviceId: key.deviceId,
          index: index + 1
        };
        if (key.onlineStatus === 1) {
          // onlineStatus 判断是否在线的标识。1 在线。0 离线；
          key.onLine = this.$t("positions.onLine");
        } else {
          key.onLine = this.$t("positions.offline");
        }
        if (key.batteryId) {
          sendData.param.push(key.deviceId);
          batteryIdArr[key.deviceId] = key.batteryId; // 制作电池id 字典。以设备id作为key，电池id作为value。
        }
        this.dataForMap.push(obj);
        this.pointerArr.push(key);
      });
      // console.log("mapInit ===>>>", pointerObj);
      this.sockets(JSON.stringify(sendData));
      // this.GaoDeMap(pointerObj);
      this.markerData = {
        data: this.dataForMap,
        type: "normal"
      };
    },
    /*
    * @params deviceId 电池列表 获取的设备id。
    * @params index 为列表的索引。这里取这个索引是为了让地图的mark点 显示点的是第几个。
     */
    checkItem (item, index) {
      if (item.onlineStatus === 0) return;
      console.log(item);
      if (item.longitude && item.latitude) {
        // this.mapCenterPoniter = new AMap.LngLat(item.longitude, item.latitude);
      }
      this.viewAll = false;
      this.devicelabel = item.deviceId;
      this.deviceId = item.deviceId;
      // ponterIndex = index + 1;
      // infoWindow && infoWindow.close(); // infoWindow 高德地图 数据展示框。
      if (this.deviceId && this.dataForMap) {
        // let keys = Object.keys(pointerObj);
        // let selectObj = {};
        this.selectObj = [];
        this.dataForMap.forEach(items => {
          if (items.deviceId === this.deviceId) {
            this.selectObj.push(items);
          }
        });
        // console.log(selectObj);
        // this.GaoDeMap(selectObj, "fromClick");
        this.markerData = {
          data: this.selectObj,
          type: "normal"
        };
      }
    },
    // GaoDeMap(data, fromWs) {
    //   this.markers && map.remove(this.markers);
    //   let allmarkerArr = Object.values(data);
    //   let markerkeys = Object.keys(data);
    //   for (let i = 0; i < allmarkerArr.length; i++) {
    //     var lngs = allmarkerArr[i].toString().split(",");
    //     if (lngs[0].length > 6 && lngs[1].length > 6 && lngs[4] === "1") {
    //       var marker = new AMap.Marker({
    //         position: [lngs[0], lngs[1]],
    //         offset: new AMap.Pixel(-12, -12),
    //         zIndex: 101,
    //         extData: {
    //           position: `${lngs[0]},${lngs[1]}`,
    //           center: new AMap.LngLat(lngs[0], lngs[1]),
    //           times: lngs[2]
    //         },
    //         map: map
    //       });
    //       if (lngs[5] === "0") {
    //         marker.setIcon("../../../static/img/gray.png");
    //       } else {
    //         marker.setIcon(
    //           `http://webapi.amap.com/theme/v1.3/markers/n/mark_b${i + 1}.png`
    //         );
    //       }
    //       if (fromWs === "fromClick") {
    //         map.setCenter(new AMap.LngLat(lngs[0], lngs[1]));
    //         marker.setIcon(
    //           `http://webapi.amap.com/theme/v1.3/markers/n/mark_r${lngs[6]}.png`
    //         );
    //       }
    //       let voltage = lngs[7];
    //       let content;
    //       if (voltage === "null") {
    //         content = `${this.$t("positions.batteryCode")}：${
    //           lngs[3]
    //         }<br/>${this.$t("positions.deviceCode")}：${markerkeys[i]}`;
    //       } else {
    //         content = `${this.$t(
    //           "positions.voltage"
    //         )}：${voltage}<br/>${this.$t("positions.batteryCode")}：${
    //           lngs[3]
    //         }<br/>${this.$t("positions.deviceCode")}：${markerkeys[i]}`;
    //       }
    //       marker.setLabel({
    //         offset: new AMap.Pixel(15, 20),
    //         content: content
    //       });
    //       this.markers.push(marker);
    //     }
    //   }
    //   if (!fromWs) {
    //     map.setFitView(); // 自适应地图
    //   }
    //   if (this.markers.length > 0) {
    //     this.markers.forEach((key, index) => {
    //       key.on("click", () => {
    //         let pointerData = key.getExtData();
    //         // console.log(key);
    //         const self = this;
    //         geocoder.getAddress(pointerData.center, function(status, result) {
    //           if (status === "complete" && result.regeocode) {
    //             let address = result.regeocode.formattedAddress;
    //             console.log(result);
    //             var info = [];
    //             info.push(
    //               `<div><div>${self.$t("positions.updateTime")}：${
    //                 pointerData.times
    //               }</div>`
    //             );
    //             info.push(
    //               `<div style="font-size:14px;">${self.$t(
    //                 "positions.address"
    //               )} :${address}</div></div>`
    //             );
    //             infoWindow = new AMap.InfoWindow({
    //               content: info.join("<br/>"), // 信息窗体框，显示信息内容
    //               autoMove: false,
    //               offset: new AMap.Pixel(0, -10)
    //             });
    //             infoWindow.open(map, pointerData.center);
    //           }
    //         });
    //       });
    //     });
    //   }
    //   map.on("click", () => {
    //     infoWindow && infoWindow.close();
    //   });
    // },
    // 查看所有点
    showAllPionter () {
      this.viewAll = true;
      this.devicelabel = null;
      this.deviceId = null;
      // this.GaoDeMap(pointerObj);
      this.markerData = {
        data: this.dataForMap,
        type: "normal"
      };
    },
    // 查看历史轨迹。路由传参 设备id
    HistoryTrack (batteryId) {
      if (this.mapType === 0) {
        this.$router.push({
          path: "history",
          query: { batteryId: batteryId }
        });
      }
      if (this.mapType === 1) {
        this.$router.push({
          path: "googleHis",
          query: { batteryId: batteryId }
        });
      }
    }
  },
  /*
  * 用beforeRouteEnter 这个路由钩子函数 来判断是从哪个路由跳转过来的
  * 以此来处理列表显示内容
  */
  beforeRouteEnter (to, from, next) {
    next(vm => {
      if (from.name === "device" && vm.pathParams) {
        vm.titles = vm.$t("positions.title1");
        vm.deviceShow = true;
      }
      if (from.name === "batteryList" && vm.pathParams) {
        vm.titles = vm.$t("positions.title2");
        vm.deviceShow = false;
      }
    });
  },
  mounted () {
    this.mapType = Number(sessionStorage.getItem("mapType"));
    this.init();
  },
  created () {
    this.pathParams = this.$route.query.deviceId; // 路由参数
  },
  beforeDestroy () {
    this.over();
  }
};
</script>
