<template>
  <div class="whiteboard-container">
    <!-- 画布和内容区域 -->
    <div class="canvas-container" ref="canvasContainer">
      <!-- HTML内容层 -->
      <div
        class="content-layer"
        ref="contentLayer"
        v-html="initialContent"
      ></div>

      <!-- 画布层 -->
      <canvas
        ref="canvas"
        @mousedown="startDrawing"
        @mousemove="draw"
        @mouseup="stopDrawing"
        @mouseleave="stopDrawing"
      ></canvas>
    </div>

    <!-- 右侧工具栏 -->
    <div class="toolbar">
      <div class="tool-section">
        <h3>工具</h3>
        <button
          @click="setTool('pen')"
          :class="{ active: currentTool === 'pen' }"
        >
          画笔
        </button>
        <button
          @click="setTool('eraser')"
          :class="{ active: currentTool === 'eraser' }"
        >
          橡皮擦
        </button>
      </div>

      <div class="tool-section">
        <h3>画笔设置</h3>
        <div class="size-control">
          <label>画笔大小: </label>
          <input type="range" v-model.number="penSize" min="1" max="20" />
          <span class="size-value">{{ penSize }}</span>
        </div>
      </div>

      <div class="tool-section">
        <h3>橡皮擦设置</h3>
        <div class="size-control">
          <label>橡皮擦大小: </label>
          <input type="range" v-model.number="eraserSize" min="5" max="50" />
          <span class="size-value">{{ eraserSize }}</span>
        </div>
      </div>

      <div class="tool-section">
        <h3>录制</h3>
        <div class="recording-controls">
          <!-- 未录制状态 -->
          <button
            v-if="!isRecording"
            @click="startRecording"
            class="record-btn"
          >
            开始录制
          </button>

          <!-- 录制中状态 -->
          <div v-if="isRecording" class="recording-status">
            <span
              class="recording-indicator"
              :class="{ paused: isPaused }"
            ></span>
            <span class="recording-label">{{ recordingStatus }}</span>
            <span class="recording-time">{{ formattedRecordingTime }}</span>
            <div class="recording-buttons">
              <!-- 暂停/继续按钮 -->
              <button
                v-if="!isPaused"
                @click="pauseRecording"
                class="pause-btn"
              >
                暂停
              </button>
              <button v-else @click="resumeRecording" class="resume-btn">
                继续
              </button>
              <!-- 停止按钮 -->
              <button @click="stopRecording" class="stop-btn">结束录制</button>
            </div>
          </div>
        </div>
        <div class="tool-section">
          <h3>操作</h3>
          <button @click="undo" :disabled="!canUndo">撤销</button>
          <button @click="redo" :disabled="!canRedo">重做</button>
          <button @click="clear">清空</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from "vue-property-decorator";
import html2canvas from "html2canvas";
interface DrawStep {
  type: "pen" | "eraser";
  points: { x: number; y: number }[];
  size: number;
}

@Component
export default class WhiteBoard extends Vue {
  @Prop({ default: "" }) readonly initialContent!: string;

  private canvas: HTMLCanvasElement | null = null;
  private ctx: any = null;
  private isDrawing = false;
  private currentTool: "pen" | "eraser" = "pen";
  private penSize = 2;
  private eraserSize = 20;
  private drawHistory: DrawStep[] = [];
  private redoStack: DrawStep[] = [];
  private currentStep: DrawStep | null = null;
  private mediaRecorder: MediaRecorder | null = null;
  private recordedChunks: any[] = [];
  private isRecording = false;
  private isMouseOnCanvas = false;
  private cursorPosition = { x: 0, y: 0 };
  private resizeObserver: ResizeObserver | null = null;

  private recordingStartTime = 0;
  private recordingTimer: number | null = null;
  private recordingTime = 0;

  private recordingCanvas: HTMLCanvasElement | null = null;
  private recordingCtx: CanvasRenderingContext2D | null = null;
  private backgroundCanvas: HTMLCanvasElement | null = null;

