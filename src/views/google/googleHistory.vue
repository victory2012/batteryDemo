<template>
  <div class="all">
    <div class="mapcontainer">
      <div class="control">
        <div class="date">
          <el-date-picker v-model="starts" type="datetime" :placeholder="$t('history.startTime')">
          </el-date-picker>
          <el-date-picker v-model="endtime" type="datetime" :placeholder="$t('history.endTime')">
          </el-date-picker>
          <!-- <vue-datepicker-local v-model="starts"
            clearable
            :placeholder="$t('history.startTime')"
            format="YYYY-MM-DD HH:mm:ss"
            show-buttons
            @confirm="selectedDate" />
          <vue-datepicker-local v-model="endtime"
            format="YYYY-MM-DD HH:mm:ss"
            clearable
            :placeholder="$t('history.endTime')"
            show-buttons
            @confirm="selectedDate" /> -->
          <el-button size="mini" type="primary" plain @click="selectedDate">
            {{$t('history.query')}}
          </el-button>
          <el-button v-show="trajectory" size="mini" plain @click="startMove" :title="$t('history.start')">
            <i class="iconfont icon-ic_song_next"></i>
          </el-button>
          <el-button v-show="trajectory" type="danger" size="small" @click="heatmap">{{$t('history.heatActive')}}</el-button>
          <el-button v-show="active" type="primary" size="mini" @click="track">{{$t('history.TrackReplay')}}</el-button>
        </div>
      </div>
      <div class="timeRange" v-show="trajectory">
        <span>{{$t('history.times')}}(s)</span>
        <el-slider :max='max' :min="min" v-model="timeSeconds" @change="speedChange" vertical height="200px">
        </el-slider>
      </div>
      <div id="mapcontainer" class="map"></div>
      <div class="HisMask" v-show="mapLoading" v-loading="mapLoading"></div>
    </div>
    <div class="panel" v-loading="loading">
      <h2>{{$t('history.batteryList')}}</h2>
      <div class="panelTop">
        <ul class="list_warp">
          <li v-for="(item, index) in pointerArr" :class="[ devicelabel == item.batteryId ? 'selected': '',devicelabel == item.deviceId ? 'selected': '' ]" :key="item.deviceId" @click="checkItem(item)">
            <span style="margin-right:5px;">{{index+1}}、{{item.batteryId}}</span>
          </li>
        </ul>
      </div>
      <div class="page">
        <el-pagination @current-change="pageChange" :current-page.sync="pageNum" small layout="prev, pager, next" :total="total">
        </el-pagination>
      </div>
      <div class="checkTime">
        <ul>
          <li v-for="(key, index) in blockArr" @click="showThisData(key, index, $event)" :class="[{'yollew': key.bgColor === 'yellow'},{'gray': key.bgColor === 'gray'},{'green': key.bgColor === 'green'}]" :key="key.id"></li>
        </ul>
        <div v-show="showTimeDetail" class="blockInfo">
          <div class="close">
            <i class="el-icon-close" @click="closeTimeDetail"></i>
          </div>
          <div class="blockInfo_warp">
            <div v-for="item in activePointer" :key="item.createTime">{{item.dateFormat}}: {{item.onlineStatus}}</div>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>
