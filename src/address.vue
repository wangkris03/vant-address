<template>
  <div class="ad-container">
    <div v-if="showNavBar" class="modal-head">
      <van-nav-bar :style="navBarStyle" :title="title" />
    </div>
    <div v-show="showSteps" class="modal-des">
      <van-steps
        direction="vertical"
        :active-color="activeColor"
        :active="active"
        @click-step="onClickStep"
      >
        <van-step :style="stepStyle" v-for="(item, i) in stepData" :key="i">{{
          item[itemKey]
        }}</van-step>
      </van-steps>
    </div>
    <div class="modal-body">
      <van-grid>
        <van-grid-item
          v-for="(item, i) in currentList"
          :key="i"
          :text="item[itemKey]"
          @click="onSelect(item)"
        >
          <slot>
            <span class="van-grid-item__text" :style="gridItemStyle">
              {{ item[itemKey] }}
            </span>
          </slot>
        </van-grid-item>
      </van-grid>
    </div>
  </div>
</template>

<script>
export default {
  name: "vant-address",

  props: {
    debug: { type: Boolean, default: false },
    title: { type: String, default: "请选择所在地区" },
    showNavBar: { type: Boolean, default: true },
    //层级
    level: {
      type: Number,
      default: 4,
      validator: function (value) {
        // 这个值必须匹配下列字符串中的一个
        return value > 0 && value < 5;
      },
    },
    //已选择的数据对象
    data: { type: Array, default: () => [] },
    //初始化请求参数
    params: { type: Object, require: true },
    //显示key
    itemKey: { type: String, default: "title" },
    activeColor: { type: String, default: "rgb(135, 202, 204)" },
    //nav bar style
    navBarStyle: { type: Object, default: () => {} },
    //step style
    stepStyle: { type: Object, default: () => {} },
    //grid item style
    gridItemStyle: { type: Object, default: () => {} },
    //请求获取地址
    action: { type: Function, require: true },
  },

  data() {
    return {
      showSteps: false,
      active: 0,
      stepData: [],
      option: 1, //1新增；2修改
      currentList: [],
      provinceList: [],
      cityList: [],
      districtList: [],
      streetList: [],
    };
  },
  mounted() {},
  watch: {},
  methods: {
    debugInfo(key, ...value) {
      if (this.debug) {
        console.log(key, ...value);
      }
    },
    async initAddress() {
      // 去除空的省市区对象
      this.dataCheck = this.data.filter((item) => item.value);
      //无首选初始化
      if (this.dataCheck.length === 0) {
        this.stepData = [];
        await this.setList(0, this.params);
        this.currentList = this.provinceList;
        return;
      }
      //有首选初始化
      if (this.data && this.data.length > 0) {
        this.showSteps = true;
        this.stepData = this.data;
        if (this.data.length === this.level) {
          this.option = 2;
          this.active = this.level - 1;
          for (let i in this.stepData) {
            if (this.stepData[i].value)
              await this.setList(+i + 1, { pid: this.stepData[i].value });
          }
          this.onClickStep(this.level - 1);
        } else {
          this.option = 1;
          let stepLength = this.stepData.length;
          if (this.stepData[stepLength - 1]["flag"]) {
            this.stepData.length = stepLength - 1;
            stepLength = this.stepData.length;
          }

          //init 省市
          for (let i in this.stepData) {
            if (this.stepData[i].value)
              await this.setList(+i + 1, { pid: this.stepData[i].value });
          }

          let tipText =
            stepLength === 1
              ? "请选择城市"
              : stepLength === 2
              ? "请选择区县"
              : stepLength === 3
              ? "请选择街道"
              : "";
          if (tipText) {
            let tipItem = {
              flag: "tip",
              title: tipText,
            };

            this.stepData.push(tipItem);
            this.active = stepLength;
            this.onClickStep(stepLength);
          }
          // }
        }
      }
    },
    async onSelect(item) {
      this.showSteps = true;
      //过滤多余的flag=tip
      this.stepData = this.stepData.filter((i) => !i.flag);

      //根据this.option判断当前选择是新增还是修改
      if (this.option === 1) {
        this.stepData.push(item);
        this.debugInfo("新增", this.stepData, this.active, item);
      }
      if (this.option === 2) {
        this.stepData[this.active] = item;
        this.stepData.length = this.active + 1;
        this.debugInfo("修改", this.stepData, this.active, item);
      }

      this.$emit("confirm", this.stepData);
      if (this.stepData.length >= this.level) {
        this.debugInfo("选完结束");
        this.option = 2;
      } else {
        //已选对象省市区街道
        this.currentList = []; //防止重选
        this.option = 1;

        let stepLength = this.stepData.length;
        let tipText =
          stepLength === 1
            ? "请选择城市"
            : stepLength === 2
            ? "请选择区县"
            : stepLength === 3
            ? "请选择街道"
            : "";
        if (tipText) {
          let tipItem = {
            flag: "tip",
            title: tipText,
          };
          this.active++;
          this.stepData.push(tipItem);
          //处理下一级 接口
          await this.setList(stepLength, { pid: item.value });
          this.onClickStep(stepLength);
        }
      }
    },
    onClickStep(index) {
      if (this.stepData[index]["flag"]) this.option = 1;
      else this.option = 2;
      this.active = index;
      if (index === 0) this.currentList = this.provinceList;
      else if (index === 1) this.currentList = this.cityList;
      else if (index === 2) this.currentList = this.districtList;
      else if (index === 3) this.currentList = this.streetList;
    },

    async setList(index, param) {
      try {
        const data = await this.action(param);
        this.debugInfo("setList", data);
        if (data) {
          if (index === 0) {
            this.provinceList = data;
          }
          if (index === 1) {
            this.cityList = data;
          }
          if (index === 2) {
            this.districtList = data;
          }
          if (index === 3) {
            this.streetList = data;
          }
        }
      } catch (error) {
        this.debugInfo("error", error);
        this.currentList = [];
      }
    },
  },
};
</script>

<style scoped>
.ad-container {
  height: 70vh;
  display: flex;
  flex-direction: column;
}
.modal-head {
  flex: 1;
}
.modal-des {
  flex: 1;
  text-align: left;
}
.modal-body {
  flex: 100;
  overflow: auto;
}
.van-step {
  font-size: 16px;
}
.van-step--vertical:not(:last-child)::after {
  border-bottom-width: 0px;
}
</style>