  private isPaused = false;
  private pausedTime = 0; // 记录暂停时的时间

  get formattedRecordingTime(): string {
    const totalSeconds = this.recordingTime;
    const minutes = Math.floor(totalSeconds / 60);
    const seconds = totalSeconds % 60;
    return `${minutes.toString().padStart(2, "0")}:${seconds
      .toString()
      .padStart(2, "0")}`;
  }

  get recordingStatus(): string {
    if (this.isPaused) return "已暂停";
    if (this.isRecording) return "录制中";
    return "准备录制";
  }

  async mounted() {
    this.initCanvas();
    window.addEventListener("mousemove", this.updateCursorPosition);

    // 初始化背景画布
    await this.initBackgroundCanvas();

    // 监听容器大小变化
    this.resizeObserver = new ResizeObserver(() => {
      this.updateCanvasSize();
    });

    const container = this.$refs.canvasContainer as HTMLElement;
    if (container) {
      this.resizeObserver.observe(container);
    }
  }

  private async initBackgroundCanvas() {
    const contentLayer = this.$refs.contentLayer as HTMLElement;
    if (!contentLayer) return;

    try {
      // 使用 html2canvas 将 HTML 内容转换为 canvas
      const htmlCanvas = await html2canvas(contentLayer, {
        backgroundColor: null,
        scale: window.devicePixelRatio || 1,
      });

      this.backgroundCanvas = htmlCanvas;
      this.redrawCanvas(); // 重绘以显示背景
    } catch (error) {
      console.error("Failed to convert HTML to canvas:", error);
    }
  }

  beforeDestroy() {
    window.removeEventListener("mousemove", this.updateCursorPosition);
    if (this.resizeObserver) {
      this.resizeObserver.disconnect();
    }
    if (this.recordingTimer) {
      clearInterval(this.recordingTimer);
    }
    if (this.mediaRecorder && this.mediaRecorder.state !== "inactive") {
      this.mediaRecorder.stop();
    }
  }

  private initCanvas() {
    this.canvas = this.$refs.canvas as HTMLCanvasElement;
    this.ctx = this.canvas.getContext("2d");

    if (this.ctx) {
      this.updateCanvasSize();

      // 设置默认样式
      this.ctx.lineCap = "round";
      this.ctx.lineJoin = "round";
    }
  }

  private drawCurrentStep() {
    if (!this.ctx || !this.currentStep || this.currentStep.points.length < 2)
      return;

    const points = this.currentStep.points;
    const lastTwoPoints = points.slice(-2);

    this.ctx.beginPath();
    this.ctx.moveTo(lastTwoPoints[0].x, lastTwoPoints[0].y);
    this.ctx.lineTo(lastTwoPoints[1].x, lastTwoPoints[1].y);

    if (this.currentStep.type === "pen") {
      this.ctx.strokeStyle = "#000";
      this.ctx.lineWidth = this.currentStep.size;
      this.ctx.lineCap = "round";
      this.ctx.lineJoin = "round";
      this.ctx.stroke();
    } else {
      this.ctx.strokeStyle = "#fff";
      this.ctx.lineWidth = this.currentStep.size;
      this.ctx.lineCap = "round";
      this.ctx.lineJoin = "round";
      this.ctx.globalCompositeOperation = "destination-out";
      this.ctx.stroke();
      this.ctx.globalCompositeOperation = "source-over";
    }
  }

  private async updateCanvasSize() {
    if (!this.canvas) return;

    const container = this.$refs.canvasContainer as HTMLElement;
    const { width, height } = container.getBoundingClientRect();

    // 设置画布尺寸为容器的实际尺寸
    this.canvas.width = width;
    this.canvas.height = height;

    // 重新初始化背景画布
    await this.initBackgroundCanvas();

    // 重新绘制历史记录
    this.redrawCanvas();
  }

