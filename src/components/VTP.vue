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
        <!-- 添加透明度控制 -->
        <tr>
          <td>
            <label for="opacityControl">Opacity:</label>
            <input type="range" id="opacityControl" min="0" max="1" step="0.01" v-model="opacity" />
          </td>
        </tr>
        <tr>
          <td>
            <label for="minScalarValue">Min Scalar Value:</label>
            <input type="number" id="minScalarValue" v-model="minScalarValue" />
          </td>
        </tr>
        <tr>
          <td>
            <label for="maxScalarValue">Max Scalar Value:</label>
            <input type="number" id="maxScalarValue" v-model="maxScalarValue" />
          </td>
        </tr>
        <tr>
          <td>
            <label for="useColorTransfer">Use ColorTransferFunction:</label>
            <input type="checkbox" id="useColorTransfer" v-model="useColorTransfer" />
          </td>
        </tr>

        <tr>
          <td>
            <label for="scalarBarVisibility">Show Scalar Bar:</label>
            <input type="checkbox" id="scalarBarVisibility" v-model="scalarBarVisible" />
          </td>
        </tr>
        <tr>
          <td>
            <label for="numberOfColors:">Number of colors:</label>
            <input type="number" id="numberOfColors" v-model="numberOfColorsInput" />
          </td>
        </tr>
        <!-- 控制裁剪平面的位置 -->
        <tr>
          <td>
            <!-- 平面1位置 -->
            <label for="clipPlane1Position">Clip Plane 1 Position:</label>
            <input type="range" id="clipPlane1Position" :min="minX" :max="maxX" :step="(maxX - minX) / 200"
              v-model="clipPlane1Position" />
          </td>
        </tr>
        <tr>
          <td>
            <!-- 平面2位置 -->
            <label for="clipPlane2Position">Clip Plane 2 Position:</label>
            <input type="range" id="clipPlane2Position" :min="minZ" :max="maxZ" :step="(maxZ - minZ) / 200"
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
import vtkScalarBarActor from '@kitware/vtk.js/Rendering/Core/ScalarBarActor';
import vtkColorTransferFunction from '@kitware/vtk.js/Rendering/Core/ColorTransferFunction';
import vtkLookupTable from '@kitware/vtk.js/Common/Core/LookupTable';

import { Scale } from '@kitware/vtk.js/Rendering/Core/ColorTransferFunction/Constants';

import * as d3 from 'd3-scale';
import { formatDefaultLocale } from 'd3-format';
// 定义响应式数据
const opacity = ref(1);  // 默认透明度为 1，完全不透明

const minScalarValue = ref(0);
const maxScalarValue = ref(1);

const scalarBarVisible = ref(true); // 是否显示 scalar bar
const useColorTransfer = ref(false); // 是否使用颜色转换函数
const numberOfColorsInput = ref(255);

const lut = ref(null)

const vtkContainer = ref(null);  // 存储VTK渲染区域的引用
const context = ref(null);  // 存储渲染上下文（包含渲染窗口、渲染器等）
const representation = ref(2);  // 初始表示方式（2：表面）

let scalarBarActor = vtkScalarBarActor.newInstance();

// 裁剪平面相关数据
// 创建两个裁剪平面
const clipPlane1 = vtkPlane.newInstance();
const clipPlane2 = vtkPlane.newInstance();
const clipPlane1Position = ref(0);  // 平面1的位置
const clipPlane2Position = ref(0);  // 平面2的位置
const clipPlane1RotationAngle = ref(0);  // 平面1的旋转角度
const clipPlane2RotationAngle = ref(0);  // 平面2的旋转角度
const clipPlane1Normal = [1, 0, 0];
const clipPlane2Normal = [0, 0, -1];

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

// Change the number of ticks (TODO: add numberOfTicks to ScalarBarActor)
function generateTicks(numberOfTicks) {
  return (helper) => {
    const lastTickBounds = helper.getLastTickBounds();
    // compute tick marks for axes
    const scale = d3
      .scaleLinear()
      .domain([0.0, 1.0])
      .range([lastTickBounds[0], lastTickBounds[1]]);
    const samples = scale.ticks(numberOfTicks);
    const ticks = samples.map((tick) => scale(tick));
    // Replace minus "\u2212" with hyphen-minus "\u002D" so that parseFloat() works
    formatDefaultLocale({ minus: '\u002D' });
    const format = scale.tickFormat(
      ticks[0],
      ticks[ticks.length - 1],
      numberOfTicks
    );

    const tickStrings = ticks
      .map(format)
      .map((tick) => Number(parseFloat(tick).toPrecision(12)).toPrecision()); // d3 sometimes adds unwanted whitespace
    helper.setTicks(ticks);
    helper.setTickStrings(tickStrings);
  };
}

