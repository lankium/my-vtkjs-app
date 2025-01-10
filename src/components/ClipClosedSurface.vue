<template>
  <div>
    <!-- vtk 渲染区域 -->
    <div ref="vtkContainer" />

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
        <!-- 控制裁剪平面的位置 -->
        <tr>
          <td>
            <!-- 平面1位置 -->
            <label for="clipPlane1Position">Clip Plane 1 Position:</label>
            <input type="range" id="clipPlane1Position" :min="minX" :max="maxX" :step="(maxX - minX) / 300"
              v-model="clipPlane1Position" />
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';

// 引入VTK库相关的模块
import '@kitware/vtk.js/Rendering/Profiles/Geometry';
import vtkFullScreenRenderWindow from '@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow';
import vtkActor from '@kitware/vtk.js/Rendering/Core/Actor';
import vtkMapper from '@kitware/vtk.js/Rendering/Core/Mapper';
import vtkXMLPolyDataReader from '@kitware/vtk.js/IO/XML/XMLPolyDataReader';  // 引入VTP文件读取器
import vtkPlane from '@kitware/vtk.js/Common/DataModel/Plane';  // 引入裁剪平面
import vtkClipClosedSurface from '@kitware/vtk.js/Filters/General/ClipClosedSurface';
import vtkSphereSource from '@kitware/vtk.js/Filters/Sources/SphereSource';
// 定义响应式数据

const vtkContainer = ref(null);  // 存储VTK渲染区域的引用
const context = ref(null);  // 存储渲染上下文（包含渲染窗口、渲染器等）
const representation = ref(2);  // 初始表示方式（2：表面）


// 裁剪平面相关数据
// 创建两个裁剪平面
const clipPlane1 = vtkPlane.newInstance();

const clipPlane1Position = ref(0);  // 平面1的位置

const clipPlane1Normal = [1, 0, 0];


const minX = ref(-100);
const maxX = ref(100);

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
    // Build pipeline
    const source = vtkSphereSource.newInstance({
      thetaResolution: 20,
      phiResolution: 11,
    });
    // 定义颜色常量
    const NAMED_COLORS = {
      BANANA: [227 / 255, 207 / 255, 87 / 255],
      TOMATO: [255 / 255, 99 / 255, 71 / 255],
      SANDY_BROWN: [244 / 255, 164 / 255, 96 / 255],
    };
    // const bounds = source.getOutputData().getBounds();
    const bounds = polyData.getBounds();
    console.log(polyData);

    // const bounds = polyData.getOutputData().getBounds();

    const center = [
      (bounds[1] + bounds[0]) / 2,
      (bounds[3] + bounds[2]) / 2,
      (bounds[5] + bounds[4]) / 2,
    ];
    const planes = [];
    const plane1 = vtkPlane.newInstance({
      origin: center,
      normal: [1.0, 0.0, 0.0],
    });
    planes.push(plane1);
    const plane2 = vtkPlane.newInstance({
      origin: center,
      normal: [0.0, 1.0, 0.0],
    });
    planes.push(plane2);
    const plane3 = vtkPlane.newInstance({
      origin: center,
      normal: [0.0, 0.0, 1.0],
    });
    // planes.push(plane3);
    minX.value = bounds[0];
    maxX.value = bounds[1];
    // console.log(Math.max(Math.abs(bounds[0]), Math.abs(bounds[1])));

    clipPlane1Position.value = minX.value

    const clipPlane1Origin = [
      clipPlane1Position.value * clipPlane1Normal[0],
      clipPlane1Position.value * clipPlane1Normal[1],
      clipPlane1Position.value * clipPlane1Normal[2],
    ];

    clipPlane1.setOrigin(clipPlane1Origin);
    clipPlane1.setNormal(clipPlane1Normal);

    // 创建 vtkClipClosedSurface 滤镜
    const filter = vtkClipClosedSurface.newInstance({
      clippingPlanes: planes, // 裁剪平面
      clipColor: NAMED_COLORS.BANANA, // 裁剪面颜色
      baseColor: NAMED_COLORS.TOMATO, // 基础表面颜色
      activePlaneColor: NAMED_COLORS.SANDY_BROWN, // 激活的裁剪面颜色
      passPointData: true, // 不传递点数据
    });

    // 假设 polyData 是你已经准备好的数据集
    filter.setInputData(polyData);
    // filter.setInputConnection(source.getOutputPort());
    filter.setScalarModeToColors(); // 设置颜色模式
    filter.update(); // 更新滤镜，应用裁剪

    const updatedData = filter.getOutputData(); // 获取更新后的数据
    // 设置Mapper的输入数据为加载的VTP数据
    mapper.setInputData(updatedData);

    // 根据当前选择的表示方式设置Actor的显示属性
    actor.getProperty().setRepresentation(representation.value);

    // 重置相机并重新渲染
    renderer.resetCamera();
    renderWindow.render();
  }
}

// 观察表示方式的变化，并更新渲染器
watch(representation, () => {
  if (context.value) {
    const { actor, renderWindow } = context.value;

    // 根据表示方式设置Actor的属性
    actor.getProperty().setRepresentation(Number(representation.value));
    // 重新渲染
    // renderer.resetCamera();
    renderWindow.render();
  }
});


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
      renderer,
      actor,
      mapper,
    };
  }
});

// 组件卸载时清理资源
onBeforeUnmount(() => {
  if (context.value) {
    const { fullScreenRenderer, actor, mapper } = context.value;

    // 删除对象，释放资源
    actor.delete();
    mapper.delete();
    fullScreenRenderer.delete();
    context.value = null;
  }
});
</script>

<style scoped>
/* 控制面板样式 */
.controls {
  position: absolute;
  top: 25px;
  left: 25px;
  background: white;
  padding: 12px;
}
</style>