  private updateCursorPosition(e: MouseEvent) {
    const rect = this.canvas?.getBoundingClientRect();
    if (rect) {
      this.cursorPosition = {
        x: e.clientX - rect.left,
        y: e.clientY - rect.top,
      };
      this.isMouseOnCanvas = e.target === this.canvas;

      // 重绘画布以更新指示器位置
      this.redrawCanvas();

      // 如果鼠标在画布上且当前工具是橡皮擦，绘制指示器
      if (this.isMouseOnCanvas && this.currentTool === "eraser" && this.ctx) {
        this.drawEraserIndicator(
          this.ctx,
          this.cursorPosition.x,
          this.cursorPosition.y,
          this.eraserSize
        );
      }
    }
  }

  private startDrawing(e: MouseEvent) {
    this.isDrawing = true;
    const point = this.getCanvasPoint(e);

    this.currentStep = {
      type: this.currentTool,
      points: [point],
      size: this.currentTool === "pen" ? this.penSize : this.eraserSize,
    };

    // 清除重做栈
    this.redoStack = [];
  }

  private stopDrawing() {
    if (this.isDrawing && this.currentStep) {
      this.drawHistory.push(this.currentStep);
    }
    this.isDrawing = false;
    this.currentStep = null;

    // 重绘以确保最终状态正确
    this.redrawCanvas();
  }

  private drawEraserIndicator(
    ctx: CanvasRenderingContext2D,
    x: number,
    y: number,
    size: number
  ) {
    ctx.save();

    // 设置合成模式，确保指示器始终可见
    ctx.globalCompositeOperation = "source-over";

    // 绘制白色填充
    ctx.beginPath();
    ctx.arc(x, y, size / 2, 0, Math.PI * 2);
    ctx.fillStyle = "rgba(255, 255, 255, 0.5)"; // 半透明白色填充
    ctx.fill();

    // 绘制边框
    ctx.beginPath();
    ctx.arc(x, y, size / 2, 0, Math.PI * 2);
    ctx.strokeStyle = "#666"; // 灰色边框
    ctx.lineWidth = 2; // 边框宽度
    ctx.stroke();

    // 绘制内圈（可选，增加视觉效果）
    ctx.beginPath();
    ctx.arc(x, y, size / 2 - 4, 0, Math.PI * 2);
    ctx.strokeStyle = "#999"; // 浅灰色内圈
    ctx.lineWidth = 1; // 内圈宽度
    ctx.stroke();

    ctx.restore();
  }

  private draw(e: MouseEvent) {
    if (!this.isDrawing || !this.ctx || !this.currentStep) return;

    const point = this.getCanvasPoint(e);
    this.currentStep.points.push(point);
    this.cursorPosition = point;

    // 只绘制当前线段，不重绘整个画布
    if (this.currentStep.points.length >= 2) {
      this.drawCurrentStep();
    }

    // 如果是橡皮擦工具，绘制指示器
    if (this.currentTool === "eraser") {
      // 保存当前的canvas状态
      this.ctx.save();

      // 清除指示器区域
      this.ctx.globalCompositeOperation = "source-over";
      const indicatorSize = this.eraserSize + 4; // 比橡皮擦尺寸稍大一些
      this.ctx.clearRect(
        point.x - indicatorSize / 2,
        point.y - indicatorSize / 2,
        indicatorSize,
        indicatorSize
      );

      // 绘制指示器
      this.drawEraserIndicator(this.ctx, point.x, point.y, this.eraserSize);

      // 恢复canvas状态
      this.ctx.restore();
    }

    // 如果正在录制，同步更新录制画布
    if (this.isRecording && this.recordingCanvas && this.recordingCtx) {
      this.recordingCtx.clearRect(
        0,
        0,
        this.recordingCanvas.width,
        this.recordingCanvas.height
      );
      this.recordingCtx.drawImage(this.canvas!, 0, 0);
    }
  }

