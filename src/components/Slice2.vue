<template>
  <div class="container">
    <div ref="vtkContainer" style="width: 500px; height: 500px;"></div>
  </div>

  <div ref="sliceContainer" style="width: 100%; height: 300px; margin-top: 20px;"></div> <!-- 用于渲染切片 -->
  <!-- 控制面板，用于调整模型的表现和分辨率 -->
  <table class="controls">
    <tbody>
      <tr>
        <td>
          <!-- 选择不同的表示方式: 点、线框、表面 -->
          <select style="width: 100%" v-model="representation">
            <option value="0">Points</option> <!-- 点表示 -->
            <option value="1">Wireframe</option> <!-- 线框表示 -->
            <option value="2">Surface</option> <!-- 表面表示 -->
          </select>
        </td>
      </tr>
      <tr>
        <td>
          <!-- 文件输入，用于加载 VTP 文件 -->
          <input type="file" @change="loadVtpFile" />
        </td>
      </tr>
      <tr>
        <td>
          <button @click="performSlice">切片</button>
          <label for="showModel">显示模型</label>
          <input type="checkbox" id="showModel" v-model="showModel" />
          <label for="showWidget">显示Widget</label>
          <input type="checkbox" id="showWidget" v-model="showWidget" />
        </td>
        <td>
          <button @click="clearSlices">取消切片</button>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script setup>
import { watch, ref, onMounted, onUnmounted } from 'vue';
import vtkFullScreenRenderWindow from '@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow';
import vtkActor from '@kitware/vtk.js/Rendering/Core/Actor';
import vtkMapper from '@kitware/vtk.js/Rendering/Core/Mapper';
import vtkPlane from '@kitware/vtk.js/Common/DataModel/Plane';
import vtkCutter from '@kitware/vtk.js/Filters/Core/Cutter';

import vtkXMLPolyDataReader from '@kitware/vtk.js/IO/XML/XMLPolyDataReader';  // 引入VTP文件读取器
import vtkWidgetManager from '@kitware/vtk.js/Widgets/Core/WidgetManager';
import vtkImplicitPlaneWidget from '@kitware/vtk.js/Widgets/Widgets3D/ImplicitPlaneWidget';

const vtkContainer = ref(null);
const context = ref(null);
const representation = ref(2);  // 初始表示方式（2：表面）
const showModel = ref(true);
const showWidget = ref(false);
// 加载VTP文件
function loadVtpFile(event) {
  const file = event.target.files[0];  // 获取用户选择的文件
  if (file) {
    const reader = new FileReader();  // 创建文件读取器
    reader.onload = () => {
      const fileContent = reader.result;  // 读取文件内容
      const vtkReader = vtkXMLPolyDataReader.newInstance();  // 创建VTP读取器
      vtkReader.parseAsArrayBuffer(fileContent);  // 解析文件内容为VTP数据

      const polyData = vtkReader.getOutputData();  // 获取解析后的数据
      updateRendererWithVtp(polyData);  // 更新渲染器
    };
    reader.readAsArrayBuffer(file);  // 读取文件内容为ArrayBuffer
  }
}

// 使用加载的VTP数据更新渲染器
function updateRendererWithVtp(polyData) {
  if (context.value) {
    const { actor, mapper, renderWindow, renderer } = context.value;
    console.log(polyData);

    // console.log('polyData.getCellData().getScalars().getName()', polyData.getCellData().getScalars().getName());

    // console.log('polyData.getCellData().getScalars().getRange()', polyData.getCellData().getScalars().getRange());
    // console.log('polyData.getCellData().getScalars().getRange()', polyData.getCellData().getScalars().getRange());
    // 获取标量数据并将其应用到模型

    // 设置Mapper的输入数据为加载的VTP数据
    mapper.setInputData(polyData);
    const bounds = polyData.getBounds();

    // ----------------------------------------------------------------------------
    // Widget manager
    // ----------------------------------------------------------------------------

    // 创建切割平面
    const plane = vtkPlane.newInstance();
    plane.setOrigin((bounds[1] + bounds[0]) / 2, (bounds[3] + bounds[2]) / 2, (bounds[5] + bounds[4]) / 2); // 初始位置
    plane.setNormal(1, 0, 0);       // 初始法线

    // 使用 Cutter 进行切割
    const cutter = vtkCutter.newInstance();
    cutter.setCutFunction(plane);
    cutter.setInputData(polyData);

    // 设置 Mapper 和 Actor
    // mapper.setInputConnection(cutter.getOutputPort());

    // 添加到场景
    context.value.plane = plane;
    context.value.cutter = cutter;
    context.value.polyData = polyData
    // 根据当前选择的表示方式设置Actor的显示属性
    actor.getProperty().setRepresentation(representation.value);
    // 重置相机并重新渲染
    renderer.resetCamera();

    renderWindow.render();
  }
}

