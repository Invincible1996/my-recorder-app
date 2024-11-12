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
        <h3>画笔颜色</h3>
        <div class="color-picker">
          <!-- 当前颜色显示和自定义颜色选择器 -->
          <label class="color-picker-label">
            <div
              class="current-color"
              :style="{ backgroundColor: currentColor }"
            ></div>
            <input
              type="color"
              v-model="currentColor"
              class="custom-color-input"
              @change="setColor(currentColor)"
            />
          </label>

          <!-- 颜色预设 -->
          <div class="color-presets">
            <div
              v-for="(color, index) in colorPresets"
              :key="index"
              class="color-preset"
              :style="{ backgroundColor: color }"
              :class="{ active: currentColor === color }"
              @click="setColor(color)"
            ></div>
          </div>
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
  color?: string; // 添加颜色属性
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

  // [新增] 添加绘制画布，专门用于存储用户绘制的内容
  private drawingCanvas: any;
  private drawingCtx: any;

  // isMouseDown
  private isMouseDown = false;

  // 添加音频相关属性
  private audioStream: MediaStream | null = null;
  private combinedStream: MediaStream | null = null;

  // 添加画笔颜色相关属性
  private currentColor = "#000000";
  private colorPresets = [
    "#000000", // 黑色
    "#FF0000", // 红色
    "#00FF00", // 绿色
    "#0000FF", // 蓝色
    "#FF9900", // 橙色
    "#FF00FF", // 粉色
    "#00FFFF", // 青色
    "#9933FF", // 紫色
  ];

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
    // 确保初始内容层可见
    const contentLayer = this.$refs.contentLayer as HTMLElement;
    if (contentLayer) {
      contentLayer.style.visibility = "visible";
      contentLayer.style.opacity = "1";
    }

    // 等待一帧以确保内容渲染
    await new Promise((resolve) => requestAnimationFrame(resolve));

    // 初始化画布
    await this.initCanvas();

    // 添加鼠标事件监听
    const canvasContainer = this.$refs.canvasContainer as HTMLElement;
    if (canvasContainer) {
      canvasContainer.addEventListener("mousemove", this.updateCursorPosition);
      canvasContainer.addEventListener("mouseenter", this.handleMouseEnter);
      canvasContainer.addEventListener("mouseleave", this.handleMouseLeave);
    }

    // 监听容器大小变化
    this.resizeObserver = new ResizeObserver(async () => {
      await this.updateCanvasSize();
    });

    if (canvasContainer) {
      this.resizeObserver.observe(canvasContainer);
    }

    // 添加平滑过渡样式
    this.updateStyle();
  }

  // 2. 添加新的鼠标事件处理方法
  private handleMouseEnter(e: MouseEvent) {
    this.isMouseOnCanvas = true;
    this.updateCursorPosition(e);
    this.redrawCanvas();
  }

  private handleMouseLeave(e: MouseEvent) {
    this.isMouseOnCanvas = false;
    this.redrawCanvas();
  }

  // [修改] 背景初始化方法
  private async initBackgroundCanvas() {
    const contentLayer = this.$refs.contentLayer as HTMLElement;
    if (!contentLayer) return;

    console.log("HTML content rendered:", contentLayer.innerHTML);
    console.log("Background canvas created:", this.backgroundCanvas);

    // 确保内容层可见并且有内容
    contentLayer.style.visibility = "visible";
    contentLayer.style.opacity = "1";

    // 等待一帧以确保内容完全渲染
    await new Promise((resolve) => requestAnimationFrame(resolve));

    try {
      // 使用 html2canvas 将 HTML 内容转换为 canvas
      const htmlCanvas = await html2canvas(contentLayer, {
        backgroundColor: null,
        scale: window.devicePixelRatio || 1,
        useCORS: false,
        allowTaint: true,
        imageTimeout: 0,
        logging: false,
        onclone: (documentClone) => {
          // 在克隆的文档中处理图片
          const images = documentClone.getElementsByTagName("img");
          for (let img of images) {
            img.crossOrigin = "anonymous";
            // 如果图片还没加载完成，等待加载
            if (!img.complete) {
              return new Promise((resolve) => {
                img.onload = resolve;
              });
            }
          }
        },
      });

      this.backgroundCanvas = htmlCanvas;

      console.log(
        "Background canvas created1111111111:",
        this.backgroundCanvas
      );

      // 将背景绘制到主画布
      this.redrawCanvas();

      // 使用淡出效果隐藏原始内容层
      contentLayer.style.transition = "opacity 0.3s ease";
      contentLayer.style.opacity = "0";
      setTimeout(() => {
        contentLayer.style.visibility = "hidden";
      }, 300);
    } catch (error) {
      console.error("Failed to convert HTML to canvas:", error);
      // 发生错误时保持内容层可见
      contentLayer.style.visibility = "visible";
      contentLayer.style.opacity = "1";
    }
  }

  beforeDestroy() {
    const canvasContainer = this.$refs.canvasContainer as HTMLElement;
    if (canvasContainer) {
      canvasContainer.removeEventListener(
        "mousemove",
        this.updateCursorPosition
      );
      canvasContainer.removeEventListener("mouseenter", this.handleMouseEnter);
      canvasContainer.removeEventListener("mouseleave", this.handleMouseLeave);
    }

    if (this.resizeObserver) {
      this.resizeObserver.disconnect();
    }
    if (this.recordingTimer) {
      clearInterval(this.recordingTimer);
    }
    if (this.mediaRecorder && this.mediaRecorder.state !== "inactive") {
      this.mediaRecorder.stop();
    }
    this.cleanupStreams();
  }

  private async initCanvas() {
    // 初始化主画布（用于显示）
    this.canvas = this.$refs.canvas as HTMLCanvasElement;
    this.ctx = this.canvas.getContext("2d", { alpha: false });

    // [新增] 初始化绘制画布
    this.drawingCanvas = document.createElement("canvas");
    this.drawingCtx = this.drawingCanvas.getContext("2d", { alpha: true });

    const container = this.$refs.canvasContainer as HTMLElement;
    const { width, height } = container.getBoundingClientRect();

    // 设置画布大小
    this.canvas.width = width;
    this.canvas.height = height;
    this.drawingCanvas.width = width;
    this.drawingCanvas.height = height;

    // 设置白色背景
    if (this.ctx) {
      this.ctx.fillStyle = "#fff";
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    }

    // 设置默认样式
    if (this.drawingCtx) {
      // 设置白色背景
      this.drawingCtx.fillRect(
        0,
        0,
        this.drawingCanvas.width,
        this.drawingCanvas.height
      );

      this.drawingCtx.lineCap = "round";
      this.drawingCtx.lineJoin = "round";
    }

    // 初始化背景
    await this.initBackgroundCanvas();
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
    if (!this.canvas || !this.drawingCanvas) return;

    const container = this.$refs.canvasContainer as HTMLElement;
    const { width, height } = container.getBoundingClientRect();

    // 更新所有画布的尺寸
    this.canvas.width = width;
    this.canvas.height = height;
    this.drawingCanvas.width = width;
    this.drawingCanvas.height = height;

    if (this.ctx) {
      this.ctx.fillStyle = "#fff";
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    }

    // 重新设置样式（因为改变画布大小会重置上下文）
    if (this.drawingCtx) {
      this.drawingCtx.lineCap = "round";
      this.drawingCtx.lineJoin = "round";
    }

    // 重新初始化背景画布
    await this.initBackgroundCanvas();

    // 重新绘制历史记录
    this.redrawDrawingCanvas();
  }

  // 修改 updateCursorPosition 方法
  private updateCursorPosition(e: MouseEvent) {
    const canvasContainer = this.$refs.canvasContainer as HTMLElement;
    if (canvasContainer) {
      const rect = canvasContainer.getBoundingClientRect();
      this.cursorPosition = {
        x: e.clientX - rect.left,
        y: e.clientY - rect.top,
      };

      // 触发重绘，这样会同时更新主画布和录制画布
      if (this.isDrawing) {
        this.redrawCanvas();
      }
    }
  }

  private startDrawing(e: MouseEvent) {
    this.isDrawing = true;
    this.isMouseDown = true;
    const point = this.getCanvasPoint(e);

    this.currentStep = {
      type: this.currentTool,
      points: [point],
      size: this.currentTool === "pen" ? this.penSize : this.eraserSize,
      color: this.currentTool === "pen" ? this.currentColor : undefined,
    };

    // 清除重做栈
    this.redoStack = [];
  }

  private stopDrawing() {
    if (this.isDrawing && this.currentStep) {
      this.drawHistory.push(this.currentStep);
    }
    this.isDrawing = false;
    this.isMouseDown = false;
    this.currentStep = null;

    // 重绘以确保最终状态正确
    this.redrawCanvas();
  }

  // 修改 drawEraserIndicator 方法
  // 修改绘制橡皮擦指示器的方法
  private drawEraserIndicator(
    ctx: CanvasRenderingContext2D,
    x: number,
    y: number,
    size: number
  ) {
    console.log("drawEraserIndicator");

    ctx.save();

    // 绘制外圈（半透明填充）
    ctx.beginPath();
    ctx.arc(x, y, size / 2, 0, Math.PI * 2);
    ctx.fillStyle = "rgba(255, 255, 255, 0.3)";
    ctx.fill();

    // 绘制边框
    ctx.beginPath();
    ctx.arc(x, y, size / 2, 0, Math.PI * 2);
    ctx.strokeStyle = "#666";
    ctx.lineWidth = 1.5;
    ctx.stroke();

    // 绘制十字线（帮助定位）
    const crossSize = 4;
    ctx.beginPath();
    ctx.moveTo(x - crossSize, y);
    ctx.lineTo(x + crossSize, y);
    ctx.moveTo(x, y - crossSize);
    ctx.lineTo(x, y + crossSize);
    ctx.strokeStyle = "#666";
    ctx.lineWidth = 1;
    ctx.stroke();

    ctx.restore();
  }

  private draw(e: MouseEvent) {
    if (!this.isDrawing || !this.drawingCtx || !this.currentStep) return;

    const point = this.getCanvasPoint(e);
    this.currentStep.points.push(point);
    this.cursorPosition = point;

    if (this.currentStep.points.length >= 2) {
      const lastTwoPoints = this.currentStep.points.slice(-2);

      this.drawingCtx.beginPath();
      this.drawingCtx.moveTo(lastTwoPoints[0].x, lastTwoPoints[0].y);
      this.drawingCtx.lineTo(lastTwoPoints[1].x, lastTwoPoints[1].y);

      if (this.currentStep.type === "pen") {
        this.drawingCtx.globalCompositeOperation = "source-over";
        this.drawingCtx.strokeStyle = this.currentStep.color || "#000";
        this.drawingCtx.lineWidth = this.currentStep.size;
      } else {
        // 橡皮擦模式
        this.drawingCtx.globalCompositeOperation = "destination-out";
        this.drawingCtx.strokeStyle = "rgba(0,0,0,1)"; // 使用不透明的颜色来擦除
        this.drawingCtx.lineWidth = this.currentStep.size;
      }
      this.drawingCtx.stroke();
    }

    // 更新主画布显示
    this.redrawCanvas();
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

  // [修改] 重绘方法，分层处理
  // 修改 redrawCanvas 方法
  private redrawCanvas() {
    if (!this.ctx || !this.canvas || !this.drawingCanvas) return;

    // 清空主画布
    // 清空并重绘白色背景
    this.ctx.fillStyle = "#ffffff";
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

    // 1. 先绘制背景（HTML内容）
    if (this.backgroundCanvas) {
      this.ctx.drawImage(
        this.backgroundCanvas,
        0,
        0,
        this.canvas.width,
        this.canvas.height
      );
    }

    // 2. 再绘制用户绘制的内容
    // 3. 绘制用户的绘制内容
    // this.ctx.globalCompositeOperation = 'source-over';
    this.ctx.drawImage(this.drawingCanvas, 0, 0);

    // 3. 最后绘制橡皮擦指示器
    if (this.currentTool === "eraser" && this.isMouseDown) {
      this.ctx.globalCompositeOperation = "source-over";
      this.drawEraserIndicator(
        this.ctx,
        this.cursorPosition.x,
        this.cursorPosition.y,
        this.eraserSize
      );
    }

    // 4. 如果正在录制，将当前画面同步到录制画布
    if (this.isRecording && this.recordingCtx && this.recordingCanvas) {
      this.recordingCtx.clearRect(
        0,
        0,
        this.recordingCanvas.width,
        this.recordingCanvas.height
      );
      this.recordingCtx.drawImage(this.canvas, 0, 0);
    }
  }

  // [新增] 重绘绘制画布的方法
  // 修改 redrawDrawingCanvas 方法
  private redrawDrawingCanvas() {
    if (!this.drawingCanvas || !this.drawingCtx) return;

    // 清空
    this.drawingCtx.clearRect(
      0,
      0,
      this.drawingCanvas.width,
      this.drawingCanvas.height
    );

    // 重绘历史记录
    this.drawHistory.forEach((step) => {
      if (step.points.length < 2) return;

      this.drawingCtx.beginPath();
      this.drawingCtx.moveTo(step.points[0].x, step.points[0].y);

      for (let i = 1; i < step.points.length; i++) {
        this.drawingCtx.lineTo(step.points[i].x, step.points[i].y);
      }

      if (step.type === "pen") {
        this.drawingCtx.globalCompositeOperation = "source-over";
        this.drawingCtx.strokeStyle = step.color || "#000";
        this.drawingCtx.lineWidth = step.size;
      } else {
        // 橡皮擦模式
        this.drawingCtx.globalCompositeOperation = "destination-out";
        this.drawingCtx.strokeStyle = "rgba(0,0,0,1)"; // 使用不透明的颜色来擦除
        this.drawingCtx.lineWidth = step.size;
      }
      this.drawingCtx.stroke();
    });

    // 重置合成模式
    this.drawingCtx.globalCompositeOperation = "source-over";

    // 更新主画布显示
    this.redrawCanvas();
  }

  // 添加选择颜色的方法
  private setColor(color: string) {
    this.currentColor = color;
    // 自动切换到画笔工具
    if (this.currentTool === "eraser") {
      this.setTool("pen");
    }
  }

  // 操作方法
  // 修改 setTool 方法
  private setTool(tool: "pen" | "eraser") {
    this.currentTool = tool;
    // 切换工具时重绘画布以显示/隐藏指示器
    this.redrawCanvas();
  }

  // 修改撤销方法
  private undo() {
    if (this.drawHistory.length > 0) {
      const step = this.drawHistory.pop()!;
      this.redoStack.push(step);
      this.redrawDrawingCanvas(); // 使用 redrawDrawingCanvas 而不是 redrawCanvas
    }
  }

  // 修改重做方法
  private redo() {
    if (this.redoStack.length > 0) {
      const step = this.redoStack.pop()!;
      this.drawHistory.push(step);
      this.redrawDrawingCanvas(); // 使用 redrawDrawingCanvas 而不是 redrawCanvas
    }
  }

  // [修改] 清除方法，只清除绘制画布
  private clear() {
    if (!this.drawingCanvas || !this.drawingCtx) return;

    this.drawHistory = [];
    this.redoStack = [];
    // 清空时重设白色背景
    this.drawingCtx.clearRect(
      0,
      0,
      this.drawingCanvas.width,
      this.drawingCanvas.height
    );
    this.redrawCanvas();
  }

  // 计算属性
  get canUndo(): boolean {
    return this.drawHistory.length > 0;
  }

  get canRedo(): boolean {
    return this.redoStack.length > 0;
  }

  async startRecording() {
    if (!this.canvas) return;

    try {
      // 1. 获取麦克风权限
      const audioStream = await navigator.mediaDevices.getUserMedia({
        audio: {
          echoCancellation: true, // 开启回声消除
          noiseSuppression: true, // 开启降噪
          sampleRate: 44100, // 设置采样率
        },
      });

      // 保存音频流的引用，用于之后释放
      this.audioStream = audioStream;

      // 创建 MediaRecorder
      const canvasStream = this.canvas.captureStream(30);

      // 3. 合并音视频流
      const combinedTracks = [
        ...canvasStream.getVideoTracks(),
        ...audioStream.getAudioTracks(),
      ];
      this.combinedStream = new MediaStream(combinedTracks);

      this.mediaRecorder = new MediaRecorder(this.combinedStream, {
        mimeType: "video/mp4",
      });

      // 设置数据处理事件
      this.mediaRecorder.ondataavailable = (event) => {
        if (event.data && event.data.size > 0) {
          this.recordedChunks.push(event.data);
        }
      };

      // 设置录制结束事件
      this.mediaRecorder.onstop = () => {
        this.finishRecording();
      };

      // 开始录制
      this.recordedChunks = [];
      this.recordingStartTime = Date.now();
      this.recordingTime = 0;

      this.isPaused = false;
      this.isRecording = true;
      // 启动录制计时器
      this.startRecordingTimer();
      // 添加录制状态类 - 使用 Vue 的 nextTick 确保 DOM 更新
      await this.$nextTick();
      // 添加录制状态类
      const container = this.$refs.canvasContainer as HTMLElement;
      if (container) {
        container.classList.add("recording");
      }
      // 最后才开始录制，避免错过初始帧
      this.mediaRecorder.start(100);
    } catch (err) {
      console.error("录制初始化失败:", err);
      // 重置所有状态
      this.isRecording = false;
      this.mediaRecorder = null;
      this.recordedChunks = [];
      // 移除录制状态类
      const container = this.$refs.canvasContainer as HTMLElement;
      if (container) {
        container.classList.remove("recording");
      }
      if ((err as Error).name === "NotAllowedError") {
        alert("无法访问麦克风，请确保已授予麦克风权限。");
      } else {
        alert("无法开始录制，请确保浏览器支持画布和音频录制功能。");
      }
      // alert("无法开始录制，请确保浏览器支持画布录制功能。");
    }
  }

  private cleanupStreams() {
    // 停止所有音频轨道
    if (this.audioStream) {
      this.audioStream.getTracks().forEach((track) => {
        track.stop();
      });
      this.audioStream = null;
    }

    // 停止合并的流
    if (this.combinedStream) {
      this.combinedStream.getTracks().forEach((track) => {
        track.stop();
      });
      this.combinedStream = null;
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
      this.cleanupStreams();
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
    if (this.recordedChunks.length === 0) {
      console.error("No recorded data available");
      return;
    }
    try {
      // 移除录制状态类
      const container = this.$refs.canvasContainer as HTMLElement;
      if (container) {
        container.classList.remove("recording");
      }
      // 等待过渡效果完成
      await this.$nextTick();
      const blob = new Blob(this.recordedChunks, {
        type: "video/mp4",
      });

      // 创建下载链接
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      document.body.appendChild(a);
      a.style.display = "none";
      a.href = url;
      a.download = `whiteboard-recording-${new Date().toISOString()}.mp4`;
      // 触发下载
      await this.$nextTick();
      a.click();

      // 清理资源
      setTimeout(() => {
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      }, 100);

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

  // 添加 CSS 过渡效果，减少视觉闪烁
  private updateStyle() {
    const style = document.createElement("style");
    style.textContent = `
    .canvas-container {
      transition: all 0.2s ease-out;
    }
    .canvas-container::after {
      transition: all 0.2s ease-out;
      opacity: 0;
    }
    .recording .canvas-container::after {
      opacity: 1;
    }
  `;
    document.head.appendChild(style);
  }

  private downloadRecording() {
    try {
      const blob = new Blob(this.recordedChunks, {
        type: "video/mp4",
      });

      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      document.body.appendChild(a);
      a.style.display = "none";
      a.href = url;
      a.download = `whiteboard-recording-${new Date().toISOString()}.mp4`;
      a.click();

      // 清理资源
      window.URL.revokeObjectURL(url);
      document.body.removeChild(a);
    } catch (error) {
      console.error("下载录制文件失败:", error);
      alert("下载录制文件时出现错误。");
    }
  }

  // 添加 watch 监听 initialContent 的变化
  @Watch("initialContent")
  async onInitialContentChanged() {
    const contentLayer = this.$refs.contentLayer as HTMLElement;
    if (contentLayer) {
      // 显示内容层
      contentLayer.style.visibility = "visible";
      contentLayer.style.opacity = "1";

      // 等待内容渲染
      await new Promise((resolve) => requestAnimationFrame(resolve));

      // 重新初始化背景
      await this.initBackgroundCanvas();
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
.color-picker {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px;
}

.color-picker-label {
  position: relative;
  display: inline-block;
  cursor: pointer;
}

.current-color {
  width: 40px;
  height: 40px;
  border-radius: 8px;
  border: 2px solid #eee;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.current-color:hover {
  transform: scale(1.05);
  box-shadow: 0 4px 8px rgba(0,0,0,0.15);
}

.custom-color-input {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
}

.color-presets {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
  margin-top: 8px;
}

.color-preset {
  width: 30px;
  height: 30px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  border: 2px solid transparent;
}

.color-preset:hover {
  transform: scale(1.1);
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.color-preset.active {
  border-color: #4a90e2;
  transform: scale(1.1);
}
</style>
