<template>
  <div>
    <el-row type="flex" align="middle" style="margin: 0 0 30px;">
      <el-button
        :disabled="loading"
        v-if="pixiStatus==='pause'"
        icon="iconfont icon-play"
        type="text"
        @click="startPixi"
      ></el-button>
      <el-button
        :disabled="loading"
        v-if="pixiStatus==='play'"
        icon="iconfont icon-stop"
        type="text"
        @click="stopPixi"
      ></el-button>
      <vue-slider
        style="margin-left:20px;width:100%"
        tooltip="none"
        :disabled="loading"
        v-model="ecgIndex"
        :max="recordTime*250"
        @change="handleSlider"
        :processStyle="{backgroundColor: '#11A59C',transition: 'none'}"
        :dotStyle="{borderColor: '#11A59C',display: 'none'}"
        :railStyle="{cursor:'pointer'}"
      ></vue-slider>
    </el-row>
    <div id="pixi-container" v-loading="loading" element-loading-background="rgba(0, 0, 0, 0.6)">
      <div id="timer">{{formatTime}}</div>
    </div>
    <div id="heart" class="heart"></div>
  </div>
</template>

<script>
import fakeData from './ecgdata';
import VueSlider from 'vue-slider-component';
import 'vue-slider-component/theme/antd.css';