// 使用加载的VTP数据更新渲染器
function updateRendererWithVtp(polyData) {
  if (context.value) {
    const { actor, mapper, renderWindow, renderer } = context.value;
    lut.value = mapper.getLookupTable();
    console.log(polyData);

    // console.log('polyData.getCellData().getScalars().getName()', polyData.getCellData().getScalars().getName());

    // console.log('polyData.getCellData().getScalars().getRange()', polyData.getCellData().getScalars().getRange());
    // console.log('polyData.getCellData().getScalars().getRange()', polyData.getCellData().getScalars().getRange());
    // 获取标量数据并将其应用到模型
    const scalars = polyData.getCellData().getScalars();
    console.log(scalars);

    mapper.setColorModeToMapScalars();
    lut.value.setVectorModeToMagnitude();
    if (scalars) {
      // 获取密度数据的名称并应用到颜色映射
      console.log("Scalar data name:", scalars.getName());
      console.log("Scalar data range:", scalars.getRange());
      // 获取密度值的最小值和最大值
      const scalarRange = scalars.getRange();
      minScalarValue.value = scalarRange[0];
      maxScalarValue.value = scalarRange[1];

      // 设置标量数据范围
      // scalars.setRange(minScalarValue.value, maxScalarValue.value);

      // 设置颜色映射
      // const colorTransferFunction = vtkColorTransferFunction.newInstance();
      // colorTransferFunction.addRGBPoint(minScalarValue.value, 0.0, 0.0, 1.0);  // 最小密度 -> 蓝色
      // colorTransferFunction.addRGBPoint(maxScalarValue.value, 1.0, 0.0, 0.0);  // 最大密度 -> 红色
      // mapper.setLookupTable(colorTransferFunction);
      mapper.setInputData(polyData);  // 更新 Mapper 的输入数据为加载的 VTP 数据
    }

    // 获取简化后的 PolyData

    mapper.setInputData(polyData);

    lut.value.setRange(parseFloat(minScalarValue.value), parseFloat(maxScalarValue.value));
    scalarBarActor.setScalarsToColors(lut.value);
    // 设置 ScalarBar 的显示
    if (scalarBarVisible.value) {
      renderer.addActor(scalarBarActor);
    } else {
      renderer.removeActor(scalarBarActor);
    }
    polyData.ComputeBounds()
    const bounds = polyData.getBounds();
    console.log('bounds', bounds);
    // -90.0999984741211, 65.39999389648438, -15.600000381469727, 6539.39990234375, -1940.18994140625, 12.8100004196167
    minX.value = -Math.max(Math.abs(bounds[0]), Math.abs(bounds[1]), Math.abs(bounds[2]), Math.abs(bounds[3]));
    maxX.value = Math.max(Math.abs(bounds[0]), Math.abs(bounds[1]), Math.abs(bounds[2]), Math.abs(bounds[3]));
    // console.log(Math.max(Math.abs(bounds[0]), Math.abs(bounds[1])));

    // minY.value = bounds[2];
    // maxY.value = bounds[3];
    minZ.value = -Math.max(Math.abs(bounds[4]), Math.abs(bounds[5]));
    maxZ.value = Math.max(Math.abs(bounds[4]), Math.abs(bounds[5]));

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
    // ----------------------------------------------------------------------------
    // Widget manager
    // ----------------------------------------------------------------------------

    // 根据当前选择的表示方式设置Actor的显示属性
    actor.getProperty().setRepresentation(representation.value);

    // 重新应用裁剪平面
    // addClippingPlanes();
    // 重置相机并重新渲染
    renderer.resetCamera();
    renderWindow.render();
  }
}

// 监听透明度变化
watch(opacity, (newOpacity) => {
  if (context.value) {
    const { actor, renderWindow } = context.value;

    // 更新Actor的透明度
    actor.getProperty().setOpacity(newOpacity);

    // 重新渲染
    renderWindow.render();
  }
});

// 监听 Scalar 范围变化
watch(minScalarValue, () => {
  if (context.value) {
    const { mapper, renderWindow } = context.value;
    // 更新标量数据范围
    let lut = mapper.getLookupTable();
    mapper.setUseLookupTableScalarRange(true);
    lut.setRange(parseFloat(minScalarValue.value), lut.getRange()[1]);
    renderWindow.render();
  }
});

