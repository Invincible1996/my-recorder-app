<template>
  <div class="hello">
    <!-- click button show dialog -->
    <el-button type="primary" @click="dialogVisible = true"
      >打开 Dialog</el-button
    >
    <!-- <el-dialog
      title="提示"
      :visible.sync="dialogVisible"
      append-to-body
      :fullscreen="true"
      @close="handleClose"
    >
      <div style="text-align: center; background: #fff; height: 600px">
        <DrawingBoard 
        backgroundHtml="<div>jklfjkladfjklafklakl;fk;fl;a</div>"  />
      </div>
    </el-dialog> -->
    <DrawingBoard :initial-content="htmlContent"/>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";

// import element-ui Dialog
import { Dialog, Button } from "element-ui";
import DrawingBoard from "./DrawingBoard.vue";

@Component({
  components: {
    "el-dialog": Dialog,
    "el-button": Button,
    DrawingBoard,
  },
})
export default class HelloWorld extends Vue {
  @Prop() private msg!: string;

  dialogVisible = false;

  private htmlContent = `
  <div>
    <div>hello world</div>
    <div>hello world</div>
    <div>hello world</div>
    <div>hello world</div>
    <p>hello world</p>
    <img src="https://www.baidu.com/img/flexible/logo/pc/result.png" />
    <div>hello world</div>
    <img src="https://www.jd.com/favicon.ico" />
  </div>
  `

  handleClose() {
    this.dialogVisible = false;
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
.hello {
  margin: 0 auto;
  width: 100%;
}
</style>