  private getCanvasPoint(e: MouseEvent) {
    const rect = this.canvas!.getBoundingClientRect();
    return {
      x: e.clientX - rect.left,
      y: e.clientY - rect.top,
    };
  }

  private drawStep(step: DrawStep) {
    if (!this.ctx || step.points.length < 2) return;

    this.ctx.beginPath();
    this.ctx.moveTo(step.points[0].x, step.points[0].y);

    for (let i = 1; i < step.points.length; i++) {
      this.ctx.lineTo(step.points[i].x, step.points[i].y);
    }

    if (step.type === "pen") {
      this.ctx.strokeStyle = "#000";
      this.ctx.lineWidth = step.size;
      this.ctx.stroke();
    } else {
      this.ctx.strokeStyle = "#fff";
      this.ctx.lineWidth = step.size;
      this.ctx.globalCompositeOperation = "destination-out";
      this.ctx.stroke();
      this.ctx.globalCompositeOperation = "source-over";
    }
  }

  private redrawCanvas() {
    if (!this.ctx || !this.canvas) return;

    // 清空画布
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 先绘制背景（如果有）
    if (this.backgroundCanvas) {
      this.ctx.drawImage(
        this.backgroundCanvas,
        0,
        0,
        this.canvas.width,
        this.canvas.height
      );
    }

    // 重绘历史记录
    this.drawHistory.forEach((step) => {
      if (step.points.length < 2) return;

      this.ctx.beginPath();
      this.ctx.moveTo(step.points[0].x, step.points[0].y);

      for (let i = 1; i < step.points.length; i++) {
        this.ctx.lineTo(step.points[i].x, step.points[i].y);
      }

      if (step.type === "pen") {
        this.ctx.strokeStyle = "#000";
        this.ctx.lineWidth = step.size;
        this.ctx.lineCap = "round";
        this.ctx.lineJoin = "round";
        this.ctx.stroke();
      } else {
        this.ctx.strokeStyle = "#fff";
        this.ctx.lineWidth = step.size;
        this.ctx.lineCap = "round";
        this.ctx.lineJoin = "round";
        this.ctx.globalCompositeOperation = "destination-out";
        this.ctx.stroke();
        this.ctx.globalCompositeOperation = "source-over";
      }
    });

    // 如果正在绘制，也绘制当前步骤
    if (this.currentStep && this.currentStep.points.length >= 2) {
      this.ctx.beginPath();
      this.ctx.moveTo(
        this.currentStep.points[0].x,
        this.currentStep.points[0].y
      );

      for (let i = 1; i < this.currentStep.points.length; i++) {
        this.ctx.lineTo(
          this.currentStep.points[i].x,
          this.currentStep.points[i].y
        );
      }

      if (this.currentStep.type === "pen") {
        this.ctx.strokeStyle = "#000";
        this.ctx.lineWidth = this.currentStep.size;
        this.ctx.lineCap = "round";
        this.ctx.lineJoin = "round";
        this.ctx.stroke();
      } else {
        this.ctx.strokeStyle = "#fff";
        this.ctx.lineWidth = this.currentStep.size;
        this.ctx.lineCap = "round";
        this.ctx.lineJoin = "round";
        this.ctx.globalCompositeOperation = "destination-out";
        this.ctx.stroke();
        this.ctx.globalCompositeOperation = "source-over";
      }
    }

    // 如果当前工具是橡皮擦且鼠标在画布上，绘制指示器
    if (this.currentTool === "eraser" && this.isMouseOnCanvas) {
      this.drawEraserIndicator(
        this.ctx,
        this.cursorPosition.x,
        this.cursorPosition.y,
        this.eraserSize
      );
    }

    // 如果正在录制，同步更新录制画布
    if (this.isRecording && this.recordingCanvas && this.recordingCtx) {
      this.recordingCtx.clearRect(
        0,
        0,
        this.recordingCanvas.width,
        this.recordingCanvas.height
      );
      this.recordingCtx.fillStyle = "#fff";
      this.recordingCtx.fillRect(
        0,
        0,
        this.recordingCanvas.width,
        this.recordingCanvas.height
      );
      // 确保背景内容被复制到录制画布
      if (this.backgroundCanvas) {
        this.recordingCtx.drawImage(
          this.backgroundCanvas,
          0,
          0,
          this.recordingCanvas.width,
          this.recordingCanvas.height
        );
      }
      this.recordingCtx.drawImage(this.canvas, 0, 0);
    }
  }

