<script setup lang="ts">
import { Camera } from '@mediapipe/camera_utils';
import { FaceMesh } from '@mediapipe/face_mesh';
import * as THREE from 'three';
import { computed, onMounted } from 'vue';

interface Props {
  lipstickMaterial: THREE.MeshBasicMaterial,
  componentSize: number,
}

const props = defineProps<Props>();
const lipstickMaterial = computed(() => props.lipstickMaterial);
const componentSize = props.componentSize ?? 640;
const lipsUpperOuter = [61, 185, 40, 39, 37, 0, 267, 269, 270, 409, 291, 306, 292, 308, 415, 310, 311, 312, 13, 82, 81, 80, 191, 78, 62, 76]; // Just an example set of points
const lipsLowerOuter = [61, 76, 62, 78, 95, 88, 178, 87, 14, 317, 402, 318, 324, 308, 292, 306, 291, 375, 321, 405, 314, 17, 84, 181, 91, 146]; // Just an example set of points

interface Landmark {
  x: number
  y: number
}
const convertLandmark = (landmark: Landmark) => {
  return {
    x: (landmark.x - 0.5) * 3,
    y: -(landmark.y - 0.5) * 3,
  }
};

onMounted(async () => {
  const calcCanvasElement = document.getElementById('calcCanvasElement') as HTMLCanvasElement;
  const renderCanvasElement = document.getElementById('renderCanvasElement') as HTMLCanvasElement;
  const videoElement = document.getElementById('videoElement') as HTMLVideoElement;
  if (!videoElement) return;

  const context = renderCanvasElement.getContext('2d');

  const stream = await navigator.mediaDevices.getUserMedia({
    video: {
      width: { ideal: componentSize },
      height: { ideal: componentSize }
    }
  });
  videoElement.srcObject = stream;
  videoElement.play();


  // シーンのセットアップ
  const scene = new THREE.Scene();
  const aspect = 1;
  const threeCamera = new THREE.PerspectiveCamera(75, aspect, 0.1, 1000);
  threeCamera.position.z = 2;

  const renderer = new THREE.WebGLRenderer({ alpha: true, canvas: calcCanvasElement });
  renderer.setSize(componentSize, componentSize);

  // MediaPipe の顔メッシュ検出をセットアップ
  const faceMesh = new FaceMesh({
    locateFile: (file) => `./node_modules/@mediapipe/face_mesh/${file}`
  });

  faceMesh.setOptions({
    maxNumFaces: 1,
    refineLandmarks: true,
    minDetectionConfidence: 0.7,
    minTrackingConfidence: 0.5
  });

  faceMesh.onResults(onResults);

  new Camera(videoElement, {
    onFrame: async () => {
      await faceMesh.send({ image: videoElement });
    },
    width: componentSize,
    height: componentSize
  }).start();

  // 毎回呼ぶ
  function onResults(results: any) {

    if (context) {
      context!.globalCompositeOperation = 'source-over';
      context!.drawImage(videoElement, 0, 0);
      context!.drawImage(calcCanvasElement, 0, 0);

      context!.globalCompositeOperation = 'soft-light';
      context!.fillStyle = 'rgba(255, 255, 255, 0.3)';
      context!.fillRect(0, 0, renderCanvasElement.width, renderCanvasElement.height);

      context!.globalCompositeOperation = 'color-dodge';
      context!.fillStyle = 'rgba(100, 100, 255, 0.08)';
      context!.fillRect(0, 0, renderCanvasElement.width, renderCanvasElement.height);
    }

    if (!results.multiFaceLandmarks || results.multiFaceLandmarks.length === 0) {
      return;
    }

    scene.children.forEach((child) => {
      scene.remove(child);
    });

    const landmarks = results.multiFaceLandmarks[0].map((landmark: Landmark) => convertLandmark(landmark));

    const shape = new THREE.Shape();
    shape.moveTo(
      landmarks[lipsLowerOuter[0]].x,
      landmarks[lipsLowerOuter[0]].y,
    );

    lipsUpperOuter.forEach(index => {
      const landmark = landmarks[index];
      shape.lineTo(landmark.x, landmark.y);
    });

    lipsLowerOuter.forEach(index => {
      const landmark = landmarks[index];
      shape.lineTo(landmark.x, landmark.y);
    });

    const geometry = new THREE.ShapeGeometry(shape);
    const mesh = new THREE.Mesh(geometry, lipstickMaterial.value);

    scene.add(mesh);

    threeCamera.lookAt(scene.position);
    renderer.render(scene, threeCamera);
  }

  function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, threeCamera);
  }

  animate();
});
</script>

<template>
  <div id="container">
    <video id="videoElement" playsinline autoplay muted
      style="display: none; width: componentSize; height: componentSize;" />
    <canvas id="calcCanvasElement" :width="componentSize" :height="componentSize" style="display: none;" />
    <canvas id="renderCanvasElement" :width="componentSize" :height="componentSize" />
  </div>
</template>

<style>
#container video,
#container canvas {
  position: absolute;
  top: 0;
  left: 0;
}
</style>