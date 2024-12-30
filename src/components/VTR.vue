<template>
  <div id="app">
    <h1>VTR 转换为 VTP</h1>
    <input type="file" @change="handleFileUpload" accept=".vtr" />
    <button @click="convertToVTP" :disabled="!vtrFile">转换并下载 VTP</button>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import vtkXMLRectilinearGridReader from '@kitware/vtk.js/IO/XML/XMLReader';
import vtkXMLPolyDataWriter from '@kitware/vtk.js/IO/XML/XMLPolyDataWriter';
import vtkPolyData from '@kitware/vtk.js/Common/DataModel/PolyData';
import vtkPoints from '@kitware/vtk.js/Common/Core/Points';

const vtrFile = ref(null);

// 文件上传处理
const handleFileUpload = (event) => {
  vtrFile.value = event.target.files[0];
};

// 转换并下载 VTP 文件
const convertToVTP = async () => {
  if (!vtrFile.value) return;
  console.log(vtkXMLRectilinearGridReader);

  const reader = vtkXMLRectilinearGridReader.newInstance();

  const arrayBuffer = await vtrFile.value.arrayBuffer();
  reader.parseAsArrayBuffer(arrayBuffer);

  // 获取 Rectilinear Grid 数据
  const rectilinearGrid = reader.getOutputData();

  // 提取网格坐标
  const xCoords = rectilinearGrid.getXCoordinates().getData();
  const yCoords = rectilinearGrid.getYCoordinates().getData();
  const zCoords = rectilinearGrid.getZCoordinates().getData();

  // 生成 vtkPolyData
  const polyData = vtkPolyData.newInstance();
  const points = vtkPoints.newInstance();

  // 构建点数据
  for (let k = 0; k < zCoords.length; k++) {
    for (let j = 0; j < yCoords.length; j++) {
      for (let i = 0; i < xCoords.length; i++) {
        points.insertNextPoint(xCoords[i], yCoords[j], zCoords[k]);
      }
    }
  }
  polyData.setPoints(points);

  // 添加标量数据（如果有）
  const pointData = rectilinearGrid.getPointData();
  const scalars = pointData.getScalars();
  if (scalars) {
    polyData.getPointData().setScalars(scalars);
  }

  // 导出 VTP 文件
  const writer = vtkXMLPolyDataWriter.newInstance();
  writer.setInputData(polyData);

  const vtpArrayBuffer = writer.writeArrayBuffer();

  // 创建 Blob 并下载
  const blob = new Blob([vtpArrayBuffer], { type: 'application/octet-stream' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = `${vtrFile.value.name.split('.')[0]}.vtp`;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);

  alert('转换成功！');
};

</script>

<style>
#app {
  font-family: Arial, sans-serif;
  text-align: center;
  margin-top: 50px;
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
</style>
