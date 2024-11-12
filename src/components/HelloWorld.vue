<template>
  <div class="hello">
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
    <DrawingBoard :initial-content="htmlContent" />
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";

// import element-ui Dialog
import DrawingBoard from "./DrawingBoard.vue";

@Component({
  components: {
    DrawingBoard,
  },
})
export default class HelloWorld extends Vue {
  @Prop() private msg!: string;

  dialogVisible = false;

  private htmlContent = `
  <div>
    <h1>Go </h1>
    <div>Ruby</div>
    <h5>Python</h5>
    <div>Java</div>
    <h3>JavaScript</h3>
    <div>PHP</div>
    <h2>Swift</h2>
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
