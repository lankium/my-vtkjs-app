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

const vtkContainer = ref(null);  // 存储VTK渲染区域的引用
const context = ref(null);  // 存储渲染上下文（包含渲染窗口、渲染器等）
const representation = ref(2);  // 初始表示方式（2：表面）

let scalarBarActor = vtkScalarBarActor.newInstance();

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

// 生成标量条刻度
function generateTicks(numberOfTicks) {
  return (helper) => {
    const lastTickBounds = helper.getLastTickBounds(); // 获取刻度范围
    const scale = d3.scaleLinear().domain([0.0, 1.0]).range([lastTickBounds[0], lastTickBounds[1]]); // 生成线性刻度
    const samples = scale.ticks(numberOfTicks); // 根据刻度数量生成样本
    const ticks = samples.map((tick) => scale(tick)); // 转换样本为刻度值

    // 格式化刻度值，替换 Unicode 减号
    formatDefaultLocale({ minus: '\u002D' });
    const format = scale.tickFormat(ticks[0], ticks[ticks.length - 1], numberOfTicks);
    const tickStrings = ticks.map(format).map((tick) =>
      Number(parseFloat(tick).toPrecision(12)).toPrecision()
    ); // 确保刻度字符串无多余空格
    helper.setTicks(ticks); // 设置刻度值
    helper.setTickStrings(tickStrings); // 设置刻度标签
  };
}

// 使用加载的VTP数据更新渲染器
function updateRendererWithVtp(polyData) {
  if (context.value) {
    const { actor, mapper, renderWindow, renderer } = context.value;
    const lut = mapper.getLookupTable();
    console.log('polyData', polyData);

    console.log('polyData.getCellData', polyData.getCellData());
    console.log('polyData.getCellData.getScalars()', polyData.getCellData().getScalars().getRange());
    // 获取标量数据并将其应用到模型
    const scalars = polyData.getCellData().getScalars();

    mapper.setColorModeToMapScalars();
    lut.setVectorModeToMagnitude();
    if (scalars) {
      // 获取密度数据的名称并应用到颜色映射
      console.log("Scalar data name:", scalars.getName());
      console.log("Scalar data range:", scalars.getRange());
      // 获取密度值的最小值和最大值
      const scalarRange = scalars.getRange();
      minScalarValue.value = scalarRange[0];
      maxScalarValue.value = scalarRange[1];

    }
    // 设置Mapper的输入数据为加载的VTP数据
    mapper.setInputData(polyData);

    lut.setRange(parseFloat(minScalarValue.value), parseFloat(maxScalarValue.value));
    scalarBarActor.setScalarsToColors(lut);
    // 设置 ScalarBar 的显示
    if (scalarBarVisible.value) {
      renderer.addActor(scalarBarActor);
    } else {
      renderer.removeActor(scalarBarActor);
    }

    // 根据当前选择的表示方式设置Actor的显示属性
    actor.getProperty().setRepresentation(representation.value);

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
    const lut = mapper.getLookupTable();
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
    const lut = mapper.getLookupTable();
    mapper.setUseLookupTableScalarRange(true);
    lut.setRange(lut.getRange()[0], parseFloat(maxScalarValue.value));
    renderWindow.render();
  }
});

// 监听 Scalar Bar 显示状态变化
watch(scalarBarVisible, () => {
  if (context.value) {
    const { mapper, renderer, renderWindow } = context.value;
    if (scalarBarVisible.value) {
      mapper.setScalarVisibility(true);
      renderer.addActor(scalarBarActor);

    } else {
      mapper.setScalarVisibility(false);
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
      // 设置最大密度 -> 红色
      ctf.addRGBPoint(1, 1.0, 0.0, 0.0);

      // 设置中间密度（假设为 0.5），设置为白色
      ctf.addRGBPoint(0.5, 1.0, 1.0, 1.0);  // 中间 -> 白色

      // 设置最小密度 -> 蓝色
      ctf.addRGBPoint(0, 0.0, 0.0, 1.0);

      // 应用颜色映射
      mapper.setLookupTable(ctf);
    } else {
      const numberOfColors = parseInt(numberOfColorsInput.value, 10);
      mapper.setLookupTable(vtkLookupTable.newInstance({ numberOfColors }));
    }

    const lut = mapper.getLookupTable();
    lut.setRange(parseFloat(minScalarValue.value), parseFloat(maxScalarValue.value));
    mapper.setColorModeToMapScalars();
    lut.setVectorModeToMagnitude();
    scalarBarActor.setScalarsToColors(lut);
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

    // 设置标量数据范围

    const lut = mapper.getLookupTable();
    lut.setRange(parseFloat(minScalarValue.value), parseFloat(maxScalarValue.value));

    scalarBarActor.setGenerateTicks(generateTicks(10));
    scalarBarActor.setScalarsToColors(lut);
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

    // 将Actor添加到渲染器
    renderer.addActor(actor);
    // 将 ScalarBarActor 添加到渲染器中
    renderer.addActor(scalarBarActor);
    // 重置相机并开始渲染
    renderer.resetCamera();
    // renderWindow.render();

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
