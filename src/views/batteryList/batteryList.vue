<template>
  <div class="batterylist">
    <div class="headerBtn">
      <el-button size="medium"
        v-show="limeted"
        @click="doCreatBattery"
        type="primary">{{$t('batteryList.batteryAddBtn')}}
        <i class="el-icon-circle-plus-outline"></i>
      </el-button>
    </div>
    <div class="table">
      <el-table v-loading="loading"
        :data="tableData"
        style="width: 100%"
        max-height="750">
        <el-table-column type="index"
          width="80"
          align="center"
          :label="$t('batteryList.serial')"></el-table-column>
        <el-table-column prop="batteryId"
          align="center"
          :label="$t('batteryList.batteryCode')">
        </el-table-column>
        <el-table-column prop="model"
          align="center"
          :label="$t('batteryList.model')">
        </el-table-column>
        <el-table-column prop="specification"
          align="center"
          :label="$t('batteryList.specif')">
        </el-table-column>
        <el-table-column prop="customerName"
          align="center"
          :label="$t('batteryList.customer')">
        </el-table-column>
        <el-table-column prop="deviceId"
          align="center"
          :label="$t('batteryList.deviceCode')">
        </el-table-column>
        <el-table-column prop="bindingName"
          align="center"
          :label="$t('batteryList.bindStatus')">
        </el-table-column>
        <el-table-column prop="online"
          align="center"
          :label="$t('batteryList.onlineStatus')">
        </el-table-column>
        <el-table-column prop="activeText"
          align="center"
          :label="$t('device.activeStatus')">
        </el-table-column>
        <el-table-column align="center"
          :label="$t('batteryList.running')">
          <template slot-scope="scope">
            <el-button @click.native.prevent="examine(scope.$index, tableData)"
              type="text"
              :disabled="!tableData[scope.$index].OLS"
              size="small">
              {{$t('batteryList.view')}}
            </el-button>
          </template>
        </el-table-column>
        <el-table-column :label="$t('alarmList.handle')"
          align="center"
          width="200">
          <template slot-scope="scope">
            <el-button @click.native.prevent="DeciveActive(scope.row)"
              type="text"
              :disabled="scope.row.alarmBtn"
              size="small">
              {{scope.row.activeBtn}}
            </el-button>
            <el-button @click.native.prevent="cancelAlarm(scope.row)"
              type="text"
              :disabled="scope.row.activeState !== 4"
              size="small">
              {{$t('device.cancelAlarm')}}
            </el-button>
          </template>
        </el-table-column>
      </el-table>
      <div class="block">
        <el-pagination @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
          :current-page.sync="currentPage"
          :page-sizes="handleSizeData"
          :page-size="handleSize"
          layout="sizes, prev, pager, next"
          :total="totalPage">
        </el-pagination>
      </div>
    </div>
    <div>
      <el-dialog :title="$t('batteryList.batteryAddBtn')"
        :visible.sync="creatBattery"
        width="600px">
        <el-form :model="BatteryForm"
          :rules="BatteryRules"
          ref="BatteryForm">
          <el-row :gutter="40">
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.enterprise')"
                prop="enterpriseName">
                <el-input v-model="BatteryForm.enterpriseName"
                  disabled
                  auto-complete="off"
                  :placeholder="$t('batteryList.warn.enterprise')"></el-input>
              </el-form-item>
            </el-col>
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.customerCom')"
                prop="customer">
                <el-select v-model="BatteryForm.customer"
                  style="width:238px">
                  <el-option v-for="item in customerList"
                    :key="item.id"
                    :label="item.name"
                    :value="item.id">
                  </el-option>
                </el-select>
                <!-- <el-input v-model="BatteryForm.customerId" auto-complete="off"></el-input> -->
              </el-form-item>
            </el-col>
          </el-row>
          <el-row :gutter="40">
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.batteryNumber')"
                prop="batteryId">
                <el-input v-model.trim="BatteryForm.batteryId"
                  auto-complete="off"
                  :placeholder="$t('batteryList.batteryNumber')"></el-input>
              </el-form-item>
            </el-col>
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.model')"
                prop="model">
                <el-input v-model.trim="BatteryForm.model"
                  auto-complete="off"
                  :placeholder="$t('batteryList.warn.model')"></el-input>
              </el-form-item>
            </el-col>
          </el-row>
          <el-row :gutter="40">
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.specif')"
                prop="specification">
                <el-input v-model.trim="BatteryForm.specification"
                  auto-complete="off"
                  :placeholder="$t('batteryList.warn.specif')"></el-input>
              </el-form-item>
            </el-col>
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.deviceCode')"
                prop="deviceId">
                <el-select v-model="BatteryForm.deviceId"
                  style="width:238px">
                  <el-option v-for="item in deviceIdList"
                    :key="item"
                    :label="item"
                    :value="item">
                  </el-option>
                </el-select>
                <!-- <el-input v-model="BatteryForm.deviceId" auto-complete="off" placeholder="请填设备编号"></el-input> -->
              </el-form-item>
            </el-col>
          </el-row>
          <el-row :gutter="40">
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.createDate')"
                prop="createDate">
                <el-date-picker v-model="BatteryForm.createDate"
                  style="width:238px"
                  type="date"
                  value-format="yyyy-MM-dd HH:mm:ss"
                  @change="formatCreatDate"
                  :placeholder="$t('batteryList.createDate')">
                </el-date-picker>
              </el-form-item>
            </el-col>
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.manufactureDate')"
                :error="manufactureDateError"
                prop="manufactureDate">
                <el-date-picker v-model="BatteryForm.manufactureDate"
                  style="width:238px"
                  type="date"
                  value-format="yyyy-MM-dd HH:mm:ss"
                  :placeholder="$t('batteryList.manufactureDate')">
                </el-date-picker>
              </el-form-item>
            </el-col>
          </el-row>
          <el-row :gutter="40">
            <el-col :span="12">
              <el-form-item :label="$t('batteryList.warrantyDate')"
                :error="warrantyDateError"
                prop="warrantyDate">
                <el-date-picker v-model="BatteryForm.warrantyDate"
                  style="width:238px"
                  type="date"
                  value-format="yyyy-MM-dd HH:mm:ss"
                  :placeholder="$t('batteryList.warrantyDate')">
                </el-date-picker>
              </el-form-item>
            </el-col>
          </el-row>
        </el-form>
        <div slot="footer"
          class="dialog-footer">
          <el-button @click="resetBatteryForm('BatteryForm')">{{$t('batteryList.cancel')}}</el-button>
          <el-button @click="regBattery('BatteryForm')"
            :loading="SureReg"
            type="primary">{{$t('batteryList.sure')}}</el-button>
        </div>
      </el-dialog>
    </div>
  </div>
