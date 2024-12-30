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
        <tr>
          <td>
            <!-- 平面2位置 -->
            <label for="clipPlane2Position">Clip Plane 2 Position:</label>
            <input type="range" id="clipPlane2Position" :min="minZ" :max="maxZ" :step="(maxZ - minZ) / 300"
              v-model="clipPlane2Position" />
          </td>
        </tr>
        <tr>
          <td>
            <!-- 控制平面1旋转 -->
            <label for="clipPlane1Rotation">Clip Plane 1 Rotation:</label>
            <input type="range" id="clipPlane1Rotation" min="0" max="180" step="1" v-model="clipPlane1RotationAngle" />
          </td>
        </tr>
        <tr>
          <td>
            <!-- 控制平面2旋转 -->
            <label for="clipPlane2Rotation">Clip Plane 2 Rotation:</label>
            <input type="range" id="clipPlane2Rotation" min="0" max="180" step="1" v-model="clipPlane2RotationAngle" />
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
import vtkMatrixBuilder from '@kitware/vtk.js/Common/Core/MatrixBuilder';

// 定义响应式数据

const vtkContainer = ref(null);  // 存储VTK渲染区域的引用
const context = ref(null);  // 存储渲染上下文（包含渲染窗口、渲染器等）
const representation = ref(2);  // 初始表示方式（2：表面）


// 裁剪平面相关数据
// 创建两个裁剪平面
const clipPlane1 = vtkPlane.newInstance();
const clipPlane2 = vtkPlane.newInstance();
const clipPlane1Position = ref(0);  // 平面1的位置
const clipPlane2Position = ref(0);  // 平面2的位置
const clipPlane1RotationAngle = ref(0);  // 平面1的旋转角度
const clipPlane2RotationAngle = ref(0);  // 平面2的旋转角度
const clipPlane1Normal = [1, 0, 0];
const clipPlane2Normal = [0, 0, 1];

const minX = ref(-100);
const maxX = ref(100);
// const minY = ref(-100);
// const maxY = ref(100);
const minZ = ref(-100);
const maxZ = ref(100);

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
    console.log('polyData.getPointData()', polyData.getPointData());

    console.log('polyData.getPointData().getScalars()', polyData.getPointData().getScalars());

    // 设置Mapper的输入数据为加载的VTP数据
    mapper.setInputData(polyData);

    const bounds = polyData.getBounds();
    minX.value = -1.3 * Math.max(Math.abs(bounds[0]), Math.abs(bounds[1]), Math.abs(bounds[2]), Math.abs(bounds[3]));
    maxX.value = 1.3 * Math.max(Math.abs(bounds[0]), Math.abs(bounds[1]), Math.abs(bounds[2]), Math.abs(bounds[3]));
    // console.log(Math.max(Math.abs(bounds[0]), Math.abs(bounds[1])));

    // minY.value = bounds[2];
    // maxY.value = bounds[3];
    console.log('bounds', bounds);
    minZ.value = -1.3 * Math.max(Math.abs(bounds[2]), Math.abs(bounds[3]), Math.abs(bounds[4]), Math.abs(bounds[5]));
    console.log('minZ.value', minZ.value);

    maxZ.value = 1.3 * Math.max(Math.abs(bounds[2]), Math.abs(bounds[3]), Math.abs(bounds[4]), Math.abs(bounds[5]));
    console.log('maxZ.value', maxZ.value);


    clipPlane1Position.value = minX.value
    clipPlane2Position.value = minZ.value

    const clipPlane1Origin = [
      clipPlane1Position.value * clipPlane1Normal[0],
      clipPlane1Position.value * clipPlane1Normal[1],
      clipPlane1Position.value * clipPlane1Normal[2],
    ];
    const clipPlane2Origin = [
      clipPlane2Position.value * clipPlane2Normal[0],
      clipPlane2Position.value * clipPlane2Normal[1],
      clipPlane2Position.value * clipPlane2Normal[2],
    ];
    clipPlane1.setOrigin(clipPlane1Origin);
    clipPlane1.setNormal(clipPlane1Normal);
    mapper.addClippingPlane(clipPlane1);

    clipPlane2.setOrigin(clipPlane2Origin);
    clipPlane2.setNormal(clipPlane2Normal);
    mapper.addClippingPlane(clipPlane2);

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

// 监听裁剪平面1位置的变化
watch(clipPlane1Position, () => {
  if (context.value) {
    const { renderWindow } = context.value;

    // 重新添加裁剪平面
    const clipPlane1Origin = [
      clipPlane1Position.value * clipPlane1Normal[0],
      clipPlane1Position.value * clipPlane1Normal[1],
      clipPlane1Position.value * clipPlane1Normal[2],
    ];
    clipPlane1.setOrigin(clipPlane1Origin);

    // 重新渲染
    renderWindow.render();
  }
});

// 监听裁剪平面1旋转的变化
watch(clipPlane1RotationAngle, (newVal, oldVal) => {
  const { renderWindow } = context.value;
  vtkMatrixBuilder
    .buildFromDegree()
    .rotate(Number(newVal) - Number(oldVal), [0, 0, 1])
    .apply(clipPlane1Normal);

  // 更新裁剪平面的位置
  clipPlane1.setNormal(clipPlane1Normal);
  renderWindow.render();
})

// 监听裁剪平面2位置的变化
watch(clipPlane2Position, () => {
  if (context.value) {
    const { renderWindow } = context.value;

    // 重新添加裁剪平面
    const clipPlane2Origin = [
      clipPlane2Position.value * clipPlane2Normal[0],
      clipPlane2Position.value * clipPlane2Normal[1],
      clipPlane2Position.value * clipPlane2Normal[2],
    ];
    clipPlane2.setOrigin(clipPlane2Origin);

    // 重新渲染
    renderWindow.render();
  }
});

// 监听裁剪平面2旋转的变化
watch(clipPlane2RotationAngle, (newVal, oldVal) => {
  const { renderWindow } = context.value;

  vtkMatrixBuilder
    .buildFromDegree()
    .rotate(Number(newVal) - Number(oldVal), [1, 0, 0])
    .apply(clipPlane2Normal);
  // 更新裁剪平面的位置
  console.log('clipPlane2Normal', clipPlane2Normal);

  clipPlane2.setNormal(clipPlane2Normal);
  renderWindow.render();
})

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
