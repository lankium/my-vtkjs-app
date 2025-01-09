<template>
  <div>
    <div ref="vtkContainer" />
  </div>
</template>

<script setup>
import { ref } from 'vue';
// Load the rendering pieces we want to use (for both WebGL and WebGPU)
import '@kitware/vtk.js/Rendering/Profiles/Geometry';

import vtkActor from '@kitware/vtk.js/Rendering/Core/Actor';
import vtkFullScreenRenderWindow from '@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow';
import vtkMapper from '@kitware/vtk.js/Rendering/Core/Mapper';
import vtkPolyDataReader from '@kitware/vtk.js/IO/Legacy/PolyDataReader';
import vtkHttpDataSetReader from '@kitware/vtk.js/IO/Core/HttpDataSetReader';
import vtkHttpDataSetSeriesReader from '@kitware/vtk.js/IO/Core/HttpDataSetSeriesReader';
import XMLImageDataReader from '@kitware/vtk.js/IO/XML/XMLImageDataReader';  // 引入VTP文件读取器


const vtkContainer = ref(null);

const fileName = 'head-ascii.vti'; // 'uh60.vtk'; // 'luggaBody.vtk';

// ----------------------------------------------------------------------------
// Standard rendering code setup
// ----------------------------------------------------------------------------

const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance(
  {
    rootContainer: vtkContainer.value,
  }
);
const renderer = fullScreenRenderer.getRenderer();
const renderWindow = fullScreenRenderer.getRenderWindow();

const resetCamera = renderer.resetCamera;
const render = renderWindow.render;

// ----------------------------------------------------------------------------
// Example code
// ----------------------------------------------------------------------------

// const reader = vtkHttpDataSetReader.newInstance();
// const reader = vtkPolyDataReader.newInstance();
// const reader = vtkHttpDataSetSeriesReader.newInstance({ fetchGzip: true });
const reader = XMLImageDataReader.newInstance();


reader.setUrl(`../../public/VTK/${fileName}`).then(() => {
  const polydata = reader.getOutputData(0);
  const mapper = vtkMapper.newInstance();
  const actor = vtkActor.newInstance();

  actor.setMapper(mapper);
  mapper.setInputData(polydata);

  renderer.addActor(actor);

  resetCamera();
  render();
});
</script>

<style lang="scss" scoped></style>