<script>
import google from "google";
import { GetTrajectory, GetDeviceList, timeList } from "../../api/index.js";
import {
  timeFormatSort,
  timeFormats,
  timeFormat,
  getTime,
  yesTody
} from "../../utils/transition.js";
import { onWarn, onError } from "../../utils/callback.js";
var map;
let heatmapData;
let line;
let animate1;
let pointArr = [];
export default {
  data () {
    return {
      mapLoading: false,
      loading: false,
      trajectory: false,
      active: true,
      navg: null,
      map: null,
      devicelabel: "",
      starts: yesTody(),
      endtime: new Date(),
      chooseTime: [],
      pointerArr: [],
      gridData: [],
      markerArr: [],
      alldistance: 0,
      timeSeconds: 50,
      max: 100,
      min: 1,
      pageNum: 1,
      total: 0,
      localMakerArr: [],
      markerPointer: {
        sdPointer: [],
        mapPointer: []
      },
      blockArr: [],
      queryDevice: null,
      showTimeDetail: false,
      activePointer: []
    };
  },
  mounted () {
    this.init();
  },
  methods: {
    /* 点击小格子 事件 */
    showThisData (key, index, event) {
      this.activePointer = [];
      pointArr.forEach(key => {
        if (key.index === index + 1) {
          this.showTimeDetail = true;
          this.activePointer.push(key);
        }
      });
    },
    closeTimeDetail () {
      this.showTimeDetail = false;
    },
    // 通过设备id 来查看对应的上下线时间数据
    getTimeList (id) {
      let param = {
        deviceId: id,
        startTime: timeFormat(this.starts),
        endTime: timeFormat(this.endtime)
      };
      timeList(param).then(res => {
        const result = res.data;
        console.log(result);
        if (result.code === 0) {
          /* getTime
            // 返回开始时间到结束时间的毫秒数 getTime第一个参数:开始时间  第二个:结束时间；
            // {100000} 是毫秒数1000 * 100个格子。代表每个小格子代表的时间区间,单位是 秒；向上取整；
            最终返回的是 每个格子所代表的时间段；
          */
          let perBlock = getTime(this.starts, this.endtime);
          let arrs = [];
          this.blockArr = [];
          pointArr = [];
          if (result.data.length > 0) {
            result.data.forEach(key => {
              key.createTime = new Date(key.createTime).getTime();
              key.dateFormat = timeFormats(key.createTime);
              key.pre = new Date(key.createTime) - new Date(this.starts);
              key.index = Math.ceil(key.pre / perBlock); // 得出此时间是处于第几个格子； 向上取整；
              key.onlineStatus =
                key.status === 0
                  ? this.$t("history.offLine")
                  : this.$t("history.online");
              pointArr.push(key);
            });
            let bgColor = pointArr[0].status === 0 ? "green" : "gray";
            this.onlineStatus = null;
            /* {100} 是要循环100个小格子  */
            for (let i = 0; i < 100; i++) {
              let obj = {};
              obj.bgColor = bgColor;
              obj.startTime = new Date(this.starts).getTime() + perBlock * i;
              obj.endTime =
                new Date(this.starts).getTime() + perBlock * (i + 1);
              obj.id = i;
              arrs.push(obj);
              if (this.onlineStatus && this.onlineStatus != null) {
                arrs[i].bgColor = this.onlineStatus === "0" ? "gray" : "green";
              }
              pointArr.forEach(key => {
                if (
                  arrs[i].endTime - key.createTime > 0 &&
                  key.createTime - arrs[i].startTime > 0
                ) {
                  arrs[i].bgColor = "yellow";
                  this.onlineStatus = key.status;
                }
              });
            }
          } else {
            for (let i = 0; i < 100; i++) {
              let obj = {};
              obj.bgColor = "gray";
              obj.startTime = new Date(this.starts).getTime() + perBlock * i;
              obj.endTime =
                new Date(this.starts).getTime() + perBlock * (i + 1);
              obj.id = i;
              obj.dateFormat = timeFormats(obj.endTime);
              arrs.push(obj);
            }
          }
          this.blockArr = arrs;
        }
      });
    },
    speedChange () {
      let speed = this.gridData.length / this.timeSeconds;
      this.haomiao = 1000 / speed;
      this.animateCircle(this.haomiao);
    },
    pageChange () {
      this.blockArr = [];
      let pageObj = {
        pageNum: this.pageNum,
        pageSize: 10
      };
      this.getHisData(pageObj);
    },
    /* 时间确认按钮 */
    selectedDate (date) {
      if (!this.starts) {
        onWarn(`${this.$t("history.startTime")}`);
        return;
      }
      if (!this.endtime) {
        onWarn(`${this.$t("history.endTime")}`);
        return;
      }
      const startTime = new Date(this.starts).getTime();
      const endTime = new Date(this.endtime).getTime();
      if (startTime > endTime) {
        onWarn(`${this.$t("history.checkErr")}`);
        return;
      }
      // if (endTime - startTime > 86400000) {
      //   onWarn(`${this.$t("history.timeErr")}`);
      //   return;
      // }
      let opts = {
        pushDateStart: timeFormatSort(this.starts),
        pushDateEnd: timeFormatSort(this.endtime)
      };
      opts.batteryId = this.devicelabel;
      this.getTimeList(this.queryDevice);
      this.clearMap();
      this.getData(opts);
    },
    /* 获取数据 */
    getData (params) {
      this.mapLoading = true;
      GetTrajectory(params).then(res => {
        console.log(res);
        this.mapLoading = false;
        if (res.data.code === 0) {
          let result = res.data.data;
          // console.log(result);
          this.lineArr = [];
          if (result.length > 0) {
            this.gridData = [];
            for (let i = 0; i < result.length; i++) {
              var key = result[i];
              var obj = {};
              obj.pushTime = key.pushTime;
              obj.ponter = new google.maps.LatLng(key.latitude, key.longitude);
              this.lineArr.push(obj);
              if (Number(key.latitude) > 1 && Number(key.longitude) > 1) {
                this.gridData.push(
                  new google.maps.LatLng(key.latitude, key.longitude)
                );
              }
            }
            map.setCenter(this.gridData[0]);
            if (this.active) {
              this.heatmap();
            } else {
              this.track();
            }
          } else {
            onWarn(`${this.$t("history.noData")}`);
          }
        }
      });
    },
    heatmap () {
      this.mapLoading = true;
      this.trajectory = false;
      this.active = true;
      this.clearMap();
      heatmapData = new google.maps.visualization.HeatmapLayer({
        data: this.gridData,
        map: map,
        radius: 10,
        opacity: 1,
        maxIntensity: 15,
        dissipating: true,
        gradient: [
          "rgba(0, 0, 255, 0)",
          "rgba(55, 55, 255, 1)",
          "rgba(0, 255, 120, 1)",
          "rgba(18, 255, 0, 1)",
          "rgba(150, 255, 0, 1)",
          "rgba(210, 255, 0, 1)",
          "rgba(255, 228, 0, 1)",
          "rgba(255, 216, 0, 1)",
          "rgba(255, 132, 0, 1)",
          "rgba(255, 72, 0, 1)",
          "rgba(255, 48, 0, 1)",
          "rgba(234, 86, 61, 1)",
          "rgba(255, 36, 0, 1)",
          "rgba(255, 0, 0, 1)"
        ]
      });
      this.mapLoading = false;
      // heatmapData.set("gradient", gradient);
    },
    init () {
      try {
        map = new google.maps.Map(document.getElementById("mapcontainer"), {
          center: {
            lat: 0,
            lng: 0
          },
          zoom: 15
        });
        this.getHisData();
        // var geocoders = new google.maps.Geocoder();
        map.addListener("click", e => {
          if (this.localMakerArr.length > 0) {
            this.localMakerArr.forEach(key => {
              key.setMap(null);
            });
          }
          var latLngData =
            e.latLng.lat().toFixed(6) + "," + e.latLng.lng().toFixed(6);
          var localMaker = new google.maps.Marker({
            position: e.latLng,
            icon: {
              path: google.maps.SymbolPath.CIRCLE,
              scale: 3,
              strokeColor: "red"
            },
            map: map
          });
          this.localMakerArr.push(localMaker);
          this.$.ajax({
            type: "post",
            url:
              "https://maps.googleapis.com/maps/api/geocode/json?latlng=" +
              latLngData +
              "&key=AIzaSyAz6eHKxmBqalyW0LVFjs9mugr5t0PxvYI&fields=formatted_address",
            async: true,
            success: function (data) {
              console.log(data);
              let site = `${this.$t(
                "history.latLng"
              )}：${latLngData}<br />${this.$t("history.address")}：${
                data.results[0].formatted_address
              }`;
              // "坐标：" +
              // latLngData +
              // "<br />" +
              // "地址：" +
              // data.results[0].formatted_address;
              var infowindow = new google.maps.InfoWindow({
                content: site
              });
              infowindow.open(map, localMaker); // 弹出信息提示窗口
              map.addListener("click", () => {
                infowindow.close();
              });
            }
          });
        });
      } catch (err) {
        onError(`${this.$t("mapError")}`);
      }
    },
    // 获取列表数据
    getHisData () {
      let pageObj = {
        pageNum: this.pageNum,
        pageSize: 10,
        bindingStatus: "1"
      };
      this.loading = true;
      GetDeviceList(pageObj).then(res => {
        console.log("GetDeviceList", res);
        this.loading = false;
        if (res.data && res.data.code === 0) {
          let result = res.data.data.data;
          this.total = res.data.data.total;
          this.pointerArr = [];
          if (result.length > 0) {
            this.batteryId = this.$route.query.batteryId; // 路由参数
            result.forEach(key => {
              if (key.batteryId) {
                if (this.batteryId && this.batteryId === key.batteryId) {
                  this.queryDevice = key.deviceId; // 根据路由参数中的电池id 获取对应的设备id；
                }
                this.pointerArr.push(key);
              }
            });
            let params = {
              pushDateStart: timeFormatSort(this.starts),
              pushDateEnd: timeFormatSort(this.endtime)
            };
            if (this.batteryId && this.pageNum === 1) {
              this.devicelabel = this.batteryId;
              params.batteryId = this.batteryId;
              this.getData(params);
            } else {
              this.devicelabel = result[0].batteryId;
              params.batteryId = result[0].batteryId;
              this.queryDevice = result[0].deviceId;
              this.getData(params);
            }
            this.getTimeList(this.queryDevice);
          } else {
            onWarn(`${this.$t("history.noDevice")}`);
          }
        }
      });
    },
    startMove () {
      this.animateCircle(this.timeSeconds);
    },
    animateCircle (times) {
      let seconds = times || 10;
      var count = 0;
      animate1 && clearInterval(animate1);
      animate1 = window.setInterval(() => {
        count = count + 5;
        // console.log(count)
        var icons = line.get("icons");
        icons[0].offset = (count / this.gridData.length) * 100 + "%";
        line.set("icons", icons);
        console.log(count, icons[0].offset);
        // // //终点
        if (count >= this.gridData.length) {
          clearInterval(animate1);
        }
      }, seconds);
    },
    // 历史轨迹 轨迹配置
    track () {
      this.trajectory = true;
      this.active = false;
      this.clearMap();
      this.mapLoading = true;
      line = new google.maps.Polyline({
        icons: [
          {
            icon: {
              path: google.maps.SymbolPath.FORWARD_CLOSED_ARROW,
              scale: 3,
              strokeColor: "#1200ff"
            },
            offset: "0%"
          }
        ],
        strokeColor: "#71253e",
        map: map
      });
      line.setPath(this.gridData);
      let TimerMax = Math.ceil(this.gridData.length * 0.01);
      if (TimerMax > this.max) {
        this.max = Math.ceil(TimerMax * 2);
      }
      this.min = TimerMax;
      this.timeSeconds = TimerMax;
      let start = new google.maps.Marker({
        position: this.gridData[0],
        label: {
          color: "#FFF",
          text: "S"
        },
        map: map
      });
      let end = new google.maps.Marker({
        position: this.gridData[this.gridData.length - 1],
        label: {
          color: "#FFF",
          text: "E"
        },
        map: map
      });
      this.mapLoading = false;
      this.markerPointer.sdPointer.push(start);
      this.markerPointer.sdPointer.push(end);
    },
    // 清除地图上的覆盖物
    clearMap () {
      animate1 && clearInterval(animate1);
      heatmapData && heatmapData.setMap(null);
      line && line.setMap(null);
      if (this.markerPointer.sdPointer.length > 0) {
        this.markerPointer.sdPointer.forEach(marker => {
          marker.setMap(null);
        });
        this.markerPointer.sdPointer = [];
      }
      if (this.markerPointer.mapPointer.length > 0) {
        this.markerPointer.mapPointer.forEach(marker => {
          marker.setMap(null);
        });
        this.markerPointer.mapPointer = [];
      }
      if (this.localMakerArr.length > 0) {
        this.localMakerArr.forEach(key => {
          key.setMap(null);
        });
      }
    },
    // 列表点击事件
    checkItem (item) {
      this.activePointer = [];
      this.blockArr = [];
      animate1 && clearInterval(animate1);
      this.clearMap();
      let params = {
        pushDateStart: timeFormatSort(this.starts),
        pushDateEnd: timeFormatSort(this.endtime)
      };
      params.batteryId = item.batteryId;
      this.devicelabel = item.batteryId;
      this.queryDevice = item.deviceId;
      this.getData(params);
      this.getTimeList(this.queryDevice);
    }
  },
  beforeDestroy () {
    animate1 && clearInterval(animate1);
  }
};
</script>
<style lang="scss" scoped>
.all {
  position: relative;
  height: calc(100vh - 110px);
  width: 100%;
  display: flex;
  .mapcontainer {
    position: relative;
    // top: 0;
    // bottom: 0;
    flex: 1;
    .map {
      height: 100%;
      width: 100%;
    }
    .HisMask {
      position: absolute;
      top: 0;
      left: 0;
      bottom: 0;
      right: 0;
      z-index: 999;
      // background: rgba(0, 0, 0, 0.4);
    }
    .control {
      position: absolute;
      top: 5px;
      right: 5px;
      padding: 5px 10px;
      border-radius: 3px;
      box-shadow: 0 0 15px #000000;
      background: #ffffff;
      line-height: 16px;
      z-index: 999;
      .date {
        font-size: 16px;
      }
    }
    .timeRange {
      position: absolute;
      top: 70px;
      right: 8px;
      z-index: 1000;
      padding: 5px 4px 15px;
      box-shadow: 0 0 15px #000000;
      background: #ffffff;
      text-align: center;
      border-radius: 3px;
      span {
        font-size: 12px;
      }
    }
  }
  .panel {
    flex: 0 0 270px;
    box-sizing: border-box;
    padding: 10px 0;
    height: calc(100vh - 110px);
    background: #ffffff;
    border-left: 1px solid #f0f0f0;
    z-index: 999;
    h2 {
      padding: 0 20px;
      height: 40px;
      line-height: 40px;
      text-align: center;
      color: #409eff;
      font-size: 14px;
      font-weight: 500;
      border-bottom: 1px solid #409eff;
    }
    .panelTop {
      min-height: 412px;
      padding: 0 5px;
      overflow-x: hidden;
      background: #ffffff;
      .list_warp {
        li {
          height: 40px;
          border-bottom: 1px solid #f0f0f0;
          line-height: 40px;
          font-size: 14px;
          color: #303133;
          cursor: pointer;
          padding-left: 10px;
          &.selected {
            background: rgb(112, 191, 255);
            color: #fff;
          }
        }
      }
    }
    .page {
      padding-top: 20px;
      text-align: right;
    }
    .checkTime {
      position: relative;
      margin: 0 auto;
      width: 260px;
      height: auto;
      ul {
        display: flex;
        width: 100%;
        border-top: 1px solid #f0f0f0;
        border-left: 1px solid #f0f0f0;
        flex-wrap: wrap;
        li {
          flex: 0 0 25px;
          height: 15px;
          border: 1px solid #f0f0f0;
          border-top: none;
          border-left: none;
          list-style: none;
          cursor: pointer;
          &.gray {
            background: gray;
          }
          &.green {
            background: green;
          }
          &.yollew {
            background: rgb(226, 213, 26);
          }
        }
      }
      .blockInfo {
        position: absolute;
        width: 240px;
        left: 10px;
        top: -10px;
        height: auto;
        max-height: 150px;
        overflow-y: auto;
        overflow-x: hidden;
        background: #ffffff;
        box-shadow: 0 0 10px rgba(0, 0, 0, 0.8);
        .blockInfo_warp {
          border-radius: 3px;
          overflow: hidden;
          div {
            box-sizing: border-box;
            width: 100%;
            font-size: 14px;
            line-height: 30px;
            list-style: none;
            padding-left: 10px;
          }
        }
        .close {
          text-align: right;
          padding: 2px 5px;
          cursor: pointer;
        }
      }
    }
  }
}
</style>
