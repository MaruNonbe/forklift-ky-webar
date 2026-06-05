import * as THREE from "three";
import { ARButton } from "three/addons/webxr/ARButton.js";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

let camera;
let scene;
let renderer;
let controller;

let reticle;
let hitTestSource = null;
let hitTestSourceRequested = false;

let loadedModel = null;
let placedModel = null;

let mixer = null;
let clock;
let animationActions = [];
let currentAction = null;

let modelPlaced = false;
let modelBaseScale = 1.0;

const MODEL_PATH = "./models/forklift_ky.glb";

const statusEl = document.getElementById("status");
const btnPlay = document.getElementById("btnPlay");
const btnPause = document.getElementById("btnPause");
const btnReset = document.getElementById("btnReset");
const btnScaleUp = document.getElementById("btnScaleUp");
const btnScaleDown = document.getElementById("btnScaleDown");
const btnRotate = document.getElementById("btnRotate");

init();

function init() {
  clock = new THREE.Clock();

  scene = new THREE.Scene();

  camera = new THREE.PerspectiveCamera(
    70,
    window.innerWidth / window.innerHeight,
    0.01,
    40
  );

  const ambientLight = new THREE.HemisphereLight(0xffffff, 0xbbbbff, 2.2);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 2.0);
  directionalLight.position.set(1, 3, 2);
  scene.add(directionalLight);

  renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true
  });

  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.xr.enabled = true;

  document.getElementById("app").appendChild(renderer.domElement);

  const arButton = ARButton.createButton(renderer, {
    requiredFeatures: ["hit-test"],
    optionalFeatures: ["dom-overlay"],
    domOverlay: { root: document.body }
  });

  arButton.style.bottom = "172px";
  arButton.style.borderRadius = "999px";
  arButton.style.background = "#f97316";
  arButton.style.fontWeight = "800";
  arButton.style.boxShadow = "0 6px 18px rgba(0,0,0,0.35)";
  document.body.appendChild(arButton);

  controller = renderer.xr.getController(0);
  controller.addEventListener("select", onSelect);
  scene.add(controller);

  createReticle();
  loadModel();

  renderer.setAnimationLoop(render);
  window.addEventListener("resize", onWindowResize);

  btnPlay.addEventListener("click", playAnimation);
  btnPause.addEventListener("click", pauseAnimation);
  btnReset.addEventListener("click", resetAnimation);
  btnScaleUp.addEventListener("click", scaleUpModel);
  btnScaleDown.addEventListener("click", scaleDownModel);
  btnRotate.addEventListener("click", rotateModel);

  updateStatus("GLBモデルを読み込み中です。");
}

function createReticle() {
  reticle = new THREE.Mesh(
    new THREE.RingGeometry(0.12, 0.16, 32).rotateX(-Math.PI / 2),
    new THREE.MeshBasicMaterial({
      color: 0x00ff88
    })
  );

  reticle.matrixAutoUpdate = false;
  reticle.visible = false;
  scene.add(reticle);
}

function loadModel() {
  const loader = new GLTFLoader();

  loader.load(
    MODEL_PATH,
    (gltf) => {
      loadedModel = gltf.scene;

      loadedModel.traverse((child) => {
        if (child.isMesh) {
          child.frustumCulled = false;

          if (child.material) {
            child.material.side = THREE.DoubleSide;
          }
        }
      });

      normalizeModel(loadedModel);

      if (gltf.animations && gltf.animations.length > 0) {
        mixer = new THREE.AnimationMixer(loadedModel);

        animationActions = gltf.animations.map((clip) => {
          const action = mixer.clipAction(clip);
          action.clampWhenFinished = true;
          action.loop = THREE.LoopRepeat;
          return action;
        });

        currentAction = animationActions[0];

        updateStatus(
          `モデルを読み込みました。アニメーション数：${gltf.animations.length}。ARを開始して、画面をタップして配置してください。`
        );
      } else {
        updateStatus(
          "モデルを読み込みました。GLB内にアニメーションは見つかりませんでした。ARを開始して、画面をタップして配置してください。"
        );
      }
    },
    (xhr) => {
      if (xhr.total > 0) {
        const percent = Math.round((xhr.loaded / xhr.total) * 100);
        updateStatus(`GLBモデルを読み込み中です。${percent}%`);
      }
    },
    (error) => {
      console.error("GLB load error:", error);
      updateStatus(
        "GLBモデルの読み込みに失敗しました。models/forklift_ky.glb の場所とファイル名を確認してください。"
      );
    }
  );
}

