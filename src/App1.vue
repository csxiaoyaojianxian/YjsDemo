<template>
  <div class="container">
      <canvas ref="canvasRef" @mousedown="startDrawing" @mousemove="draw"
          @mouseup="stopDrawing" @mouseleave="stopDrawing">
      </canvas>
  </div>
  <div style="position: absolute; bottom: 10px; display: flex; justify-content: center; height: 40px; width: 100vw;">
      <div style="width: 100px; height: 40px; display: flex; align-items: center; justify-content: center; color: white;"
          :style="{ backgroundColor: color }">
          <span>当前颜色</span>
      </div>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="switchMode(DrawType.Point)">画点</button>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="switchMode(DrawType.Line)">直线</button>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="switchMode(DrawType.Draw)">涂鸦</button>
      <button style="width: 100px; height: 40px; margin-left: 10px;" @click="clearCanvas">清除</button>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';
import { v4 as uuidv4 } from 'uuid';

const urlParams = new URLSearchParams(new URL(window.location.href).search);
const demoId = urlParams.get('id') || 'demo';

const canvasRef = ref<null | HTMLCanvasElement>(null);
const ctx = ref<CanvasRenderingContext2D | null>(null);
const drawing = ref(false); // 当前绘画状态
const color = ref<string>("black");

class Point {
  x: number = 0.0;
  y: number = 0.0;
}

enum DrawType {
  None,
  Point,
  Line,
  Draw,
}

const colors = [
  "#FF5733", "#33FF57", "#5733FF", "#FF33A2", "#A2FF33",
  "#33A2FF", "#FF33C2", "#C2FF33", "#33C2FF", "#FF3362",
  "#6233FF", "#FF336B", "#6BFF33", "#33FFA8", "#A833FF",
  "#33FFAA", "#AA33FF", "#FFAA33", "#33FF8C", "#8C33FF"
];

// 随机选择一个颜色
function getRandomColor() {
  const randomIndex = Math.floor(Math.random() * colors.length);
  return colors[randomIndex];
}

class DrawElementProp {
  color: string = "black";
}

class DrawElement {
  id: string = "";
  version: string = "";
  type: DrawType = DrawType.None;
  geometry: Point[] = [];
  properties: DrawElementProp = new DrawElementProp();
}

// 当前绘画模式
const drawMode = ref<DrawType>(DrawType.Draw);
// 定义变量来跟踪第一个点的坐标和鼠标是否按下
const point = ref<Point | null>(null);

// 创建 ydoc，作为 Yjs 的顶层容器，代表一个独立的协作文档
// 所有共享数据类型（如 Map、Array、Text 等）都挂载在 Y.Doc 下
// 核心特性：
//  状态管理：维护所有共享数据的状态和变更历史。
//  事务机制：通过 transact 方法批量操作数据，确保原子性。
//  协同同步：通过 Provider（如 y-websocket）与其他客户端同步变更
// 【Yjs】
const ydoc = new Y.Doc();

// 创建 websocketProvider
// 【Yjs】
const websocketProvider = new WebsocketProvider(
  'ws://localhost:10086/ws', demoId, ydoc
)

// 创建一个 Yjs Map，用于存储绘图数据
// 【Yjs】
const drawingData = ydoc.getMap<DrawElement>('drawingData');
// const drawingData = ydoc.getArray<DrawElement>('drawingData');
drawingData.observe(event => {
  console.log('observer data', event);
  if (ctx.value && canvasRef.value) {
    const context = ctx.value!
    // 清空 Canvas
    context.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);
    // 遍历绘图数据，绘制点、路径等
    drawingData.forEach((data: DrawElement) => {
      doAutoPaint(data, context);
    })
  }
})

// 记录当前路径的编号
const currentID = ref<string>("");

