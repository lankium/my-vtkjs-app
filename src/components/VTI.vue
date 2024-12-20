<template>
  <div>
    <!-- 文件上传区域 -->
    <div>
      <input type="file" accept=".vti" @change="onFileChange" class="file-input" />
    </div>

    <!-- VTK 渲染区域 -->
    <div ref="vtkContainer" class="vtk-container"></div>

    <!-- 控制面板 -->
    <div ref="rootControllerContainer" class="root-controller" v-if="showController">
      <div class="control">
        <label for="window">Window:</label>
        <input v-model="windowLevel.window" id="window" type="number" @input="updateWindowLevel" />
        <label for="level">Level:</label>
        <input v-model="windowLevel.level" id="level" type="number" @input="updateWindowLevel" />
        <label for="interpolation">Linear interpolation:</label>
        <input type="checkbox" v-model="interpolation" @input="updateInterpolation" />
      </div>
    </div>

    <!-- FPS 显示 -->
    <div ref="fpsMonitor" class="fps-monitor" v-if="fpsEnabled"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import '@kitware/vtk.js/favicon';
import vtkFullScreenRenderWindow from '@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow';
import vtkImageMapper from '@kitware/vtk.js/Rendering/Core/ImageMapper';
import vtkImageSlice from '@kitware/vtk.js/Rendering/Core/ImageSlice';
import vtkXMLImageDataReader from '@kitware/vtk.js/IO/XML/XMLImageDataReader';
import vtkColorTransferFunction from '@kitware/vtk.js/Rendering/Core/ColorTransferFunction';
import vtkPiecewiseFunction from '@kitware/vtk.js/Common/DataModel/PiecewiseFunction';
import vtkFPSMonitor from '@kitware/vtk.js/Interaction/UI/FPSMonitor';
import vtkInteractorStyleImage from '@kitware/vtk.js/Interaction/Style/InteractorStyleImage';
import vtkColorMaps from '@kitware/vtk.js/Rendering/Core/ColorTransferFunction/ColorMaps';
import vtkBoundingBox from '@kitware/vtk.js/Common/DataModel/BoundingBox'; // 添加导入

const vtkContainer = ref(null);
const rootControllerContainer = ref(null);
const fpsMonitor = ref(null);

const userParams = new URLSearchParams(window.location.search);
let autoInit = true;
let background = [0, 0, 0];
const lutName = userParams.get('lut') || 'Grayscale';
const lookupTable = vtkColorTransferFunction.newInstance();

const windowLevel = ref({
  window: 256,
  level: 128
});
const interpolation = ref(true);
const showController = ref(true);
const fpsEnabled = ref(true);

// 初始化 pipeline 状态
const pipeline = ref({
  actor: null,
  renderer: null,
  renderWindow: null,
  lookupTable: lookupTable,
  mapper: null,
  source: null,
  piecewiseFunction: vtkPiecewiseFunction.newInstance(),
  fullScreenRenderer: null,
  fps: null,
});


// Initialize the viewer when mounted
onMounted(() => {
  if (!autoInit) return;

  const container = vtkContainer.value;
  if (!container) return;

  // 监听文件上传
  const fileInput = document.createElement('input');
  fileInput.type = 'file';
  fileInput.accept = '.vti';
  fileInput.addEventListener('change', onFileChange);

  // 自动触发文件选择（或通过UI提供的按钮手动选择）
  document.body.appendChild(fileInput);
  fileInput.click();
});

onBeforeUnmount(() => {
  // Clean up the resources when component is unmounted
  if (vtkContainer.value) {
    vtkContainer.value.innerHTML = '';
  }
});

// 处理文件上传的函数
function onFileChange(event) {
  const file = event.target.files[0];

  if (file && file.name.endsWith('.vti')) {
    const reader = new FileReader();
    reader.onload = function (e) {
      loadViewer(vtkContainer.value, e.target.result);
    };
    reader.readAsArrayBuffer(file);
  } else {
    alert('请上传一个有效的 .vti 文件');
  }
}

// Function to handle window and level changes
function updateWindowLevel() {
  const { window, level } = windowLevel.value;
  pipeline.value.actor.getProperty().setColorWindow(window);
  pipeline.value.actor.getProperty().setColorLevel(level);
  pipeline.value.lookupTable.setMappingRange(level - window * 0.5, level + window * 0.5);
  pipeline.value.lookupTable.updateRange();
  pipeline.value.renderWindow.render();
}

// Function to handle interpolation change
function updateInterpolation() {
  const imageSlice = pipeline.value.actor;
  if (interpolation.value) {
    imageSlice.getProperty().setInterpolationTypeToLinear();
  } else {
    imageSlice.getProperty().setInterpolationTypeToNearest();
  }
  pipeline.value.renderWindow.render();
}

// Function to handle the viewer setup
function loadViewer(container, fileContents) {
  const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance({
    rootContainer: container,
    background,
  });

  const renderer = fullScreenRenderer.getRenderer();
  const renderWindow = fullScreenRenderer.getRenderWindow();
  const interactor = renderWindow.getInteractor();

  const vtiReader = vtkXMLImageDataReader.newInstance();
  vtiReader.parseAsArrayBuffer(fileContents);
  const source = vtiReader.getOutputData(0);
  console.log(source.toJSON());

  const mapper = vtkImageMapper.newInstance();
  console.log(mapper.toJSON());

  const actor = vtkImageSlice.newInstance();

  mapper.setInputData(source);
  mapper.setSliceAtFocalPoint(true);
  mapper.setSlicingMode(2); // Z slice
  actor.setMapper(mapper);
  renderer.addActor(actor);

  // Set up the camera
  const bounds = source.getBounds();
  const camera = renderer.getActiveCamera();
  camera.setFocalPoint(...vtkBoundingBox.getCenter(bounds));  // 这里使用 vtkBoundingBox 获取中心点
  camera.setPosition(...vtkBoundingBox.getCenter(bounds));
  camera.setParallelProjection(true);

  actor.getProperty().setRGBTransferFunction(0, lookupTable);
  actor.getProperty().setScalarOpacity(0, vtkPiecewiseFunction.newInstance());

  // Add FPS monitor
  const fps = vtkFPSMonitor.newInstance();
  fps.setRenderWindow(renderWindow);
  fps.setContainer(fpsMonitor.value);
  fps.update();

  pipeline.value = {
    actor,
    renderer,
    renderWindow,
    lookupTable,
    mapper,
    source,
    fullScreenRenderer,
    fps,
  };

  renderWindow.render();
}
</script>

<style scoped>
.vtk-container {
  width: 100%;
  height: 100%;
}

.root-controller {
  position: absolute;
  top: 25px;
  left: 25px;
  background: white;
  padding: 12px;
}

.control {
  margin-bottom: 10px;
}

.fps-monitor {
  position: fixed;
  top: 10px;
  right: 10px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 5px;
}

.file-input {
  margin: 30px;
}
</style>