function normalizeModel(model) {
  const box = new THREE.Box3().setFromObject(model);
  const size = new THREE.Vector3();
  const center = new THREE.Vector3();

  box.getSize(size);
  box.getCenter(center);

  model.position.sub(center);

  const maxDimension = Math.max(size.x, size.y, size.z);

  if (maxDimension > 0) {
    const targetSize = 1.6;
    modelBaseScale = targetSize / maxDimension;
    model.scale.setScalar(modelBaseScale);
  }

  model.rotation.y = 0;
}

function onSelect() {
  if (!reticle.visible || !loadedModel) {
    updateStatus("床面が見つかっていません。スマホをゆっくり左右に動かしてください。");
    return;
  }

  if (!modelPlaced) {
    placedModel = loadedModel;
    placedModel.position.setFromMatrixPosition(reticle.matrix);
    placedModel.quaternion.setFromRotationMatrix(reticle.matrix);

    scene.add(placedModel);
    modelPlaced = true;

    updateStatus(
      "モデルを配置しました。危険エリアや作業者の位置を確認してください。アニメ再生ボタンで動きを確認できます。"
    );
  } else if (placedModel) {
    placedModel.position.setFromMatrixPosition(reticle.matrix);
    placedModel.quaternion.setFromRotationMatrix(reticle.matrix);

    updateStatus("モデル位置を再配置しました。");
  }
}

function playAnimation() {
  if (!currentAction) {
    updateStatus("このGLBには再生できるアニメーションがありません。BlenderのGLB書き出し設定を確認してください。");
    return;
  }

  currentAction.paused = false;
  currentAction.play();

  updateStatus("アニメーションを再生しています。フォークリフト後退と作業者の動きを確認してください。");
}

function pauseAnimation() {
  if (!currentAction) {
    updateStatus("一時停止できるアニメーションがありません。");
    return;
  }

  currentAction.paused = true;
  updateStatus("アニメーションを一時停止しました。");
}

function resetAnimation() {
  if (!currentAction) {
    updateStatus("リセットできるアニメーションがありません。");
    return;
  }

  currentAction.stop();
  currentAction.reset();
  currentAction.paused = true;

  if (mixer) {
    mixer.setTime(0);
  }

  updateStatus("アニメーションを最初の状態に戻しました。");
}

function scaleUpModel() {
  if (!placedModel) {
    updateStatus("先にAR空間へモデルを配置してください。");
    return;
  }

  placedModel.scale.multiplyScalar(1.15);
  updateStatus("モデルを拡大しました。");
}

function scaleDownModel() {
  if (!placedModel) {
    updateStatus("先にAR空間へモデルを配置してください。");
    return;
  }

  placedModel.scale.multiplyScalar(0.85);
  updateStatus("モデルを縮小しました。");
}

function rotateModel() {
  if (!placedModel) {
    updateStatus("先にAR空間へモデルを配置してください。");
    return;
  }

  placedModel.rotation.y += THREE.MathUtils.degToRad(30);
  updateStatus("モデルを30度回転しました。");
}

function render(timestamp, frame) {
  const delta = clock.getDelta();

  if (mixer) {
    mixer.update(delta);
  }

  if (frame) {
    const referenceSpace = renderer.xr.getReferenceSpace();
    const session = renderer.xr.getSession();

    if (!hitTestSourceRequested) {
      session.requestReferenceSpace("viewer").then((viewerSpace) => {
        session.requestHitTestSource({ space: viewerSpace }).then((source) => {
          hitTestSource = source;
        });
      });

      session.addEventListener("end", () => {
        hitTestSourceRequested = false;
        hitTestSource = null;
        reticle.visible = false;
        modelPlaced = false;
        placedModel = null;
        updateStatus("ARセッションを終了しました。");
      });

      hitTestSourceRequested = true;
    }

    if (hitTestSource) {
      const hitTestResults = frame.getHitTestResults(hitTestSource);

      if (hitTestResults.length) {
        const hit = hitTestResults[0];
        const pose = hit.getPose(referenceSpace);

        reticle.visible = true;
        reticle.matrix.fromArray(pose.transform.matrix);

        if (!modelPlaced) {
          updateStatus("床面を検出しました。画面をタップしてモデルを配置してください。");
        }
      } else {
        reticle.visible = false;

        if (!modelPlaced) {
          updateStatus("床面を探しています。スマホをゆっくり動かしてください。");
        }
      }
    }
  }

  renderer.render(scene, camera);
}

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();

  renderer.setSize(window.innerWidth, window.innerHeight);
}

function updateStatus(text) {
  if (statusEl) {
    statusEl.textContent = text;
  }
}