// 编辑器界面用户触发绘画
const startDrawing = (e: any) => {
  const version = new Date().getTime().toString();
  if (ctx.value) {

    // 画点
    if (drawMode.value === DrawType.Point) {
      // 分配编号
      currentID.value = uuidv4();
      // 增加记录
      let point: DrawElement = {
        id: currentID.value,
        version: version,
        type: DrawType.Point,
        geometry: [{ x: e.clientX, y: e.clientY }],
        properties: { color: color.value }
      }
      // 【Yjs】
      drawingData.set(currentID.value, point);

      // 重置编号
      currentID.value = "";
      return;
    }

    // 画线
    if (drawMode.value === DrawType.Line) {
      // 分配编号
      if (currentID.value == "") {
        currentID.value = uuidv4();
      }
      if (!drawing.value) {
        // 开始绘画
        drawing.value = true;
      }
      // 获取当前线的信息，如果没有则创建
      // 【Yjs】
      let line: DrawElement | undefined = drawingData.get(currentID.value)

      if (line) {
        line.version = version;
        line.geometry.push({ x: e.clientX, y: e.clientY });
      } else {
        line = {
          id: currentID.value,
          version: version,
          type: DrawType.Line,
          geometry: [{ x: e.clientX, y: e.clientY }],
          properties: { color: color.value }
        }
      }
      // 【Yjs】
      drawingData.set(currentID.value, line);
      return
    }

    // 涂鸦
    if (drawMode.value === DrawType.Draw) {
      // 分配编号
      if (currentID.value == "") {
        currentID.value = uuidv4();
        let path: DrawElement = {
          id: currentID.value,
          version: version,
          type: DrawType.Draw,
          geometry: [{ x: e.clientX, y: e.clientY }],
          properties: { color: color.value }
        }
        // 【Yjs】
        drawingData.set(currentID.value, path);
      }
      if (!drawing.value) {
        // 开始绘画
        drawing.value = true;
      }
    }
  }
};
const draw = (e: any) => {
  if (drawing.value && ctx.value) {
    // 涂鸦模式持续绘画
    if (drawMode.value === DrawType.Draw) {
      // 获取当前线的信息，如果没有则创建
      // 【Yjs】
      let path: DrawElement | undefined = drawingData.get(currentID.value)
      if (path) {
        path.geometry.push({ x: e.clientX, y: e.clientY });
        // 【Yjs】
        console.log("path", path)
        drawingData.set(currentID.value, path);
        return
      }
      console.log("error: not found path", currentID.value)
    }
  }
};
const stopDrawing = () => {
  if (drawing.value && ctx.value) {
    if (drawMode.value === DrawType.Draw) {
      // 鼠标放开时，关闭当前路径绘画
      currentID.value = "";
      drawing.value = false;
    }
  }
};

// 数据遍历绘制(非人工绘制)
const doAutoPaint = (data: DrawElement, context) => {
  if (data.type == DrawType.Point) {
    context.fillStyle = data.properties.color; // 设置点的填充颜色
    context.strokeStyle = data.properties.color; // 设置点的边框颜色
    context.beginPath();
    context.moveTo(data.geometry[0].x, data.geometry[0].y);
    context.arc(data.geometry[0].x, data.geometry[0].y, 2.5, 0, Math.PI * 2); // 创建一个圆形路径
    context.fill(); // 填充路径，形成圆点
    context.closePath();
  } else if (data.type == DrawType.Line) {
    context.fillStyle = data.properties.color; // 设置点的填充颜色
    context.strokeStyle = data.properties.color; // 设置点的边框颜色
    context.beginPath();
    // 遍历所有点
    data.geometry.forEach((p: Point, index: number) => {
      if (index == 0) {
        context.moveTo(p.x, p.y);
        context.fillRect(p.x, p.y, 5, 5);
      } else {
        context.lineTo(p.x, p.y);
        context.stroke();
        context.fillRect(p.x, p.y, 5, 5);
      }
    })
  } else if (data.type == DrawType.Draw) {
    context.fillStyle = data.properties.color; // 设置点的填充颜色
    context.strokeStyle = data.properties.color; // 设置点的边框颜色
    context.beginPath();
    // 遍历所有点
    data.geometry.forEach((p: Point, index: number) => {
      if (index == 0) {
        context.moveTo(p.x, p.y);
      } else {
        context.lineTo(p.x, p.y);
        context.stroke();
      }
    })
  } else {
    console.log("Invalid draw data", data)
  }
}


onMounted(() => {
  if (canvasRef.value) {
    // 初始化后随机选择一种颜色
    color.value = getRandomColor()
    canvasRef.value.height = window.innerHeight - 10;
    canvasRef.value.width = window.innerWidth;
    const context = canvasRef.value.getContext('2d');
    if (context) {
      ctx.value = context;
      context.lineWidth = 5;
      context.fillStyle = color.value; // 设置点的填充颜色
      context.strokeStyle = color.value; // 设置点的边框颜色
      context.lineJoin = 'round';
    }
  }
  window.addEventListener('keydown', handleKeyDown);
});
const handleKeyDown = (event: KeyboardEvent) => {
  if (event.key === 'Escape') {
    // 重置编号
    if (currentID.value) {
        currentID.value = "";
    }
    // 结束路径和绘画
    if (drawing.value && ctx.value) {
        ctx.value.closePath();
        drawing.value = false;
    }
  }
}

// 模式切换
const switchMode = (mode: DrawType) => {
  // 重置状态
  currentID.value = "";
  drawing.value = false;
  drawMode.value = mode;
  point.value = null
}

// 清空画布
const clearCanvas = () => {
  if (canvasRef.value && ctx.value) {
    ctx.value.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);
    drawingData.clear();
  }
};
</script>

<style>
.container {
  width: 100%; height: 100%; overflow: hidden;
}
</style>