  // 操作方法
  private setTool(tool: "pen" | "eraser") {
    this.currentTool = tool;
  }

  private undo() {
    if (this.drawHistory.length > 0) {
      const step = this.drawHistory.pop()!;
      this.redoStack.push(step);
      this.redrawCanvas();
    }
  }

  private redo() {
    if (this.redoStack.length > 0) {
      const step = this.redoStack.pop()!;
      this.drawHistory.push(step);
      this.redrawCanvas();
    }
  }

  private clear() {
    this.drawHistory = [];
    this.redoStack = [];
    this.redrawCanvas();
  }

  // 计算属性
  get canUndo(): boolean {
    return this.drawHistory.length > 0;
  }

  get canRedo(): boolean {
    return this.redoStack.length > 0;
  }

  startRecording() {
    if (!this.canvas) return;

    try {
      // 创建录制用的canvas
      this.recordingCanvas = document.createElement("canvas");
      this.recordingCanvas.width = this.canvas.width;
      this.recordingCanvas.height = this.canvas.height;
      this.recordingCtx = this.recordingCanvas.getContext("2d");

      if (!this.recordingCtx) {
        throw new Error("无法创建录制上下文");
      }

      // 创建 MediaRecorder
      const stream = this.recordingCanvas.captureStream(30);
      this.mediaRecorder = new MediaRecorder(stream, {
        mimeType: "video/webm;codecs=vp8",
      });

      this.recordedChunks = [];
      this.recordingTime = 0;
      this.recordingStartTime = Date.now();
      this.pausedTime = 0;
      this.isPaused = false;

      // 处理录制数据
      this.mediaRecorder.ondataavailable = (e) => {
        if (e.data && e.data.size > 0) {
          this.recordedChunks.push(e.data);
        }
      };

      // 处理录制停止
      this.mediaRecorder.onstop = () => {
        this.finishRecording();
      };

      // 开始录制
      this.mediaRecorder.start(1000);
      this.isRecording = true;

      // 更新录制时间
      this.startRecordingTimer();
    } catch (err) {
      console.error("录制失败:", err);
      alert("无法开始录制，请确保浏览器支持画布录制功能。");
    }
  }

  pauseRecording() {
    if (this.mediaRecorder && this.mediaRecorder.state === "recording") {
      this.mediaRecorder.pause();
      this.isPaused = true;
      this.pausedTime = Date.now();
      if (this.recordingTimer) {
        clearInterval(this.recordingTimer);
        this.recordingTimer = null;
      }
    }
  }

  resumeRecording() {
    if (this.mediaRecorder && this.mediaRecorder.state === "paused") {
      this.mediaRecorder.resume();
      this.isPaused = false;
      // 调整开始时间，考虑暂停的时间
      if (this.pausedTime) {
        const pauseDuration = Date.now() - this.pausedTime;
        this.recordingStartTime += pauseDuration;
      }
      this.startRecordingTimer();
    }
  }

  stopRecording() {
    if (this.mediaRecorder && this.mediaRecorder.state !== "inactive") {
      this.mediaRecorder.stop();
    }
  }

  private startRecordingTimer() {
    if (this.recordingTimer) {
      clearInterval(this.recordingTimer);
    }

    this.recordingTimer = window.setInterval(() => {
      this.recordingTime = Math.floor(
        (Date.now() - this.recordingStartTime) / 1000
      );
    }, 1000);
  }