export default {
  name: 'ECG',
  components: {
    VueSlider
  },
  props: {
    data: Object
  },
  data() {
    return {
      param: {
        width: 0, //canvas width
        height: 0, //canvas height
        offsetx: 0, //grid position
        offsety: 0, //grid position
        d: 10, //scale of small box
        ref_x: 20, // reference pulse x,
        ref_y: 300, // reference pulse y, height / 2 + 5 * d
        data_x: 0, //ecgData x
        data_y: 700, //ecgData y
        hidden_left: 0,
        hidden_right: 0,
        grid_color: 0x00ff00,
        data_color: 0xffff00,
        timePivot_color: 0xff0000
      },
      descTextStyle: {
        fontFamily: 'Arial',
        fontSize: 16,
        fontStyle: 'italic',
        fontWeight: 'bold',
        fill: ['#ffffff', '#00ff99'], // gradient
        stroke: '#4a1850',
        strokeThickness: 5,
        dropShadow: true,
        dropShadowColor: '#000000',
        dropShadowBlur: 4,
        dropShadowAngle: Math.PI / 6,
        dropShadowDistance: 6,
        wordWrap: true,
        wordWrapWidth: 440
      },
      ecgData: {},
      bpmData: [],
      ecgIndex: 0,
      ecgLine: null,
      labelContainer: null,
      labelMapArr: [
        {
          label: 'N',
          style: {
            fill: ['#63cc4d', '#3cbf42', '#40ff5c'],
            fillGradientStops: [0.1],
            fontFamily: 'Tahoma',
            miterLimit: '',
            stroke: '#319340',
            strokeThickness: 2
          }
        },
        {
          label: 'U',
          style: {
            fill: ['#ffe064', '#fc0'],
            fillGradientStops: [0.1],
            fontFamily: 'Tahoma',
            miterLimit: '',
            stroke: '#d0a700',
            strokeThickness: 2
          }
        },
        {
          label: 'V',
          style: {
            fill: ['white', '#fb0006', '#ff4045'],
            fillGradientStops: [0.1],
            fontFamily: 'Tahoma',
            miterLimit: '',
            stroke: '#d9404b',
            strokeThickness: 2
          }
        },
        {
          label: 'S',
          style: {
            fill: ['white', '#fb0006', '#ff4045'],
            fillGradientStops: [0.1],
            fontFamily: 'Tahoma',
            miterLimit: '',
            stroke: '#d9404b',
            strokeThickness: 2
          }
        }
      ],
      time: 0,
      recordTime: 60,
      pixiApp: null,
      pixiStatus: 'pause',
      loading: true
    };
  },
  mounted() {
    this.labelMapArr[-1] = {
      label: '?',
      style: {
        fill: ['#63cc4d', '#3cbf42', '#40ff5c'],
        fillGradientStops: [0.1],
        fontFamily: 'Tahoma',
        miterLimit: '',
        stroke: '#319340',
        strokeThickness: 2
      }
    };
    this.$nextTick(function() {
      const parent = document.getElementById('pixi-container');
      this.param.width = parent.clientWidth;
      this.param.height = parent.clientHeight;
      this.param.data_x = parseInt(parent.clientWidth / 100) * 50 + 50;
      this.param.data_y = 700; //Todo , 計算適當的位置
      this.param.hidden_left =
        this.param.data_x - this.param.ref_x - 8 * this.param.d;
      this.param.hidden_right = this.param.data_x * 2;
      document.getElementById('timer').style.left =
        this.param.data_x - 4 + 'px';

      this.pixiApp = new PIXI.Application({
        width: this.param.width,
        height: this.param.height,
        antialias: true
      });
      document.getElementById('pixi-container').appendChild(this.pixiApp.view);

      this.drawGrid(this.pixiApp);
      this.loading = true;

      //////fake start
      this.ecgData = fakeData.data;
      this.initPixi();
      //////fake end

      // this.$api
      //   .ecg(this.data.dataId, {
      //     startTime: this.data.startTime,
      //     interval: this.data.runningTime
      //   })
      //   .then(resp => {
      //     this.recordTime = resp.ecg.length / 250;
      //     this.ecgData = resp;
      //     this.initPixi();
      //   })
      //   .catch(err => {
      //     this.$message({
      //       type: 'error',
      //       message: err
      //     });
      //     this.loading = false;
      //   });

      this.$api
        .heartrate(this.data.dataId, {
          startTime: this.data.startTime,
          times: Math.floor(this.data.runningTime / 3) + 1
        })
        .then(resp => {
          this.bpmData = resp;
        })
        .catch(err => {
          this.$message({
            type: 'error',
            message: err
          });
          this.loading = false;
        });
    });
  },
  destroyed() {
    this.pixiApp.destroy();
  },
  methods: {
    handleSlider(val) {
      this.time = val * 0.004;
      this.ecgLine.clear();
      this.drawData(this.pixiApp);
      this.updatePixi();
    },
    // formatTooltip(val) {
    //   return this.formatTime;
    // },
    initPixi() {
      this.ecgLine = new PIXI.Graphics();
      this.labelContainer = new PIXI.Container();
      this.pixiApp.stage.addChild(this.labelContainer);
      this.bpmContainer = new PIXI.Container();
      this.pixiApp.stage.addChild(this.bpmContainer);
      this.drawData(this.pixiApp);
      this.pixiApp.ticker.update();
      const vue = this;
      this.pixiApp.ticker.add(function(delta) {
        vue.time += delta / 60;
        if (vue.time > vue.recordTime) vue.time = vue.recordTime;
        vue.ecgLine.clear();
        vue.labelContainer.removeChildren();
        vue.bpmContainer.removeChildren();
        vue.drawData(vue.pixiApp);
      });
      this.loading = false;
      this.stopPixi();
    },
    startPixi() {
      if (this.time >= this.recordTime) this.time = 0;
      this.pixiApp.ticker.start();
      this.pixiStatus = 'play';
    },
    stopPixi() {
      this.pixiApp.ticker.stop();
      this.pixiStatus = 'pause';
    },
    updatePixi() {
      this.pixiApp.ticker.update();
    },
    drawData(app) {
      const {
        height,
        data_x,
        data_y,
        ref_x,
        d,
        hidden_left,
        hidden_right,
        data_color
      } = this.param;
      const { ecg, peak } = this.ecgData;
      const bpmArr = this.bpmData;
      const line = this.ecgLine;

      line.lineStyle(2, data_color, 1); //ecgLine style (width,color,opacity)

      this.ecgIndex = parseInt(this.time * 250);
      if (this.ecgIndex >= ecg.length) this.stopPixi();

      const start =
        this.ecgIndex - hidden_left > 0 ? this.ecgIndex - hidden_left : 0;
      const end = this.ecgIndex + hidden_right;
      const subArray = ecg.slice(start, end);
      const start_x = data_x - Math.min(hidden_left, this.ecgIndex);

      //渲染ecgLine
      line.moveTo(start_x, data_y - subArray[0] * 10 * d);
      subArray.forEach((item, index) => {
        line.lineTo(start_x + (index * d) / 10, data_y - item * 10 * d);
      });
      app.stage.addChild(line);

      //渲染peak label
      const subPeak = peak.filter(item => {
        return item[0] >= start && item[0] <= end;
      });
      subPeak.forEach(peak => {
        const labelText = new PIXI.Text(
          this.labelMapArr[peak[1]].label,
          this.labelMapArr[peak[1]].style
        );
        labelText.x = start_x - 8 + ((peak[0] - start) * d) / 10;
        labelText.y = d;
        this.labelContainer.addChild(labelText);
      });

      // 渲染bpm
      const bpmText = new PIXI.Text(`BPM`, this.descTextStyle);
      bpmText.x = 46;
      bpmText.y = height - 30;
      this.bpmContainer.addChild(bpmText);
      const bpmText2 = new PIXI.Text(
        this.bpmData[Math.floor(this.time / 3)] === 0
          ? '???'
          : this.bpmData[Math.floor(this.time / 3)],
        this.descTextStyle
      );
      bpmText2.x = 90;
      bpmText2.y = height - 30;
      this.bpmContainer.addChild(bpmText2);

      //心跳動畫速率
      if (this.bpmData[Math.floor(this.time / 3)] === 0) {
        document.getElementById('heart').style.WebkitAnimation = 'none';
        document.getElementById('heart').style.animation = 'none';
      } else {
        const onePeriod = 60 / this.bpmData[Math.floor(this.time / 3)];
        document.getElementById(
          'heart'
        ).style.WebkitAnimation = `heartbeat ${onePeriod}s infinite`;
        document.getElementById(
          'heart'
        ).style.animation = `heartbeat ${onePeriod}s infinite`;
      }
    },
    drawGrid(app) {
      const {
        width,
        height,
        offsetx,
        offsety,
        d,
        ref_x,
        ref_y,
        data_x,
        timePivot_color,
        grid_color,
        data_color
      } = this.param;

      let line = new PIXI.Graphics();

      /*渲染 ecg grid 直線*/
      for (let i = 0; i <= width / d; i++) {
        if (i % 5 === 0) {
          line.lineStyle(1, grid_color, 0.6);
        } else {
          line.lineStyle(1, grid_color, 0.3);
        }
        line.moveTo(i * d + offsetx, offsety);
        line.lineTo(i * d + offsetx, height - offsety);
        app.stage.addChild(line);
      }
      /*渲染 ecg grid 橫線*/
      for (let j = 0; j <= height / d; j++) {
        if (j % 5 === 0) {
          line.lineStyle(1, grid_color, 0.6);
        } else {
          line.lineStyle(1, grid_color, 0.3);
        }
        line.moveTo(offsetx, j * d + offsety);
        line.lineTo(width + offsetx, j * d + offsety);
        app.stage.addChild(line);
      }
      /*渲染XY軸說明*/
      let style = new PIXI.TextStyle(this.descTextStyle);
      let richText = new PIXI.Text('25mm/s', style);
      richText.position.set(width - 70, height - 26);
      app.stage.addChild(richText);
      richText = new PIXI.Text('10mm/mv', style);
      richText.position.set(0, 0);
      app.stage.addChild(richText);
      /*渲染1mv參考波 reference pulse*/
      line.lineStyle(2, data_color, 1);
      line.moveTo(ref_x, ref_y);
      line.lineTo(ref_x + d, ref_y);
      line.lineTo(ref_x + d, ref_y - 10 * d);
      line.lineTo(ref_x + 6 * d, ref_y - 10 * d);
      line.lineTo(ref_x + 6 * d, ref_y);
      line.lineTo(ref_x + 7 * d, ref_y);
      app.stage.addChild(line);
      /*渲染時間參考線*/
      line.lineStyle(2, timePivot_color, 0.6);
      line.moveTo(data_x, 0);
      line.lineTo(data_x, height);
      app.stage.addChild(line);
    }
  },
  computed: {
    formatTime() {
      let min = Math.floor(this.time / 60);
      min = min > 9 ? min : `0${min}`;
      let sec = Math.floor(this.time % 60);
      sec = sec > 9 ? sec : `0${sec}`;
      let csec = this.time.toFixed(2).split('.');
      return `${min}:${sec}:${csec[1]}`;
    }
  }
};
</script>

<style lang="scss" scoped>
#pixi-container {
  position: relative;
  height: 450px;
}
#timer {
  position: absolute;
  top: -40px;
  width: 100px;
  margin-left: -50px;
  text-align: center;
  font-size: 30px;
  font-weight: bold;
  font-family: digital;
}

$heartSize: 18px;
.heart {
  /* Safari 和 Chrome */
  width: $heartSize - 2px;
  height: $heartSize - 2px;
  background: #f00;
  position: relative;
  filter: drop-shadow(0px 0px 2px rgb(255, 20, 20));
  transform: rotate(45deg);
  left: 18px;
  top: -26px;
}
.heart:before,
.heart:after {
  content: '';
  position: absolute;
  width: $heartSize;
  height: $heartSize;
  background: #f00;
  border-radius: 100px;
}
.heart:before {
  left: $heartSize/-2;
}
.heart:after {
  left: 0;
  top: $heartSize/-2;
}
</style>