// 监听 Scalar 范围变化
watch(maxScalarValue, () => {
  if (context.value) {
    const { mapper, renderWindow } = context.value;
    // 更新标量数据范围
    let lut = mapper.getLookupTable();
    mapper.setUseLookupTableScalarRange(true);
    lut.setRange(lut.getRange()[0], parseFloat(maxScalarValue.value));
    renderWindow.render();
  }
});

// 监听 Scalar Bar 显示状态变化
watch(scalarBarVisible, () => {

  if (context.value) {
    const { renderer, renderWindow } = context.value;
    if (scalarBarVisible.value) {
      renderer.addActor(scalarBarActor);
    } else {

      renderer.removeActor(scalarBarActor);
    }
    renderWindow.render()
  }
});

watch(useColorTransfer, () => {
  // 设置颜色映射
  if (context.value) {
    const { mapper, renderWindow } = context.value;
    if (useColorTransfer.value) {
      const ctf = vtkColorTransferFunction.newInstance({
        discretize: true,
        numberOfValues: parseInt(numberOfColorsInput.value, 10),
        scale: Scale.LOG10,
      });
      ctf.addRGBPoint(1, 1.0, 0.0, 0.0);  // 最大密度 -> 红色
      // 中间密度 -> 绿色
      ctf.addRGBPoint(0, 0.0, 0.0, 1.0);  // 最小密度 -> 蓝色

      mapper.setLookupTable(ctf);

    } else {
      const numberOfColors = parseInt(numberOfColorsInput.value, 10);
      mapper.setLookupTable(vtkLookupTable.newInstance({ numberOfColors }));
    }

    lut.value = mapper.getLookupTable();
    lut.value.setRange(parseFloat(minScalarValue.value), parseFloat(maxScalarValue.value));
    mapper.setColorModeToMapScalars();
    lut.value.setVectorModeToMagnitude();
    scalarBarActor.setScalarsToColors(lut.value);
    renderWindow.render()
  }
})

watch(numberOfColorsInput, () => {
  if (context.value) {
    const { mapper, renderWindow } = context.value;
    const lut = mapper.getLookupTable();

    if (lut.isA('vtkLookupTable')) {
      lut.setNumberOfColors(parseInt(numberOfColors.value, 10));
      lut.modified();
      lut.build();
    } else {
      const numberOfColors = parseInt(numberOfColorsInput.value, 10);
      mapper.setLookupTable(vtkLookupTable.newInstance({ numberOfColors }));
    }
    lut.modified();
    mapper.setColorModeToMapScalars();
    lut.value.setVectorModeToMagnitude();
    scalarBarActor.setScalarsToColors(lut);
    scalarBarActor.modified();
    renderWindow.render();
  }
})

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
    // renderer.resetCamera();
    renderWindow.render();
  }
});
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
    // renderer.resetCamera();
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
// 监听裁剪平面2旋转的变化
watch(clipPlane2RotationAngle, (newVal, oldVal) => {
  const { renderWindow } = context.value;

  vtkMatrixBuilder
    .buildFromDegree()
    .rotate(Number(newVal) - Number(oldVal), [0, 1, 0])
    .apply(clipPlane2Normal);
  // 更新裁剪平面的位置

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
    const mapper = vtkMapper.newInstance({
      interpolateScalarsBeforeMapping: true,
      useLookupTableScalarRange: false,
      lookupTable: vtkLookupTable.newInstance(),
      scalarVisibility: true,
    });

    // 设置标量数据范围

    lut.value = mapper.getLookupTable();
    lut.value.setRange(parseFloat(minScalarValue.value), parseFloat(maxScalarValue.value));
    // scalarBarActor.setAutomated(true)
    // scalarBarActor.setVisibility(scalarBarVisible.value);
    scalarBarActor.setGenerateTicks(generateTicks(10));
    scalarBarActor.setScalarsToColors(lut.value);
    // 设置 ScalarBar 的显示

    // 创建Actor并将Mapper关联到Actor
    const actor = vtkActor.newInstance();
    actor.setMapper(mapper);

    // 获取渲染器和渲染窗口
    const renderer = fullScreenRenderer.getRenderer();
    const renderWindow = fullScreenRenderer.getRenderWindow();
    if (scalarBarVisible.value) {
      renderer.addActor(scalarBarActor);
    } else {
      renderer.removeActor(scalarBarActor);
    }
    const str = '0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.2 0.5 0.5 0.5 0.5 0.5'
    console.log(str.split(' ').length);

    // 将Actor添加到渲染器
    renderer.addActor(actor);
    // 将 ScalarBarActor 添加到渲染器中
    renderer.addActor(scalarBarActor);
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