  private async finishRecording() {
    try {
      const blob = new Blob(this.recordedChunks, {
        type: "video/webm",
      });

      // 创建下载链接
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      document.body.appendChild(a);
      a.style.display = "none";
      a.href = url;
      a.download = `whiteboard-recording-${new Date().toISOString()}.webm`;
      a.click();

      // 清理资源
      window.URL.revokeObjectURL(url);
      document.body.removeChild(a);

      // 重置状态
      this.isRecording = false;
      this.isPaused = false;
      this.recordingTime = 0;
      if (this.recordingTimer) {
        clearInterval(this.recordingTimer);
        this.recordingTimer = null;
      }
      this.recordedChunks = [];
      this.mediaRecorder = null;
      this.recordingCanvas = null;
      this.recordingCtx = null;
    } catch (error) {
      console.error("保存录制文件失败:", error);
      alert("保存录制文件时出现错误。");
    }
  }

  private downloadRecording() {
    try {
      const blob = new Blob(this.recordedChunks, {
        type: "video/webm",
      });

      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      document.body.appendChild(a);
      a.style.display = "none";
      a.href = url;
      a.download = `whiteboard-recording-${new Date().toISOString()}.webm`;
      a.click();

      // 清理资源
      window.URL.revokeObjectURL(url);
      document.body.removeChild(a);
    } catch (error) {
      console.error("下载录制文件失败:", error);
      alert("下载录制文件时出现错误。");
    }
  }
}
</script>

<style scoped>
.whiteboard-container.recording .canvas-container::after {
  content: "录制中";
  position: absolute;
  top: 10px;
  right: 10px;
  background: rgba(255, 0, 0, 0.8);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  pointer-events: none;
}

.whiteboard-container.recording.paused .canvas-container::after {
  content: "已暂停";
  background: rgba(102, 102, 102, 0.8);
}

.canvas-container {
  flex: 1;
  position: relative;
  background: #fff;
  min-width: 0;
  border: 2px solid #e0e0e0;
  margin: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  overflow: hidden; /* 确保内容不会溢出容器 */
}

/* 更新橡皮擦指示器样式 */
.eraser-indicator {
  position: absolute; /* 改为absolute定位 */
  border: 2px solid #666;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  pointer-events: none;
  transform: translate(-50%, -50%);
  z-index: 1000; /* 确保在最上层 */
  mix-blend-mode: difference; /* 使指示器在不同背景色上都清晰可见 */
}

.toolbar {
  width: 250px;
  height: 100%;
  padding: 20px;
  background: #f5f5f5;
  display: flex;
  flex-direction: column;
  gap: 20px;
  overflow-y: auto;
  flex-shrink: 0;
  border-left: 1px solid #ddd;
  box-shadow: -2px 0 8px rgba(0, 0, 0, 0.05);
}

.whiteboard-container {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  overflow: hidden;
  background: #f9f9f9; /* 添加容器背景色 */
}

canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  cursor: crosshair;
  touch-action: none;
  border-radius: 8px; /* 与容器圆角保持一致 */
  z-index: 1;
}

/* 更新工具栏按钮样式，增加视觉层次 */
button {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  background: white;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 14px;
  color: #333;
}

button:hover {
  background: #f0f0f0;
  border-color: #ccc;
  transform: translateY(-1px);
}

button.active {
  background: #4a90e2;
  color: white;
  border-color: #357abd;
  box-shadow: 0 2px 4px rgba(74, 144, 226, 0.2);
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

/* 工具栏分组样式优化 */
.tool-section {
  background: white;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.tool-section h3 {
  margin: 0;
  font-size: 15px;
  color: #333;
  border-bottom: 1px solid #eee;
  padding-bottom: 8px;
  margin-bottom: 12px;
}

.content-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  overflow: hidden;
  z-index: 0; /* 确保在最底层 */
}
</style>