// 组件挂载时初始化VTK渲染
onMounted(() => {

  if (!context.value) {
    // 创建全屏渲染窗口
    const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance({
      rootContainer: vtkContainer.value,
    });

    // 创建Mapper
    const mapper = vtkMapper.newInstance();

    const actor = vtkActor.newInstance();
    actor.setMapper(mapper);

    // 获取渲染器和渲染窗口
    const renderer = fullScreenRenderer.getRenderer();
    const renderWindow = fullScreenRenderer.getRenderWindow();

    // 将Actor添加到渲染器
    renderer.addActor(actor);

    // 重置相机并开始渲染
    renderer.resetCamera();
    renderWindow.render();

    // 存储上下文，包含渲染器、窗口、Actor和Mapper
    context.value = {
      fullScreenRenderer,
      renderWindow,
      widgetInstance: null,
      renderer,
      actor,
      mapper,
    };
  }
});

let sliceActors = [];  // 存储所有切割面



// 切片操作
function performSlice() {
  if (!context.value) {
    console.error('Context is not available');
    return;
  }

  const { renderer, renderWindow, polyData } = context.value;

  // 获取当前 Widget 的法线和原点
  const { widgetInstance } = context.value;
  const normal = widgetInstance.getWidgetState().getNormal();
  const origin = widgetInstance.getWidgetState().getOrigin();

  // 创建新的切割平面
  const plane = vtkPlane.newInstance();
  plane.setNormal(normal);
  plane.setOrigin(origin);

  // 创建新的 Cutter 对象
  const cutter = vtkCutter.newInstance();
  cutter.setCutFunction(plane);
  cutter.setInputData(polyData);

  // 创建新的 Mapper 和 Actor
  const cutterMapper = vtkMapper.newInstance();
  cutterMapper.setInputConnection(cutter.getOutputPort());

  const cutterActor = vtkActor.newInstance();
  cutterActor.setMapper(cutterMapper);
  // cutterActor.getProperty().setColor(1, 0, 0);  // 将切割面颜色设置为红色
  // 将新的切割 Actor 添加到渲染器中
  renderer.addActor(cutterActor);

  // 保存切割 Actor 以便后续删除
  sliceActors.push(cutterActor);

  // 重新渲染
  renderWindow.render();

  console.log('切片操作已执行');
}

// 取消所有切片
function clearSlices() {
  if (context.value) {
    const { renderer, renderWindow } = context.value;

    // 移除所有切割面
    sliceActors.forEach(actor => renderer.removeActor(actor));
    sliceActors = [];  // 清空存储的切割面

    // 重新渲染
    renderWindow.render();

    console.log('所有切割面已清除');
  }
}

onUnmounted(() => {
  if (context.value) {
    const { fullScreenRenderer, actor, mapper } = context.value;
    // 删除对象，释放资源
    actor.delete();
    mapper.delete();
    fullScreenRenderer.delete();
    context.value = null;
  }
});

watch(representation, (newValue) => {
  if (context.value) {
    const { actor, renderWindow } = context.value;
    actor.getProperty().setRepresentation(newValue);
    renderWindow.render();
  }
});
watch(showModel, () => {
  if (context.value) {
    const { actor, renderWindow } = context.value;
    actor.setVisibility(showModel.value);
    renderWindow.render();
  }
})

// 控制widget显示/隐藏
watch(showWidget, () => {
  if (context.value) {
    const { renderer, widgetInstance, renderWindow, polyData } = context.value;
    console.log('widgetInstance', widgetInstance);
    // const isEnabled = widgetInstance.getEnabled();
    // 启用或禁用控件的交互
    if (widgetInstance && widgetInstance.getEnabled()) {
      widgetInstance.setEnabled(showWidget.value);  // 切换交互功能

      if (showWidget.value) {
        // 显示控件
        widgetInstance.placeWidget(); // 重新放置控件，如果需要
        widgetInstance.setVisibility(true);
      } else {
        // 隐藏控件
        widgetInstance.setVisibility(false);
      }
    } else {
      // 初始化 Widget 管理器
      const widgetManager = vtkWidgetManager.newInstance();
      widgetManager.setRenderer(renderer);

      // // 创建 ImplicitPlaneWidget
      const planeWidget = vtkImplicitPlaneWidget.newInstance();
      const widgetInstance = widgetManager.addWidget(planeWidget, 'manipulator');

      // // 初始化平面 Widget 的位置和参数
      widgetInstance.placeWidget(polyData.getBounds());
      const bounds = polyData.getBounds();
      // // 设置平面控件的初始法线和原点
      widgetInstance.getWidgetState().setNormal(1, 0, 0);
      widgetInstance.getWidgetState().setOrigin((bounds[1] + bounds[0]) / 2, (bounds[3] + bounds[2]) / 2, (bounds[5] + bounds[4]) / 2);
      widgetManager.enablePicking();
      context.value.widgetInstance = widgetInstance;
    }

    renderWindow.render();
  }
})
</script>

<style scoped>
/* 控制按钮样式 */
.container {
  width: 1000px;
  height: 500px;
}

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