</template>
<script>
import {
  addBattery,
  deviceListOnly,
  GetList,
  activeDevice,
  deleteBattery
} from "../../api/index.js";
import { onSuccess } from "../../utils/callback.js";
export default {
  data () {
    return {
      warrantyDateError: '',
      manufactureDateError: '',
      totalPage: 1, // 总数
      currentPage: 1, // 当前页
      handleSize: 10, // 每页显示条数
      handleSizeData: [10, 20, 30, 40, 50],
      tableData: [],
      loading: true,
      formLabelWidth: "120px",
      SureReg: false,
      BatteryForm: {},
      deviceIdList: [],
      deviceId: "",
      BatteryRules: {
        batteryId: [
          {
            required: true,
            message: this.$t("batteryList.warn.batteryCode"),
            trigger: "change"
          }
        ],
        model: [
          {
            required: true,
            message: this.$t("batteryList.warn.model"),
            trigger: "change"
          }
        ],
        specification: [
          {
            required: true,
            message: this.$t("batteryList.warn.specif"),
            trigger: "change"
          }
        ],
        manufactureDate: [
          {
            required: true,
            message: this.$t("batteryList.warn.manufactureDate"),
            trigger: "change"
          }
        ],
        createDate: [
          {
            required: true,
            message: this.$t("batteryList.warn.createDate"),
            trigger: "change"
          }
        ],
        warrantyDate: [
          {
            required: true,
            message: this.$t("batteryList.warn.warrantyDate"),
            trigger: "change"
          }
        ],
        deviceId: [
          {
            required: true,
            message: this.$t("batteryList.warn.deviceId"),
            trigger: "change"
          }
        ]
      },
      regForm: false,
      search: false,
      input10: "",
      creatBattery: false,
      form: {},
      customerList: [],
      CreatDatefirst: "",
      value: ""
    };
  },
  computed: {
    limeted () {
      let loginData = JSON.parse(sessionStorage.getItem("loginData"));
      return loginData.userRole !== "plat_super_admin";
    }
  },
  mounted () {
    this.getData();
  },
  methods: {
    DeciveActive (data) {
      if (data.bindingStatus !== 1) {
        return
      }
      const param = {
        batteryId: data.batteryId,
        deviceId: data.deviceId,
        originalActiveState: data.activeState
      }
      if (data.activeState === 0) {
        param.activeState = 1;
      }
      if (data.activeState === 2) {
        param.activeState = -2;
      }
      activeDevice(param).then(res => {
        console.log('active', res)
        if (res.data && res.data.code === 0) {
          this.$message({
            message: this.$t("device.activeReqSuccess"),
            type: 'success'
          })
          this.getData();
        }
      });
    },
    cancelAlarm (data) {
      if (data.activeState !== 4) {
        return
      }
      const param = {
        batteryId: data.batteryId,
        deviceId: data.deviceId,
        activeState: -4,
        originalActiveState: data.activeState
      }
      activeDevice(param).then(res => {
        console.log('active', res)
        if (res.data && res.data.code === 0) {
          this.$message({
            message: this.$t("device.activeReqSuccess"),
            type: 'success'
          })
          this.getData();
        }
      });
    },
    batteryReg () {
      this.regForm = true;
    },
    renew () {
      this.$router.push({
        path: "/blacklist"
      });
    },
    formatCreatDate (date) {
      this.CreatDatefirst = date;
    },
    doCreatBattery () {
      let loginData = JSON.parse(sessionStorage.getItem("loginData"));
      this.creatBattery = true;
      this.customerList = loginData.customer;
      this.BatteryForm.enterpriseName = loginData.enterpriseName;
      this.getDeviceListOnly();
      // this.getCustomerList();
    },
    getTime (date) {
      return new Date(date).getTime();
    },
    regBattery (form) {
      console.log('adsasdasdasdasds');
      console.log('form', form);
      if (this.getTime(this.BatteryForm.manufactureDate) < this.getTime(this.BatteryForm.createDate)) {
        this.manufactureDateError = this.$t('batteryList.warn.CheckmanufactureDate')
        return
      }
      if (this.getTime(this.BatteryForm.warrantyDate) < this.getTime(this.BatteryForm.manufactureDate)) {
        this.warrantyDateError = this.$t('batteryList.warn.CheckWarrantyDate')
        return
      }
      this.$refs[form].validate(valid => {
        if (valid) {
          let loginData = JSON.parse(sessionStorage.getItem("loginData"));
          // if (loginData.customer.length > 0) {
          //   this.BatteryForm.customerId = loginData.customer.customerId;
          //   this.BatteryForm.customerName = loginData.customer.customerName;
          // }
          this.BatteryForm.customerId = "";
          this.BatteryForm.customerName = "";
          this.BatteryForm.manufacturerName = loginData.enterpriseName;
          this.BatteryForm.manufacturerId = loginData.enterpriseId;
          console.log("BatteryForm", this.BatteryForm);
          // if (loginData.enterpriseName) {
          //   return
          // }
          addBattery(this.BatteryForm).then(res => {
            console.log(res);
            if (res.data.code === 0) {
              onSuccess(`${this.$t("batteryList.success")}`);
              this.creatBattery = false;
              this.resetBatteryForm(form);
              this.getData();
            }
          });
        }
      });
    },
    resetBatteryForm (form) {
      this.warrantyDateError = "";
      this.manufactureDateError = "";
      this.$refs[form].resetFields();
    },
    getDeviceListOnly () {
      deviceListOnly().then(res => {
        console.log(res);
        if (res.data.code === 0) {
          this.deviceIdList = res.data.data;
        }
      });
    },
    /* 获取电池列表数据 */
    getData () {
      let pageObj = {
        pageSize: this.handleSize,
        pageNum: this.currentPage
      };
      GetList(pageObj).then(res => {
        this.loading = false;
        console.log(res);
        let result = res.data;

        if (result.code === 0) {
          if (result.data.data) {
            let tableObj = result.data.data;
            this.totalPage = result.data.total;
            this.tableData = [];
            tableObj.forEach(key => {
              if (key.bindingStatus === 0) {
                key.bindingName = this.$t("device.nobind");
              } else {
                key.bindingName = this.$t("device.hasbind");
                key.activeBtn = this.$t("device.activeBtn"); // '激活';
                key.alarmBtn = true;
                if (key.activeState === 0) {
                  key.activeText = this.$t("device.active"); // '激活';
                  key.alarmBtn = false;
                }
                if (key.activeState === 1) {
                  key.activeText = this.$t("device.activing"); // '激活中';
                }
                if (key.activeState === 2) {
                  key.activeText = this.$t("device.activeSuccess"); // '激活完成';
                  key.activeBtn = this.$t("device.cancelActive"); // '取消激活';
                  key.alarmBtn = false;
                }
                if (key.activeState === 3) {
                  key.activeText = '待告警'; // '待告警';
                }
                if (key.activeState === 4) {
                  key.activeText = this.$t("device.alarm"); // '告警';
                }
                if (key.activeState === -2) {
                  key.activeText = this.$t("device.CancelActiving"); // '取消激活中';
                  key.alarmBtn = true;
                }
                if (key.activeState === -4) {
                  key.activeText = this.$t("device.alarmCanceling"); // '取消告警中';
                }
              }
              if (key.onlineStatus === 1) {
                key.online = this.$t("batteryList.online");
                key.OLS = true;
              } else {
                key.online = this.$t("batteryList.offline");
                key.OLS = false;
              }
              // key.bindingName =
              //   key.bindingStatus === 0
              //     ? this.$t("batteryList.noBind")
              //     : this.$t("batteryList.hasBind");
              // key.online = key.onlineStatus === 1 ? "离线" : "在线";
              key.status = key.status === 0 ? false : true;
              this.tableData.push(key);
            });
          }
        }
      });
    },
    /*
    * 删除
    */
    deleteRow (index, tableData) {
      // console.log("index", index);
      console.log("data", tableData);
      this.loading = true;
      let params = {
        manufacturer: tableData[index].manufacturerId,
        customer: tableData[index].customerId,
        batteryId: tableData[index].batteryId
      };
      deleteBattery(params).then(res => {
        this.loading = false;
        if (res.data.code === 0) {
          onSuccess(`${this.$t("batteryList.delSuccess")}`);
          this.getData();
        }
      });
    },
    /*
     * 查看运行状态
     */
    examine (index, tableData) {
      let userData = JSON.parse(sessionStorage.getItem("loginData"));
      let deviceId = tableData[index];
      // this.$router.push({
      //   path: "googlePos",
      //   query: { deviceId: deviceId.deviceId }
      // });
      if (userData.mapType === 0) {
        this.$router.push({
          path: "position",
          query: { deviceId: deviceId.deviceId }
        });
      }
      if (userData.mapType === 1) {
        this.$router.push({
          path: "googlePos",
          query: { deviceId: deviceId.deviceId }
        });
      }
    },
    /*
    * 改变每页显示的条数
    */
    handleSizeChange (index) {
      // index为选中的页数
      this.handleSize = index;
      this.getData();
    },
    /*
    * 显示第几页
    */
    handleCurrentChange () {
      this.getData();
    }
  }
};
</script>
<style scoped>
.block {
  text-align: right;
  padding-top: 20px;
}
.el-form {
  padding: 0 20px;
}
.batterylist {
  box-sizing: border-box;
  width: 100%;
  background: #ffffff;
  border-radius: 5px;
  padding: 10px 15px;
}
.flex {
  display: flex;
  justify-content: space-between;
}
.table {
  padding-top: 30px;
}
</style>
