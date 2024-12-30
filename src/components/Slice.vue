<template>
  <div ref="vtkContainer" style="width: 100%; height: 100%;"></div>
  <div class="controls">
    <!-- 切片按钮 -->
    <button @click="performSlice">切片</button>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import vtkFullScreenRenderWindow from '@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow';
import vtkActor from '@kitware/vtk.js/Rendering/Core/Actor';
import vtkMapper from '@kitware/vtk.js/Rendering/Core/Mapper';
import vtkPlane from '@kitware/vtk.js/Common/DataModel/Plane';
import vtkCutter from '@kitware/vtk.js/Filters/Core/Cutter';
import vtkPolyData from '@kitware/vtk.js/Common/DataModel/PolyData';
import vtkWidgetManager from '@kitware/vtk.js/Widgets/Core/WidgetManager';
import vtkImplicitPlaneWidget from '@kitware/vtk.js/Widgets/Widgets3D/ImplicitPlaneWidget';

const vtkContainer = ref(null);
const context = ref(null);

onMounted(() => {
  if (!vtkContainer.value) {
    console.error('Container not mounted');
    return;
  }

  // 创建全屏渲染窗口
  const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance({
    rootContainer: vtkContainer.value,
    background: [0.2, 0.3, 0.4], // 背景颜色
  });

  const renderer = fullScreenRenderer.getRenderer();
  const renderWindow = fullScreenRenderer.getRenderWindow();

  // 创建正方体的 PolyData
  const polyData = vtkPolyData.newInstance();
  const points = new Float32Array([
    0, 0, 0, 1, 0, 0, 1, 1, 0, 0, 1, 0, // 底面顶点
    0, 0, 1, 1, 0, 1, 1, 1, 1, 0, 1, 1, // 顶面顶点
  ]);
  const polys = new Uint32Array([
    4, 0, 1, 2, 3, // 底面
    4, 4, 5, 6, 7, // 顶面
    4, 0, 1, 5, 4, // 前面
    4, 1, 2, 6, 5, // 右面
    4, 2, 3, 7, 6, // 后面
    4, 3, 0, 4, 7, // 左面
  ]);
  polyData.getPoints().setData(points);
  polyData.getPolys().setData(polys);

  // 创建切割平面
  const plane = vtkPlane.newInstance();
  plane.setOrigin(0.5, 0.5, 0.5); // 初始位置
  plane.setNormal(1, 0, 0);       // 初始法线

  // 使用 Cutter 进行切割
  const cutter = vtkCutter.newInstance();
  cutter.setCutFunction(plane);
  cutter.setInputData(polyData);

  // 设置 Mapper 和 Actor
  const mapper = vtkMapper.newInstance();
  const actor = vtkActor.newInstance();
  mapper.setInputConnection(cutter.getOutputPort());
  actor.setMapper(mapper);

  // 添加到场景
  renderer.addActor(actor);

  // 初始化 Widget 管理器
  const widgetManager = vtkWidgetManager.newInstance();
  widgetManager.setRenderer(renderer);

  // 创建 ImplicitPlaneWidget
  const planeWidget = vtkImplicitPlaneWidget.newInstance();
  const widgetInstance = widgetManager.addWidget(planeWidget, 'manipulator');

  // 初始化平面 Widget 的位置和参数
  // widgetInstance.setPlaceFactor(1.25);
  widgetInstance.placeWidget(polyData.getBounds());

  // 设置平面控件的初始法线和原点
  widgetInstance.getWidgetState().setNormal(1, 0, 0);
  widgetInstance.getWidgetState().setOrigin(0.5, 0.5, 0.5);

  // 渲染场景
  renderer.resetCamera();
  renderWindow.render();

  // 保存上下文
  context.value = {
    fullScreenRenderer,
    renderWindow,
    widgetManager,
    widgetInstance,
    cutter,
    mapper,
    actor,
    plane,
    renderer,
  };
});

// 切片操作
function performSlice() {
  if (!context.value) {
    console.error('Context is not available');
    return;
  }

  const { widgetInstance, plane, renderWindow } = context.value;

  // 获取当前 Widget 的法线和原点
  const normal = widgetInstance.getWidgetState().getNormal();
  const origin = widgetInstance.getWidgetState().getOrigin();

  // 更新切割平面
  plane.setNormal(normal);
  plane.setOrigin(origin);

  // 重新渲染切割结果
  renderWindow.render();

  console.log('切片操作已执行');
}

onUnmounted(() => {
  if (context.value) {
    context.value.fullScreenRenderer.delete();
    context.value.widgetManager.delete();
  }
});
</script>

<style scoped>
/* 控制按钮样式 */
.controls {
  position: absolute;
  top: 10px;
  left: 10px;
  background: rgba(255, 255, 255, 0.8);
  padding: 8px;
  border-radius: 4px;
}

button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  background-color: #007bff;
  color: white;
  font-size: 14px;
  cursor: pointer;
}

button:hover {
  background-color: #0056b3;
}
</style>
