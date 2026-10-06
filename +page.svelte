<script lang="ts">
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
  


  // --- Constants ---
  const REF_HEAD_COLORS = [
    0x800000, // 0: SE, maroon
    0xff8800, // 1 E, orange
    0x333333, // 2: NW, grey
    0x00ff00, // 3 N, green
    0x00ffff, // 4: NR, cyan
    0x0000ff, // 5 W, blue
    0xff00ff, // 6: SW, magenta
    0x8800ff, // 7 S, purple
    0x111111  // 8: 
  ];

  // --- Configuration ---
  let canvas: HTMLCanvasElement;
  let mocapCanvas: HTMLCanvasElement; 
  let fileInput: HTMLInputElement; 
  let imagePoseInput: HTMLInputElement; 
  let videoPoseInput: HTMLInputElement; 
  let videoElement: HTMLVideoElement; 
  
  let renderWidth = 512;
  let renderHeight = 512;
  
  // --- Three.js State ---
  let scene: THREE.Scene;
  let camera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let orbitControl: OrbitControls;
  let skeletonHelper: THREE.SkeletonHelper;
  let ambientLight: THREE.AmbientLight;
  let dirLight: THREE.DirectionalLight;
  let viewportNoLighting = false;
  let originalMaterials: Map<THREE.Mesh, THREE.Material | THREE.Material[]> = new Map();
  
  // --- Interaction State ---
  let raycaster = new THREE.Raycaster();
  let pointer = new THREE.Vector2();
  let isDraggingBone = false;
  let lastPointerX = 0;
  let lastPointerY = 0;
  let rotationSpeed = 0.015;
  let moveSpeed = 0.01;
  let isShiftDown = false;
  
  type ToolMode = 'rotate' | 'twist' | 'move';
  let currentTool: ToolMode = 'rotate';

  // --- Animation State ---
  let bones: THREE.Bone[] = [];
  let selectedBoneIndex = -1;
  let selectedBoneName: string | null = null;
  let keyframes: any[] = [];
  let activeFrameIndex = -1; 
  let undoStack: any[] = [];
  
  let draggingFrameIndex: number | null = null;
  let dragOverIndex: number | null = null;
  
  let isPlaying = false;
  let playbackSpeed = 4.0;
  let animationTime = 0;
  let clock = new THREE.Clock();

  // --- Mocap State ---
  let isWebcamRunning = false;
  let isMirrored = true; 

  let lastMocapResults: any = null; 
  
  let legAmplitude = 0.7; 
  let armAmplitude = 1.0; 
  let depthAmplitude = 1.0; 
  let armForwardOffset = 0.0; 
  
  let headTwist = 0; // Degrees
  
  let isProcessingVideo = false;
  let processingStatus = "";
  let showMocapDebug = false;

  // --- Bone Color State ---
  let boneColors: Record<string, string> = {
      Hips: '#8A2BE2',
      Torso: '#4169E1',
      TorsoLower: '#2E5090',
      Neck: '#FFFF00',
      Head: '#FFD700',
      UpperArm_L: '#DC143C',
      LowerArm_L: '#FF6347',
      Hand_L: '#FFA07A',
      UpperArm_R: '#006400',
      LowerArm_R: '#32CD32',
      Hand_R: '#98FB98',
      Thigh_L: '#008B8B',
      Shin_L: '#00CED1',
      Foot_L: '#AAFFEE',
      Thigh_R: '#FF8C00',
      Shin_R: '#FFA500',
      Foot_R: '#FFDAB9'
  };

  // --- Export State ---
  let exportCanvas: HTMLCanvasElement;
  let isExporting = false; 
  
  // Export Modal State
  let showExportModal = false;
  let exportSettings = {
      width: 128,
      height: 128,
      layout: 'grid' as 'grid' | 'horizontal' | 'vertical',
    flatShading: false,
    normalizeHeadColor: false,
    normalizeBackground: false,
    useBoundingBox: false,
    boundingBoxWidth: 100,
    boundingBoxHeight: 300,
    boundingBoxOffsetX: 0,
    boundingBoxOffsetY: -30,
    transparentBackground: false
  };

  // --- Shape Segment System ---
  type ShapeCategory = 'Head' | 'Torso' | 'Hips' | 'Arms' | 'Legs' | 'Feet';
  
  let shapeState: Record<ShapeCategory, number> = {
      Head: 0,
      Torso: 5,
      Hips: 2,
      Arms: 2,
      Legs: 0,
      Feet: 2
  };

  const SHAPE_REGISTRY = {
      Head: [
          { name: '8-Sided', gen: () => { 
              const g = new THREE.CylinderGeometry(0.15, 0.15, 0.25, 8, 1, false); 
              g.rotateY(Math.PI / 8); 
              g.translate(0, 0.125, 0); 
              g.clearGroups();
              for(let i=0; i<8; i++) g.addGroup(i * 6, 6, i);
              const capStartIndex = 48;
              const capCount = g.index ? (g.index.count - capStartIndex) : 0;
              g.addGroup(capStartIndex, capCount, 8); 
              return g; 
          }},
          { name: 'Box', gen: () => { const g = new THREE.BoxGeometry(0.25, 0.25, 0.25); g.translate(0, 0.125, 0); return g; } },
          { name: 'Round', gen: () => { const g = new THREE.SphereGeometry(0.14, 16, 16); g.translate(0, 0.14, 0); return g; } }
      ],
      Torso: [
          { name: 'Box', gen: () => { const g = new THREE.BoxGeometry(0.4, 0.5, 0.25); g.translate(0, 0.25, 0); return g; } },
          { name: 'Round', gen: () => { 
              const g = new THREE.CapsuleGeometry(0.2, 0.25, 4, 8); 
              g.translate(0, 0.325, 0); 
              return g; 
          }},
          { name: 'Broad', gen: () => { 
              const points = [
                  new THREE.Vector2(0.20, 0),      // waist (matches pelvis)
                  new THREE.Vector2(0.22, 0.15),   // mid torso
                  new THREE.Vector2(0.28, 0.35),   // chest
                  new THREE.Vector2(0.26, 0.45),   // shoulders curve
                  new THREE.Vector2(0.18, 0.52),   // top of shoulders (rounded)
              ];
              const g = new THREE.LatheGeometry(points, 12); 
              g.scale(1, 1, 0.55);
              g.translate(0, 0, 0); 
              return g; 
          }},
          { name: 'Bezel', gen: () => {
              const points = [];
              points.push(new THREE.Vector2(0.2, 0)); // bottom right
              points.push(new THREE.Vector2(0.2, 0.3)); // shoulder right
              // semi-circle
              for (let i = 0; i <= 8; i++) {
                  const theta = Math.PI * i / 8;
                  const x = 0.2 * Math.cos(theta);
                  const y = 0.3 + 0.2 * Math.sin(theta);
                  points.push(new THREE.Vector2(x, y));
              }
              points.push(new THREE.Vector2(-0.2, 0.3)); // shoulder left
              points.push(new THREE.Vector2(-0.2, 0)); // bottom left
              const g = new THREE.LatheGeometry(points, 12);
              g.scale(1, 1, 0.625);
              g.translate(0, 0.0, 0);
              return g;
          }},
          { name: 'Bezel Curved', gen: () => {
              const points = [];
              points.push(new THREE.Vector2(0.2, 0));
              points.push(new THREE.Vector2(0.21, 0.075));
              points.push(new THREE.Vector2(0.22, 0.15));
              points.push(new THREE.Vector2(0.21, 0.225));
              points.push(new THREE.Vector2(0.2, 0.3));
              // semi-circle
              for (let i = 0; i <= 8; i++) {
                  const theta = Math.PI * i / 8;
                  const x = 0.2 * Math.cos(theta);
                  const y = 0.3 + 0.2 * Math.sin(theta);
                  points.push(new THREE.Vector2(x, y));
              }
              points.push(new THREE.Vector2(-0.2, 0.3));
              points.push(new THREE.Vector2(-0.21, 0.225));
              points.push(new THREE.Vector2(-0.22, 0.15));
              points.push(new THREE.Vector2(-0.21, 0.075));
              points.push(new THREE.Vector2(-0.2, 0));
              const g = new THREE.LatheGeometry(points, 12);
              g.scale(1, 1, 0.625);
              g.translate(0, 0.0, 0);
              return g;
          }},
          { name: 'Box + Top Cylinder', gen: () => { const g = new THREE.BoxGeometry(0.4, 0.38, 0.25); g.translate(0, 0.16, 0); return g; } }
      ],
      Hips: [
          { name: 'Box', gen: () => { const g = new THREE.BoxGeometry(0.35, 0.2, 0.25); g.translate(0, 0.1, 0); return g; } },
          { name: 'Round', gen: () => { 
              const g = new THREE.CapsuleGeometry(0.18, 0.05, 4, 8); 
              g.rotateX(Math.PI / 2);
              g.scale(1, 1, 0.7);
              g.translate(0, 0.1, 0); 
              return g; 
          }},
          { name: 'Box + Round', gen: () => { const g = new THREE.BoxGeometry(0.35, 0.2, 0.23); g.translate(0, 0.1, 0); return g; } },
          { name: 'Triangle', gen: () => { 
              const points = [
                  new THREE.Vector2(0.03, 0),      // bottom point (narrow)
                  new THREE.Vector2(0.18, 0.16),   // widen out
                  new THREE.Vector2(0.20, 0.32),   // top (widest)
              ];
              const g = new THREE.LatheGeometry(points, 8); 
              g.scale(1, 1, 0.75);
              g.translate(0, -0.15, 0); 
              return g; 
          }}
      ],
      Arms: [
          { name: 'Box', gen: (w:number, h:number, d:number) => { const g = new THREE.BoxGeometry(w, h, d); g.translate(0, -h/2, 0); return g; } },
          { name: 'Round', gen: (w:number, h:number, d:number) => { const g = new THREE.CylinderGeometry(w/1.5, w/2, h, 8); g.translate(0, -h/2, 0); return g; } },
          { name: 'Box + Top Cylinder', gen: (w:number, h:number, d:number) => { const g = new THREE.BoxGeometry(w, h, d); g.translate(0, -h/2, 0); return g; } }
      ],
      Legs: [
          { name: 'Box', gen: (w:number, h:number, d:number) => { const g = new THREE.BoxGeometry(w, h, d); g.translate(0, -h/2, 0); return g; } },
          { name: 'Round', gen: (w:number, h:number, d:number) => { const g = new THREE.CylinderGeometry(w/1.5, w/2, h, 8); g.translate(0, -h/2, 0); return g; } }
      ],
      Feet: [
          { name: 'Box', gen: () => { const g = new THREE.BoxGeometry(0.15, 0.1, 0.2); g.translate(0, -0.05, 0.05); return g; } },
          { name: 'Big Boot', gen: () => { const g = new THREE.BoxGeometry(0.25, 0.15, 0.35); g.translate(0, -0.075, 0.05); return g; } },
          { name: 'Long Boot', gen: () => { const g = new THREE.BoxGeometry(0.15, 0.1, 0.25); g.translate(0, -0.05, 0.0); return g; } }
      ]
  };

  const MP_CHAINS = [
    { name: "UpperArm_L", start: 11, end: 13, isLeg: false }, 
    { name: "LowerArm_L", start: 13, end: 15, isLeg: false }, 
    { name: "UpperArm_R", start: 12, end: 14, isLeg: false }, 
    { name: "LowerArm_R", start: 14, end: 16, isLeg: false }, 
    { name: "Thigh_L",    start: 23, end: 25, isLeg: true }, 
    { name: "Shin_L",     start: 25, end: 27, isLeg: true }, 
    { name: "Thigh_R",    start: 24, end: 26, isLeg: true }, 
    { name: "Shin_R",     start: 26, end: 28, isLeg: true }, 
  ];
  
  const T_POSE_VECTOR = new THREE.Vector3(0, -1, 0); 
  const tempVector = new THREE.Vector3();
  const tempQuaternion = new THREE.Quaternion();
  const identityQuaternion = new THREE.Quaternion();

  // Toggle shadow casting based on viewportNoLighting setting
  function toggleViewportMaterials(noLighting: boolean) {
    // Disable the directional light to remove angled lighting/shading
    if (dirLight) dirLight.visible = !noLighting;
    
    bones.forEach(bone => {
      const mesh = bone.children.find(c => c instanceof THREE.Mesh) as THREE.Mesh;
      if (mesh) {
        // Toggle both shadow casting and receiving for all meshes
        mesh.castShadow = !noLighting;
        mesh.receiveShadow = !noLighting;
      }
    });
  }

  $: if (bones.length > 0) {
    toggleViewportMaterials(viewportNoLighting);
  }

  onMount(() => {
    initThree();
    createHardcodedHumanoid();
    animate();

    window.addEventListener('resize', onResize);
    window.addEventListener('keydown', onKeyDown);
    window.addEventListener('keyup', onKeyUp);
    
    return () => {
      window.removeEventListener('resize', onResize);
      window.removeEventListener('keydown', onKeyDown);
      window.removeEventListener('keyup', onKeyUp);
      renderer.dispose();
    };
  });

  function initThree() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x2a2a2a); 

    camera = new THREE.PerspectiveCamera(45, renderWidth / renderHeight, 0.1, 1000);
    camera.position.set(4, 3, 6);
    camera.lookAt(0, 1, 0);

    renderer = new THREE.WebGLRenderer({ canvas, antialias: false, preserveDrawingBuffer: true, alpha: true });
    renderer.setSize(renderWidth, renderHeight, false);
    renderer.setPixelRatio(1);
    renderer.shadowMap.enabled = true;

    ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);
    dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
    dirLight.position.set(5, 10, 7);
    dirLight.castShadow = true;
    scene.add(dirLight);

    const grid = new THREE.GridHelper(10, 10);
    scene.add(grid);

    orbitControl = new OrbitControls(camera, renderer.domElement);
    orbitControl.enableDamping = true;
    orbitControl.dampingFactor = 0.05;
  }

  // --- Segment Logic ---

  function cycleShape(part: ShapeCategory, direction: number) {
      const options = SHAPE_REGISTRY[part];
      let newIndex = shapeState[part] + direction;
      if (newIndex < 0) newIndex = options.length - 1;
      if (newIndex >= options.length) newIndex = 0;
      shapeState[part] = newIndex;
      updateMeshesForPart(part);
  }

  function selectBoneByName(boneName: string) {
      const boneIndex = bones.findIndex(b => b.name === boneName);
      if (boneIndex !== -1) {
          selectedBoneIndex = boneIndex;
          selectedBoneName = boneName;
      }
  }

  function changeBoneColor(boneName: string, hexColor: string) {
      boneColors[boneName] = hexColor;
      
      // Helper to change material color, skipping decorative elements
      const changeMatColor = (mat: THREE.Material) => {
          if (mat instanceof THREE.MeshStandardMaterial || mat instanceof THREE.MeshBasicMaterial) {
              mat.color.setStyle(hexColor);
          }
      };
      
      // Helper to change a mesh's color
      const changeMeshColor = (mesh: THREE.Mesh, skipDecorative: boolean = true) => {
          // Always skip decorative elements (yellow triangle, etc)
          if (mesh.userData && mesh.userData.isTorsoTriangle) {
              return;
          }
          if (Array.isArray(mesh.material)) {
              (mesh.material as THREE.Material[]).forEach(changeMatColor);
          } else {
              changeMatColor(mesh.material as THREE.Material);
          }
      };
      
      // Handle special case where boneName is like "TorsoLower" - find the mesh in the Torso bone
      if (boneName === 'TorsoLower') {
          const torsoBone = bones.find(b => b.name === 'Torso');
          if (torsoBone) {
              // Find meshes that are NOT the first main mesh (skip index 0)
              torsoBone.children.forEach((child, index) => {
                  if (index > 0 && child instanceof THREE.Mesh) {
                      changeMeshColor(child);
                      // Also change direct mesh children (cylinders, capsules, etc.)
                      child.children.forEach(subchild => {
                          if (subchild instanceof THREE.Mesh) {
                              changeMeshColor(subchild, true);
                          }
                      });
                  }
              });
          }
      } else {
          const bone = bones.find(b => b.name === boneName);
          if (bone) {
              // Change all direct mesh children and their mesh children
              bone.children.forEach((child) => {
                  if (child instanceof THREE.Mesh) {
                      changeMeshColor(child);
                      // Also change direct mesh children (cylinders, capsules, etc.)
                      child.children.forEach(subchild => {
                          if (subchild instanceof THREE.Mesh) {
                              changeMeshColor(subchild, true);
                          }
                      });
                  }
              });
          }
      }
  }

  const updateArmOutlines = (mesh: THREE.Mesh, shapeName: string) => {
      // Remove existing arm outlines
      for (let i = mesh.children.length - 1; i >= 0; i--) {
          const c: any = mesh.children[i];
          if (c && c.userData && c.userData.isArmOutline) {
              mesh.remove(c);
              if (c.geometry) c.geometry.dispose();
              if (c.material && c.material.dispose) c.material.dispose();
          }
      }

      // Add vertical line outline for Box + Top Cylinder variant
      if (shapeName === 'Box + Top Cylinder') {
          try {
              const geometry = mesh.geometry as THREE.BoxGeometry;
              geometry.computeBoundingBox();
              const bbox = geometry.boundingBox!;
              
              const width = bbox.max.x - bbox.min.x;
              const height = bbox.max.y - bbox.min.y;
              const depth = bbox.max.z - bbox.min.z;
              const centerX = (bbox.max.x + bbox.min.x) / 2;
              const centerY = (bbox.max.y + bbox.min.y) / 2;
              const centerZ = (bbox.max.z + bbox.min.z) / 2;

              // Create 4 vertical lines at the corners of the box
              const verticalLines = new THREE.BufferGeometry();
              const linePositions: number[] = [];

              // Corner positions for vertical edges
              const corners = [
                  [centerX - width/2, centerZ - depth/2],  // front-left
                  [centerX + width/2, centerZ - depth/2],  // front-right
                  [centerX - width/2, centerZ + depth/2],  // back-left
                  [centerX + width/2, centerZ + depth/2]   // back-right
              ];

              corners.forEach(([x, z]) => {
                  linePositions.push(x, bbox.min.y, z);
                  linePositions.push(x, bbox.max.y, z);
              });

              if (linePositions.length > 0) {
                  verticalLines.setAttribute('position', new THREE.BufferAttribute(new Float32Array(linePositions), 3));
                  const edgesMaterial = new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 2 });
                  const wireframe = new THREE.LineSegments(verticalLines, edgesMaterial);
                  wireframe.userData.isArmOutline = true;
                  mesh.add(wireframe);
              }
          } catch (e) {
              console.warn('Failed to add arm outlines:', e);
          }
      }
  };

  function updateMeshesForPart(part: ShapeCategory) {
      const index = shapeState[part];
      const generatorObj = SHAPE_REGISTRY[part][index];
      
      let targets: { name: string, dims?: number[], defaultColor?: number }[] = [];

      if (part === 'Head') targets = [{ name: 'Head', defaultColor: 0xFFD700 }];
      if (part === 'Torso') targets = [{ name: 'Torso', defaultColor: 0x4169E1 }];
      if (part === 'Hips') targets = [{ name: 'Hips', defaultColor: 0x8A2BE2 }];
      if (part === 'Feet') targets = [{ name: 'Foot_L', defaultColor: 0xAFEEEE }, { name: 'Foot_R', defaultColor: 0xFFDAB9 }];
      
      if (part === 'Arms') {
          targets = [
              { name: 'UpperArm_L', dims: [0.12, 0.4, 0.12], defaultColor: 0xDC143C },
              { name: 'LowerArm_L', dims: [0.10, 0.35, 0.10], defaultColor: 0xFF6347 },
              { name: 'UpperArm_R', dims: [0.12, 0.4, 0.12], defaultColor: 0x006400 },
              { name: 'LowerArm_R', dims: [0.10, 0.35, 0.10], defaultColor: 0x32CD32 },
              { name: 'Hand_L', dims: [0.10, 0.10, 0.10], defaultColor: 0xFFA07A }, 
              { name: 'Hand_R', dims: [0.10, 0.10, 0.10], defaultColor: 0x98FB98 }
          ];
      }
      if (part === 'Legs') {
          targets = [
              { name: 'Thigh_L', dims: [0.16, 0.45, 0.16], defaultColor: 0x008B8B },
              { name: 'Shin_L',  dims: [0.14, 0.40, 0.14], defaultColor: 0x00CED1 },
              { name: 'Thigh_R', dims: [0.16, 0.45, 0.16], defaultColor: 0xFF8C00 },
              { name: 'Shin_R',  dims: [0.14, 0.40, 0.14], defaultColor: 0xFFA500 }
          ];
      }

      targets.forEach(target => {
          const bone = bones.find(b => b.name === target.name);
          if (bone) {
              const mesh = bone.children.find(c => c instanceof THREE.Mesh) as THREE.Mesh;
              if (mesh) {
                  mesh.geometry.dispose(); 
                  if (target.dims) {
                      // For the "Box + Top Cylinder" arms variant, shorten upper/lower arms slightly while keeping hands unchanged
                      let [w, h, d] = [target.dims[0], target.dims[1], target.dims[2]];
                      if (generatorObj.name === 'Box + Top Cylinder' && target.name && (target.name.startsWith('UpperArm') || target.name.startsWith('LowerArm'))) {
                          h = h * 0.85; // reduce height by 15%
                      }
                      mesh.geometry = generatorObj.gen(w, h, d);

                      // Adjust child bone positions so segments remain connected with no gaps.
                      if (part === 'Arms' && target.name) {
                          // If this is an upper arm, move the corresponding lower arm bone to match new length
                          if (target.name.startsWith('UpperArm')) {
                              const lowerName = target.name.replace('UpperArm', 'LowerArm');
                              const lowerBone = bones.find(b => b.name === lowerName);
                              if (lowerBone) lowerBone.position.set(0, -h, 0);
                          }
                          // If this is a lower arm, move the hand bone to attach at the end
                          if (target.name.startsWith('LowerArm')) {
                              const handName = target.name.replace('LowerArm', 'Hand');
                              const handBone = bones.find(b => b.name === handName);
                              if (handBone) handBone.position.set(0, -h, 0);
                          }
                      }
                  } else {
                      mesh.geometry = generatorObj.gen();
                  }
                  
                  // Use constant for colors
                  if (part === 'Head' && generatorObj.name === '8-Sided') {
                      for(let i=mesh.children.length-1; i>=0; i--) mesh.remove(mesh.children[i]);
                      const materials = REF_HEAD_COLORS.map(c => new THREE.MeshStandardMaterial({ color: c, roughness: 0.6, metalness: 0.1 }));
                      mesh.material = materials;
                  } 
                  else {
                      if (Array.isArray(mesh.material)) {
                          mesh.material.forEach(m => m.dispose());
                      }
                      mesh.material = new THREE.MeshStandardMaterial({ 
                          color: target.defaultColor || 0xffffff, 
                          roughness: 0.6, 
                          metalness: 0.1 
                      });

                      // Torso: remove any previous top cylinder, and add a cylinder when using the special generator
                      if (part === 'Torso') {
                          for (let i = mesh.children.length - 1; i >= 0; i--) {
                              const c: any = mesh.children[i];
                              if (c && c.userData && c.userData.isTopCylinder) {
                                  mesh.remove(c);
                                  if (c.geometry) c.geometry.dispose();
                                  if (c.material && c.material.dispose) c.material.dispose();
                              }
                          }

                          if (generatorObj.name === 'Box + Top Cylinder') {
                              // compute dimensions from the box geometry so cylinder always aligns with the actual box
                              mesh.geometry.computeBoundingBox();
                              const bbox = mesh.geometry.boundingBox!;
                              const depth = bbox.max.z - bbox.min.z;
                              const width = bbox.max.x - bbox.min.x;
                              const topY = bbox.max.y;

                              const radius = depth / 2;
                              const length = width * 0.975;
                              const cylGeo = new THREE.CylinderGeometry(radius, radius, length, 32, 1, false);
                              cylGeo.rotateZ(Math.PI / 2); // align axis along X
                              const cylMat = (mesh.material instanceof Array) ? (mesh.material[0] as any).clone() : (mesh.material as any).clone();
                              const cylMesh = new THREE.Mesh(cylGeo, cylMat);
                              cylMesh.userData.isTopCylinder = true;
                              cylMesh.position.set(0, topY, 0); // center at top of box
                              cylMesh.castShadow = true;
                              cylMesh.receiveShadow = true;
                              mesh.add(cylMesh);

                              // Add circular edge outlines to the cylinder
                              const cylCirclePositions: number[] = [];
                              const segments = 32;
                              const cylRadius = radius;

                              // Front circle
                              for (let j = 0; j <= segments; j++) {
                                  const angle = (j / segments) * Math.PI * 2;
                                  const y = Math.cos(angle) * cylRadius;
                                  const z = Math.sin(angle) * cylRadius;
                                  cylCirclePositions.push(length/2, y, z);
                              }

                              // Back circle
                              for (let j = 0; j <= segments; j++) {
                                  const angle = (j / segments) * Math.PI * 2;
                                  const y = Math.cos(angle) * cylRadius;
                                  const z = Math.sin(angle) * cylRadius;
                                  cylCirclePositions.push(-length/2, y, z);
                              }

                              if (cylCirclePositions.length > 0) {
                                  const circleGeometry = new THREE.BufferGeometry();
                                  circleGeometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(cylCirclePositions), 3));
                                  const edgesMaterial = new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 2 });
                                  const circleLines = new THREE.Line(circleGeometry, edgesMaterial);
                                  circleLines.userData.isCylinderOutline = true;
                                  cylMesh.add(circleLines);
                              }
                          }

                          // After torso geometry changes, reposition the yellow triangle so its flat side aligns with the top
                          if (part === 'Torso') {
                              // compute top Y in world space considering torso mesh and any child meshes (eg. top cylinder)
                              mesh.updateMatrixWorld(true);
                              let worldTopY = -Infinity;
                              const tmpVec = new THREE.Vector3();
                              const considerMesh = (m: any) => {
                                  if (!m.geometry) return;
                                  if (!m.geometry.boundingBox) m.geometry.computeBoundingBox();
                                  const bb = m.geometry.boundingBox!;
                                  // use the local top center point as representative
                                  tmpVec.set((bb.max.x + bb.min.x) * 0.5, bb.max.y, (bb.max.z + bb.min.z) * 0.5);
                                  m.localToWorld(tmpVec);
                                  worldTopY = Math.max(worldTopY, tmpVec.y);
                              };
                              considerMesh(mesh);
                              mesh.children.forEach((c: any) => { if (c instanceof THREE.Mesh) considerMesh(c); });

                              const tri: any = mesh.children.find(c => c && c.userData && c.userData.isTorsoTriangle);
                              if (tri && tri.geometry) {
                                  if (!tri.geometry.boundingBox) tri.geometry.computeBoundingBox();
                                  const flatLocalY = -tri.geometry.boundingBox!.min.y;
                                  // convert worldTopY into torso-local Y
                                  const localTop = new THREE.Vector3(0, worldTopY, 0);
                                  mesh.worldToLocal(localTop);
                                  tri.position.y = localTop.y - flatLocalY;
                                  tri.position.z = 0.126;
                              }
                          }
                      }

                      // Hips: remove any previous hip capsule and add one when using the Box + Round generator
                      if (part === 'Hips') {
                          for (let i = mesh.children.length - 1; i >= 0; i--) {
                              const c: any = mesh.children[i];
                              if (c && c.userData && c.userData.isHipCapsule) {
                                  mesh.remove(c);
                                  if (c.geometry) c.geometry.dispose();
                                  if (c.material && c.material.dispose) c.material.dispose();
                              }
                          }

                          if (generatorObj.name === 'Box + Round') {
                              // Smaller, lower capsule to sit under the box portion
                              const capGeo = new THREE.CapsuleGeometry(0.145, 0.04, 4, 8);
                              capGeo.rotateX(Math.PI / 2);
                              // compress slightly in XY and more in Z so it appears smaller and tucked under
                              capGeo.scale(0.9, 0.9, 0.6);
                              // translate lower than before so it sits further down relative to the box
                              capGeo.translate(0, 0.03, 0);
                              const capMat = (mesh.material instanceof Array) ? (mesh.material[0] as any).clone() : (mesh.material as any).clone();
                              const capMesh = new THREE.Mesh(capGeo, capMat);
                              capMesh.userData.isHipCapsule = true;
                              capMesh.castShadow = true;
                              capMesh.receiveShadow = true;
                              mesh.add(capMesh);
                          }
                      }

                      // Arms: add a rounded top on UpperArm_* when the selected arms generator is the Box + Top Cylinder variant
                      if (part === 'Arms') {
                          // remove any previous arm top cylinders
                          for (let i = mesh.children.length - 1; i >= 0; i--) {
                              const c: any = mesh.children[i];
                              if (c && c.userData && c.userData.isArmTopCylinder) {
                                  mesh.remove(c);
                                  if (c.geometry) c.geometry.dispose();
                                  if (c.material && c.material.dispose) c.material.dispose();
                              }
                          }

                          if (generatorObj.name === 'Box + Top Cylinder' && target.name && target.name.startsWith('UpperArm')) {
                              mesh.geometry.computeBoundingBox();
                              const bbox = mesh.geometry.boundingBox!;
                              const width = bbox.max.x - bbox.min.x;
                              const depth = bbox.max.z - bbox.min.z;
                              const topY = bbox.max.y;

                              const radius = width / 2;
                              const length = depth * 0.97;
                              const cylGeo = new THREE.CylinderGeometry(radius, radius, length, 32, 1, false);
                              cylGeo.rotateZ(Math.PI / 2); // align axis along Z
                              const cylMat = (mesh.material instanceof Array) ? (mesh.material[0] as any).clone() : (mesh.material as any).clone();
                              const cylMesh = new THREE.Mesh(cylGeo, cylMat);
                              cylMesh.userData.isArmTopCylinder = true;
                              cylMesh.position.set(0, topY, 0);
                              cylMesh.castShadow = true;
                              cylMesh.receiveShadow = true;
                              mesh.add(cylMesh);

                              // Add circular edge outlines to the cylinder
                              const cylCirclePositions: number[] = [];
                              const segments = 32;
                              const cylRadius = radius;

                              // Front circle
                              for (let j = 0; j <= segments; j++) {
                                  const angle = (j / segments) * Math.PI * 2;
                                  const x = Math.cos(angle) * cylRadius;
                                  const y = Math.sin(angle) * cylRadius;
                                  cylCirclePositions.push(length/2, x, y);
                              }

                              // Back circle
                              for (let j = 0; j <= segments; j++) {
                                  const angle = (j / segments) * Math.PI * 2;
                                  const x = Math.cos(angle) * cylRadius;
                                  const y = Math.sin(angle) * cylRadius;
                                  cylCirclePositions.push(-length/2, x, y);
                              }

                              if (cylCirclePositions.length > 0) {
                                  const circleGeometry = new THREE.BufferGeometry();
                                  circleGeometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(cylCirclePositions), 3));
                                  const edgesMaterial = new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 2 });
                                  const circleLines = new THREE.Line(circleGeometry, edgesMaterial);
                                  circleLines.userData.isCylinderOutline = true;
                                  cylMesh.add(circleLines);
                              }
                          }

                          // Update vertical outlines for all arm pieces (UpperArm, LowerArm)
                          if (target.name && (target.name.startsWith('UpperArm') || target.name.startsWith('LowerArm'))) {
                              updateArmOutlines(mesh, generatorObj.name);
                          }
                      }

                      if (part === 'Head') {
                          for(let i=mesh.children.length-1; i>=0; i--) mesh.remove(mesh.children[i]);
                          const lineGeo = new THREE.BoxGeometry(0.03, 0.25, 0.005); 
                          const lineMat = new THREE.MeshBasicMaterial({ color: 0x000000 }); 
                          const lineMesh = new THREE.Mesh(lineGeo, lineMat);
                          const zOffset = (generatorObj.name === 'Round') ? 0.13 : 0.126;
                          const yOffset = (generatorObj.name === 'Round') ? 0.14 : 0.125;
                          lineMesh.position.set(0, yOffset, zOffset);
                          mesh.add(lineMesh);
                      }
                  }
              }
          }
      });
  }

  function createHardcodedHumanoid() {
    bones = []; 

    const createBoneAndMesh = (name: string, geometry: THREE.BufferGeometry, color: number, parent: THREE.Object3D, pos: [number, number, number]) => {
         const bone = new THREE.Bone();
         bone.name = name;
         bone.position.set(...pos);
         parent.add(bone);
         bones.push(bone);
         
         bone.userData = { 
             isLimb: false,
             initialPos: bone.position.clone(),
             initialRot: bone.rotation.clone()
         };

         const material = new THREE.MeshStandardMaterial({ color: color, roughness: 0.6, metalness: 0.1 });
         const mesh = new THREE.Mesh(geometry, material);
         mesh.castShadow = true;
         mesh.receiveShadow = true;
         mesh.userData = { isLimb: true, boneReference: bone };
         bone.add(mesh);
         return { bone, mesh };
    };

    const addBlackOutline = (mesh: THREE.Mesh) => {
         const edgesGeometry = new THREE.EdgesGeometry(mesh.geometry);
         const edgesMaterial = new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 2 });
         const wireframe = new THREE.LineSegments(edgesGeometry, edgesMaterial);
         mesh.add(wireframe);
    };

    const addVerticalLineOutline = (mesh: THREE.Mesh) => {
         const geometry = mesh.geometry;
         const positions = geometry.attributes.position.array as Float32Array;
         const index = geometry.index?.array as Uint32Array | Uint16Array | null;
         
         const verticalLines = new THREE.BufferGeometry();
         const linePositions: number[] = [];
         
         if (index) {
             for (let i = 0; i < index.length; i += 2) {
                 const i1 = index[i] * 3;
                 const i2 = index[i + 1] * 3;
                 
                 const x1 = positions[i1];
                 const y1 = positions[i1 + 1];
                 const z1 = positions[i1 + 2];
                 
                 const x2 = positions[i2];
                 const y2 = positions[i2 + 1];
                 const z2 = positions[i2 + 2];
                 
                 // Check if edge is vertical (similar X and Z, different Y)
                 const xDiff = Math.abs(x2 - x1);
                 const zDiff = Math.abs(z2 - z1);
                 const yDiff = Math.abs(y2 - y1);
                 
                 if (xDiff < 0.02 && zDiff < 0.02 && yDiff > 0.05) {
                     linePositions.push(x1, y1, z1, x2, y2, z2);
                 }
             }
         }
         
         if (linePositions.length > 0) {
             verticalLines.setAttribute('position', new THREE.BufferAttribute(new Float32Array(linePositions), 3));
             const edgesMaterial = new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 2 });
             const wireframe = new THREE.LineSegments(verticalLines, edgesMaterial);
             mesh.add(wireframe);
         }
    };

    const addThickVerticalLineOutline = (mesh: THREE.Mesh, tubeRadius: number = 0.008) => {
         const geometry = mesh.geometry;
         const positions = geometry.attributes.position.array as Float32Array;
         const index = geometry.index?.array as Uint32Array | Uint16Array | null;
         
         const group = new THREE.Group();
         
         if (index) {
             for (let i = 0; i < index.length; i += 2) {
                 const i1 = index[i] * 3;
                 const i2 = index[i + 1] * 3;
                 
                 const x1 = positions[i1];
                 const y1 = positions[i1 + 1];
                 const z1 = positions[i1 + 2];
                 
                 const x2 = positions[i2];
                 const y2 = positions[i2 + 1];
                 const z2 = positions[i2 + 2];
                 
                 // Check if edge is vertical (similar X and Z, different Y)
                 const xDiff = Math.abs(x2 - x1);
                 const zDiff = Math.abs(z2 - z1);
                 const yDiff = Math.abs(y2 - y1);
                 
                 if (xDiff < 0.02 && zDiff < 0.02 && yDiff > 0.05) {
                     const v1 = new THREE.Vector3(x1, y1, z1);
                     const v2 = new THREE.Vector3(x2, y2, z2);
                     const curve = new THREE.LineCurve3(v1, v2);
                     const tubeGeo = new THREE.TubeGeometry(curve, 2, tubeRadius, 4, false);
                     const tubeMat = new THREE.MeshBasicMaterial({ color: 0x000000 });
                     const tube = new THREE.Mesh(tubeGeo, tubeMat);
                     group.add(tube);
                 }
             }
         }
         
         if (group.children.length > 0) {
             mesh.add(group);
         }
    };

    const getGeo = (part: ShapeCategory, dims?: number[]) => {
        const gen = SHAPE_REGISTRY[part][shapeState[part]].gen;
        return dims ? gen(dims[0], dims[1], dims[2]) : gen();
    };

    // Hips
    const hipsData = createBoneAndMesh("Hips", getGeo('Hips'), 0x8A2BE2, scene, [0, 0.9, 0]); 
    addThickVerticalLineOutline(hipsData.mesh, 0.006);
    addVerticalLineOutline(hipsData.mesh);
    updateMeshesForPart('Hips');

    // Torso
    const torsoData = createBoneAndMesh("Torso", getGeo('Torso'), 0x4169E1, hipsData.bone, [0, 0.2, 0]);
    addVerticalLineOutline(torsoData.mesh);
    
    // Add Lower Torso mesh (different color)
    const lowerTorsoGeo = new THREE.BoxGeometry(0.41, 0.27, 0.26);
    lowerTorsoGeo.translate(0, 0.1, 0);
    const lowerTorsoMat = new THREE.MeshStandardMaterial({ color: 0x2E5090, roughness: 0.6, metalness: 0.1 });
    const lowerTorsoMesh = new THREE.Mesh(lowerTorsoGeo, lowerTorsoMat);
    lowerTorsoMesh.castShadow = true;
    lowerTorsoMesh.receiveShadow = true;
    lowerTorsoMesh.userData = { isLimb: true, boneReference: torsoData.bone };
    torsoData.bone.add(lowerTorsoMesh);
    addThickVerticalLineOutline(lowerTorsoMesh, 0.006);
    addVerticalLineOutline(lowerTorsoMesh);
    // Add yellow triangle on front side only
    const yellowTriangleGeo = new THREE.BufferGeometry();
    const triangleVertices = new Float32Array([
        -0.06, -0.17, 0,   // bottom left
        0.06, -0.17, 0,    // bottom right
        0, -0.01, 0        // top
    ]);
    yellowTriangleGeo.setAttribute('position', new THREE.BufferAttribute(triangleVertices, 3));
    yellowTriangleGeo.computeVertexNormals();
    const yellowTriangleMat = new THREE.MeshBasicMaterial({ color: 0xFFFF00 });
    const yellowTriangleMesh = new THREE.Mesh(yellowTriangleGeo, yellowTriangleMat);
    // mark for later dynamic positioning and keep Z offset
    yellowTriangleMesh.userData = { isTorsoTriangle: true };
    yellowTriangleMesh.position.set(0, 0.42, 0.126);
    yellowTriangleMesh.rotation.z = -Math.PI;
    yellowTriangleMesh.rotation.x = -0.2;
    // torsoData.mesh.add(yellowTriangleMesh);

    // Add second yellow triangle without rotation.x
    const yellowTriangleGeo2 = new THREE.BufferGeometry();
    const triangleVertices2 = new Float32Array([
        -0.06, -0.1, 0,   // bottom left
        0.06, -0.1, 0,    // bottom right
        0, 0.02, 0        // top
    ]);
    yellowTriangleGeo2.setAttribute('position', new THREE.BufferAttribute(triangleVertices2, 3));
    yellowTriangleGeo2.computeVertexNormals();
    const yellowTriangleMat2 = new THREE.MeshStandardMaterial({ color: 0xFFFF00, roughness: 0.6, metalness: 0.1 });
    const yellowTriangleMesh2 = new THREE.Mesh(yellowTriangleGeo2, yellowTriangleMat2);
    // Triangle is now part of neck, so it will color with the neck
    yellowTriangleMesh2.userData = {};
    yellowTriangleMesh2.position.set(0, 0, 0.13);  // Adjusted for neck local space
    yellowTriangleMesh2.rotation.z = -Math.PI;
    yellowTriangleMesh2.rotation.x = -0.4;
    // Will be added to neck after neck is created
    
    updateMeshesForPart('Torso');
    
    // Neck (Yellow cylinder from torso to head)
    const neckGeo = new THREE.CylinderGeometry(0.08, 0.08, 0.2, 8);
    neckGeo.translate(0, 0.075, 0);
    const neckData = createBoneAndMesh("Neck", neckGeo, 0xFFFF00, torsoData.bone, [0, 0.37, 0]);
    // Add the yellow triangle to the neck mesh
    neckData.mesh.add(yellowTriangleMesh2);
    // Change neck material to MeshStandardMaterial for proper lighting
    const neckMesh = neckData.mesh;
    if (neckMesh.material && !(Array.isArray(neckMesh.material))) {
        (neckMesh.material as any).dispose();
        neckMesh.material = new THREE.MeshStandardMaterial({ color: 0xFFFF00, roughness: 0.6, metalness: 0.1 });
    }
    
    // Head (Default 8-Sided)
    const headData = createBoneAndMesh("Head", getGeo('Head'), 0xFFD700, neckData.bone, [0, 0.18, 0]); 
    updateMeshesForPart('Head'); 

    // Arms
    const uArmDims = [0.12, 0.4, 0.12];
    const lArmDims = [0.10, 0.35, 0.10];
    const handDims = [0.10, 0.10, 0.10];

    const upperArmL = createBoneAndMesh("UpperArm_L", getGeo('Arms', uArmDims), 0xDC143C, torsoData.bone, [0.27, 0.40, 0]);
    const lowerArmL = createBoneAndMesh("LowerArm_L", getGeo('Arms', lArmDims), 0xFF6347, upperArmL.bone, [0, -0.4, 0]);
    const handL     = createBoneAndMesh("Hand_L",     getGeo('Arms', handDims), 0xFFA07A, lowerArmL.bone, [0, -0.35, 0]);
    addBlackOutline(handL.mesh);

    const upperArmR = createBoneAndMesh("UpperArm_R", getGeo('Arms', uArmDims), 0x006400, torsoData.bone, [-0.27, 0.40, 0]);
    const lowerArmR = createBoneAndMesh("LowerArm_R", getGeo('Arms', lArmDims), 0x32CD32, upperArmR.bone, [0, -0.4, 0]);
    const handR     = createBoneAndMesh("Hand_R",     getGeo('Arms', handDims), 0x98FB98, lowerArmR.bone, [0, -0.35, 0]);
    addBlackOutline(handR.mesh);

    // Apply selected arms variant (so the top cylinders appear immediately if Box + Top Cylinder is selected)
    updateMeshesForPart('Arms');

    // Legs
    const thighDims = [0.16, 0.45, 0.16];
    const shinDims  = [0.14, 0.40, 0.14];

    const upperLegL = createBoneAndMesh("Thigh_L", getGeo('Legs', thighDims), 0x008B8B, hipsData.bone, [0.1, 0, 0]);
    addVerticalLineOutline(upperLegL.mesh);
    const lowerLegL = createBoneAndMesh("Shin_L",  getGeo('Legs', shinDims), 0x00CED1, upperLegL.bone, [0, -0.45, 0]);
    addVerticalLineOutline(lowerLegL.mesh);
    const footL     = createBoneAndMesh("Foot_L",  getGeo('Feet'), 0xAFEEEE, lowerLegL.bone, [0, -0.4, 0.05]);
    addBlackOutline(footL.mesh);

    const upperLegR = createBoneAndMesh("Thigh_R", getGeo('Legs', thighDims), 0xFF8C00, hipsData.bone, [-0.1, 0, 0]);
    addVerticalLineOutline(upperLegR.mesh);
    const lowerLegR = createBoneAndMesh("Shin_R",  getGeo('Legs', shinDims), 0xFFA500, upperLegR.bone, [0, -0.45, 0]);
    addVerticalLineOutline(lowerLegR.mesh);
    const footR     = createBoneAndMesh("Foot_R",  getGeo('Feet'), 0xFFDAB9, lowerLegR.bone, [0, -0.4, 0.05]);
    addBlackOutline(footR.mesh);

    skeletonHelper = new THREE.SkeletonHelper(hipsData.bone);
    scene.add(skeletonHelper);
  }

  // --- Logic Functions ---
  
  function saveHistory() {
      const snapshot = bones.map(b => ({ rot: b.rotation.clone(), pos: b.position.clone() }));
      undoStack.push(snapshot);
      if (undoStack.length > 3) undoStack.shift(); 
      undoStack = undoStack; 
  }

  function undoLastAction() {
      if (undoStack.length === 0 || isPlaying) return;
      const lastState = undoStack.pop();
      if (lastState) {
          bones.forEach((b, i) => {
              b.rotation.copy(lastState[i].rot);
              b.position.copy(lastState[i].pos);
          });
      }
      undoStack = undoStack;
      if (activeFrameIndex > -1) updateCurrentFrame();
  }

  function resetRootPosition() {
    saveHistory(); 
    const hips = bones.find(b => b.name === "Hips");
    if(hips) hips.position.set(0, 0.9, 0);
    if (activeFrameIndex > -1) updateCurrentFrame();
  }
  
  function resetPose() {
      saveHistory();
      bones.forEach(b => {
          if (b.userData.initialRot) b.rotation.copy(b.userData.initialRot);
          if (b.userData.initialPos) b.position.copy(b.userData.initialPos);
      });
      if (activeFrameIndex > -1) updateCurrentFrame();
  }

  function mirrorCurrentPose() {
    if(isPlaying) return;
    saveHistory();
    const pairs = [['UpperArm_L', 'UpperArm_R'], ['LowerArm_L', 'LowerArm_R'], ['Hand_L', 'Hand_R'], ['Thigh_L', 'Thigh_R'], ['Shin_L', 'Shin_R'], ['Foot_L', 'Foot_R']];
    pairs.forEach(([left, right]) => {
        const lBone = bones.find(b => b.name === left);
        const rBone = bones.find(b => b.name === right);
        if (lBone && rBone) {
            const lRot = lBone.rotation.clone();
            const rRot = rBone.rotation.clone();
            lBone.rotation.set(rRot.x, -rRot.y, -rRot.z);
            rBone.rotation.set(lRot.x, -lRot.y, -lRot.z);
        }
    });
    ['Hips', 'Torso', 'Head'].forEach(name => {
        const bone = bones.find(b => b.name === name);
        if (bone) { bone.rotation.y *= -1; bone.rotation.z *= -1; if (name === 'Hips') bone.position.x *= -1; }
    });
    if (activeFrameIndex > -1) updateCurrentFrame();
  }

  function onPointerDown(event: MouseEvent) {
    if (isPlaying || isWebcamRunning || isExporting || isProcessingVideo) return; 
    const rect = canvas.getBoundingClientRect();
    pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(pointer, camera);
    const intersects = raycaster.intersectObjects(scene.children, true);
    const limbHit = intersects.find(i => i.object.userData.isLimb);
    if (limbHit) {
      const bone = limbHit.object.userData.boneReference;
      selectedBoneIndex = bones.indexOf(bone);
      saveHistory(); 
      isDraggingBone = true;
      orbitControl.enabled = false;
      lastPointerX = event.clientX;
      lastPointerY = event.clientY;
    } else {
      selectedBoneIndex = -1;
      isDraggingBone = false;
      orbitControl.enabled = true;
    }
  }

  function onPointerMove(event: MouseEvent) {
    if (!isDraggingBone || selectedBoneIndex === -1 || isPlaying || isExporting || isProcessingVideo) return;
    const deltaX = event.clientX - lastPointerX;
    const deltaY = event.clientY - lastPointerY;
    lastPointerX = event.clientX;
    lastPointerY = event.clientY;
    const bone = bones[selectedBoneIndex];
    if (currentTool === 'rotate') {
        if (isShiftDown) { bone.rotation.y += deltaX * rotationSpeed; } else { bone.rotation.z += deltaX * rotationSpeed; bone.rotation.x += deltaY * rotationSpeed; }
    } else if (currentTool === 'twist') {
        bone.rotation.y += deltaX * rotationSpeed;
    } else if (currentTool === 'move') {
        if (isShiftDown) { bone.position.z += deltaY * moveSpeed; } else { bone.position.x += deltaX * moveSpeed; bone.position.y -= deltaY * moveSpeed; }
    }
  }

  function onPointerUp() { isDraggingBone = false; orbitControl.enabled = true; if(activeFrameIndex > -1) updateCurrentFrame(); }
  function onKeyDown(event: KeyboardEvent) {
    if (event.key === 'Shift') isShiftDown = true;
    if (event.key === 'Escape') selectedBoneIndex = -1;
    if (event.key.toLowerCase() === 'r') currentTool = 'rotate';
    if (event.key.toLowerCase() === 'e') currentTool = 'twist';
    if (event.key.toLowerCase() === 't') currentTool = 'move';
    if ((event.ctrlKey || event.metaKey) && event.key.toLowerCase() === 'z') undoLastAction();
  }
  function onKeyUp(event: KeyboardEvent) { if (event.key === 'Shift') isShiftDown = false; }

  // --- Mocap & Processing ---


  function toggleWebcam() {
    console.log('Webcam functionality has been removed');
  }

  function handlePoseImageSelect(event: Event) {
    console.log('Pose image processing has been removed');
    const input = event.target as HTMLInputElement;
    input.value = '';
  }

  async function handlePoseVideoSelect(event: Event) {
    console.log('Pose video processing has been removed');
    const input = event.target as HTMLInputElement;
    input.value = '';
  }



  function applyPoseToBones(lm: any[], legAmp: number, armAmp: number, depthAmp: number, armOffset: number, instant: boolean) {
    const posLerp = instant ? 1.0 : 0.1;
    const rotLerp = instant ? 1.0 : 0.2;
    if (lm[23] && lm[24]) {
       const hipsBone = bones.find(b => b.name === "Hips");
       if (hipsBone) {
          const rawX = (lm[23].x + lm[24].x) / 2;
          const rawY = (lm[23].y + lm[24].y) / 2;
          const xMult = isMirrored ? 3 : -3;
          const targetX = (rawX - 0.5) * xMult; 
          const targetY = (1 - rawY) * 3 - 0.5;
          hipsBone.position.x = THREE.MathUtils.lerp(hipsBone.position.x, targetX, posLerp);
          hipsBone.position.y = THREE.MathUtils.lerp(hipsBone.position.y, targetY, posLerp);
       }
    }
    MP_CHAINS.forEach(chain => {
       const start = lm[chain.start];
       const end = lm[chain.end];
       if (start && end && start.visibility > 0.5 && end.visibility > 0.5) {
          const bone = bones.find(b => b.name === chain.name);
          if (bone) {
             const dX = end.x - start.x; const dY = end.y - start.y; const dZ = end.z - start.z;
             const vecX = isMirrored ? -dX : dX;
             const vecY = -dY;
             let vecZ = -dZ * depthAmp; 
             if (!chain.isLeg) vecZ += armOffset;
             tempVector.set(vecX, vecY, vecZ).normalize();
             tempQuaternion.setFromUnitVectors(T_POSE_VECTOR, tempVector);
             const amp = chain.isLeg ? legAmp : armAmp;
             if (amp !== 1.0) {
                 const dampedQ = identityQuaternion.clone().slerp(tempQuaternion, amp);
                 bone.quaternion.slerp(dampedQ, rotLerp);
             } else {
                 bone.quaternion.slerp(tempQuaternion, rotLerp);
             }
          }
       }
    });
  }

  // --- Frame Management ---
  function selectFrame(index: number) {
    if (isPlaying || isWebcamRunning) return;
    activeFrameIndex = index;
    const frame = keyframes[index];
    if (frame) {
      frame.data.forEach((d: any, i: number) => { bones[i].rotation.copy(d.rot); bones[i].position.copy(d.pos); });
      if (frame.legAmplitude !== undefined) legAmplitude = frame.legAmplitude;
      if (frame.armAmplitude !== undefined) armAmplitude = frame.armAmplitude;
      if (frame.depthAmplitude !== undefined) depthAmplitude = frame.depthAmplitude; 
      if (frame.armForwardOffset !== undefined) armForwardOffset = frame.armForwardOffset;
      if (frame.sourceImage && mocapCanvas) {
          const ctx = mocapCanvas.getContext('2d');
          const img = new Image(); img.src = frame.sourceImage;
          img.onload = () => { if (ctx) ctx.drawImage(img, 0, 0, mocapCanvas.width, mocapCanvas.height); }
          showMocapDebug = true;
      } else { showMocapDebug = false; }
      if (frame.landmarks) lastMocapResults = { landmarks: frame.landmarks }; else lastMocapResults = null;
    }
  }

  function updateCurrentFrame() {
    if (activeFrameIndex === -1) return;
    const frameData = bones.map(b => ({ rot: b.rotation.clone(), pos: b.position.clone() }));
    keyframes[activeFrameIndex].data = frameData;
    keyframes[activeFrameIndex].legAmplitude = legAmplitude;
    keyframes[activeFrameIndex].armAmplitude = armAmplitude;
    keyframes[activeFrameIndex].depthAmplitude = depthAmplitude; 
    keyframes[activeFrameIndex].armForwardOffset = armForwardOffset;
    keyframes = keyframes; 
  }

  function addNewFrame() {
    const frameData = bones.map(b => ({ rot: b.rotation.clone(), pos: b.position.clone() }));
    keyframes = [...keyframes, { 
        data: frameData, id: Date.now(), legAmplitude, armAmplitude, depthAmplitude, armForwardOffset,
        sourceImage: lastMocapResults?.image || null, landmarks: lastMocapResults?.landmarks || null
    }];
    activeFrameIndex = keyframes.length - 1;
  }

  function deleteFrame(index: number) {
    if (activeFrameIndex === index) activeFrameIndex = -1;
    else if (activeFrameIndex > index) activeFrameIndex--;
    keyframes = keyframes.filter((_, i) => i !== index);
  }
  
  function onAmplitudeChange() {
      if (lastMocapResults && lastMocapResults.landmarks) {
          applyPoseToBones(lastMocapResults.landmarks, legAmplitude, armAmplitude, depthAmplitude, armForwardOffset, true); 
          if (activeFrameIndex > -1) updateCurrentFrame();
      }
  }

  function triggerPoseImageUpload() { if (isWebcamRunning) toggleWebcam(); imagePoseInput.click(); }
  function triggerPoseVideoUpload() { if (isWebcamRunning) toggleWebcam(); videoPoseInput.click(); }
  
  // --- Export Modal ---
  function openExportModal() {
      exportSettings.width = renderWidth;
      exportSettings.height = renderHeight;
      showExportModal = true;
  }
  
  function doExport() {
      showExportModal = false;
      generateSpriteSheet();
  }

  function updateResolution() {
    renderer.setSize(renderWidth, renderHeight, false);
    camera.aspect = renderWidth / renderHeight;
    camera.updateProjectionMatrix();
  }
  function onResize() {}
  
  function exportProject() {
    const projectData = { renderWidth, renderHeight, playbackSpeed, keyframes, boneColors };
    const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(projectData));
    const a = document.createElement('a'); a.href = dataStr; a.download = `anim_project_${Date.now()}.json`;
    document.body.appendChild(a); a.click(); a.remove();
  }

  function triggerImport() { fileInput.click(); }
  function handleFileSelect(event: Event) {
    const input = event.target as HTMLInputElement;
    if (!input.files || !input.files.length) return;
    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const json = JSON.parse(e.target?.result as string);
        if (json.renderWidth) renderWidth = json.renderWidth;
        if (json.renderHeight) renderHeight = json.renderHeight;
        if (json.playbackSpeed) playbackSpeed = json.playbackSpeed;
        if (json.boneColors) boneColors = json.boneColors;
        if (Array.isArray(json.keyframes)) {
           keyframes = json.keyframes.map((k: any) => ({
             id: k.id || Date.now(),
             legAmplitude: k.legAmplitude || 0.7, armAmplitude: k.armAmplitude || 1.0, 
             depthAmplitude: k.depthAmplitude || 1.0, armForwardOffset: k.armForwardOffset || 0.0, 
             sourceImage: k.sourceImage || null, landmarks: k.landmarks || null,
             data: k.data.map((b: any) => ({
                rot: new THREE.Euler(b.rot._x ?? b.rot.x, b.rot._y ?? b.rot.y, b.rot._z ?? b.rot.z),
                pos: new THREE.Vector3(b.pos.x, b.pos.y, b.pos.z)
             }))
           }));
        }
        updateResolution();
        // Apply loaded colors to bones
        Object.entries(boneColors).forEach(([boneName, color]) => {
            changeBoneColor(boneName, color as string);
        });
        if (keyframes.length > 0) selectFrame(0); else activeFrameIndex = -1;
      } catch (err) { alert("Failed to load project."); }
    };
    reader.readAsText(input.files[0]); input.value = ''; 
  }

  function handleDragStart(e: DragEvent, index: number) { if (isPlaying) return; draggingFrameIndex = index; e.dataTransfer!.effectAllowed = 'move'; }
  function handleDragOver(e: DragEvent, index: number) { e.preventDefault(); if (isPlaying) return; dragOverIndex = index; e.dataTransfer!.dropEffect = 'move'; }
  function handleDrop(e: DragEvent, targetIndex: number) {
    e.preventDefault();
    if (draggingFrameIndex === null || draggingFrameIndex === targetIndex) { dragOverIndex = null; draggingFrameIndex = null; return; }
    const activeFrameId = activeFrameIndex > -1 ? keyframes[activeFrameIndex].id : null;
    const itemToMove = keyframes[draggingFrameIndex];
    const newKeyframes = [...keyframes];
    newKeyframes.splice(draggingFrameIndex, 1);
    newKeyframes.splice(targetIndex, 0, itemToMove);
    keyframes = newKeyframes;
    if (activeFrameId) activeFrameIndex = keyframes.findIndex(f => f.id === activeFrameId);
    draggingFrameIndex = null; dragOverIndex = null;
  }

  function togglePlay() {
    isPlaying = !isPlaying;
    if (isWebcamRunning && isPlaying) toggleWebcam(); 
    if (isPlaying) { clock.start(); activeFrameIndex = -1; selectedBoneIndex = -1; } else { clock.stop(); if (keyframes.length > 0) selectFrame(Math.floor(animationTime * playbackSpeed) % keyframes.length); }
  }

  function animate() {
    requestAnimationFrame(animate);
    
    // Always apply Head Twist
    const headBone = bones.find(b => b.name === 'Head');
    if (headBone) {
        const mesh = headBone.children.find(c => c instanceof THREE.Mesh) as THREE.Mesh;
        if (mesh) {
            mesh.rotation.y = headTwist * (Math.PI / 180);
        }
    }

    if (isExporting || isProcessingVideo) return; 

    if (isPlaying && keyframes.length > 1) {
      const delta = clock.getDelta();
      animationTime += delta;
      const durationPerFrame = 1 / playbackSpeed;
      const totalDuration = durationPerFrame * keyframes.length;
      const loopedTime = animationTime % totalDuration;
      
      const frameIndex = Math.floor(loopedTime / durationPerFrame);
      const nextFrameIndex = (frameIndex + 1) % keyframes.length;
      const alpha = (loopedTime - (frameIndex * durationPerFrame)) / durationPerFrame;

      const frameA = keyframes[frameIndex];
      const frameB = keyframes[nextFrameIndex];

      if (frameA && frameB) {
        bones.forEach((bone, i) => {
          bone.position.lerpVectors(frameA.data[i].pos, frameB.data[i].pos, alpha);
          const rotA = new THREE.Quaternion().setFromEuler(frameA.data[i].rot);
          const rotB = new THREE.Quaternion().setFromEuler(frameB.data[i].rot);
          bone.quaternion.slerpQuaternions(rotA, rotB, alpha);
        });
      }
    }
    orbitControl.update();
    renderer.render(scene, camera);
  }

  async function generateSpriteSheet() {
    if (keyframes.length === 0) { alert("No keyframes!"); return; }
    isExporting = true;
    orbitControl.enabled = false;
    orbitControl.target.set(0, 1, 0); 
    skeletonHelper.visible = false;
    const grid = scene.children.find(c => c instanceof THREE.GridHelper);
    if(grid) grid.visible = false;
    
    const originalFrame = activeFrameIndex;
    const originalPos = camera.position.clone();
    const originalBackground = scene.background;
    
    // Set transparent background if enabled
    if (exportSettings.transparentBackground) {
        scene.background = null;
    }
    
    // Store Original Materials and convert all mesh descendants to MeshBasicMaterial
    const originalMaterials = new Map<THREE.Mesh, THREE.Material | THREE.Material[]>();
    
    const convertMeshToBasic = (mesh: THREE.Mesh) => {
        // Store original material
        originalMaterials.set(mesh, mesh.material);
        
        // Convert this mesh's material to MeshBasicMaterial
        if (Array.isArray(mesh.material)) {
            mesh.material = mesh.material.map(m => new THREE.MeshBasicMaterial({ color: (m as THREE.MeshStandardMaterial).color }));
        } else {
            mesh.material = new THREE.MeshBasicMaterial({ color: (mesh.material as THREE.MeshStandardMaterial).color });
        }
        
        // Recursively convert all child meshes (for cylinders, capsules, etc added to the main mesh)
        mesh.children.forEach(child => {
            if (child instanceof THREE.Mesh) {
                convertMeshToBasic(child);
            }
        });
    };
    
    bones.forEach(b => {
        const mesh = b.children.find(c => c instanceof THREE.Mesh) as THREE.Mesh;
        if (mesh) {
            convertMeshToBasic(mesh);
        }
    });

    // Resize Renderer
    renderer.setSize(exportSettings.width, exportSettings.height, false);
    camera.aspect = exportSettings.width / exportSettings.height;
    camera.updateProjectionMatrix();

    const directions = 8;
    const frames = keyframes.length;
    
    // Canvas Size Logic
    let sheetWidth = 0;
    let sheetHeight = 0;
    
    // Determine the actual cell dimensions based on bounding box setting
    const cellWidth = exportSettings.useBoundingBox ? exportSettings.boundingBoxWidth : exportSettings.width;
    const cellHeight = exportSettings.useBoundingBox ? exportSettings.boundingBoxHeight : exportSettings.height;
    
    if (exportSettings.layout === 'horizontal') {
        sheetWidth = cellWidth * (frames * directions);
        sheetHeight = cellHeight;
    } else if (exportSettings.layout === 'vertical') {
        sheetWidth = cellWidth;
        sheetHeight = cellHeight * (frames * directions);
    } else {
        sheetWidth = cellWidth * frames;
        sheetHeight = cellHeight * directions;
    }

    exportCanvas.width = sheetWidth;
    exportCanvas.height = sheetHeight;
    const ctx = exportCanvas.getContext('2d', { alpha: true });
    if(ctx) {
        ctx.imageSmoothingEnabled = false;
        // No need to clear - PNG will be transparent by default
    }

    const radius = 5;
    const height = 2;
    const headMesh = bones.find(b => b.name === 'Head')?.children.find(c => c instanceof THREE.Mesh) as THREE.Mesh;
    
    for (let d = 0; d < directions; d++) {
        const angle = (d / directions) * Math.PI * 2 - Math.PI * 0.25;
        camera.position.x = Math.sin(angle) * radius;
        camera.position.z = Math.cos(angle) * radius;
        camera.position.y = height;
        camera.lookAt(0, 1.2, 0);
        camera.updateMatrixWorld();
        
        for (let f = 0; f < frames; f++) {
            const frameData = keyframes[f].data;
            bones.forEach((b, i) => {
                b.rotation.copy(frameData[i].rot);
                b.position.copy(frameData[i].pos);
                b.updateMatrixWorld(true);
            });
            
            // NEW: Normalize Head Color Logic
            if (exportSettings.normalizeHeadColor && headMesh && shapeState.Head === 0) { // 0 is 8-Sided
                // 1. Get Head World Position
                const headWorldPos = new THREE.Vector3();
                headMesh.getWorldPosition(headWorldPos);
                
                // 2. Calculate View Vector relative to Head
                const viewVec = new THREE.Vector3().subVectors(camera.position, headWorldPos).normalize();
                
                // 3. Transform View Vector into Head's Local Space
                // Inverting the head's rotation allows us to treat the head as if it's axis aligned
                const headQuat = new THREE.Quaternion();
                headMesh.getWorldQuaternion(headQuat);
                const invQuat = headQuat.invert();
                viewVec.applyQuaternion(invQuat);
                
                // 4. Calculate Angle in XZ plane
                // Note: Our head geom is rotated Y by PI/8 initially to align a face to Z
                // Standard atan2 is from Z axis? No, Z is 0. 
                let theta = Math.atan2(viewVec.x, viewVec.z);
                
                // 5. Map to 8 segments.
                // Segment 0 center is at angle 0 (Z+).
                // Width is PI/4 (45deg). Range -22.5 to +22.5.
                // Subtract 1 to correct for the head geometry's PI/8 rotation offset.
                const sectorAngle = Math.PI / 4;
                let index = Math.round(theta / sectorAngle) - 1;
                if (index < 0) index += 8;
                index = index % 8;
                
                // 6. Apply Single Color to entire head
                headMesh.material = new THREE.MeshBasicMaterial({ color: REF_HEAD_COLORS[index] });

                // 7. Remove previous outlines and markers except the bottom edge
                for (let i = headMesh.children.length - 1; i >= 0; i--) {
                    const child = headMesh.children[i];
                    // Remove all except bottom edge marker (we'll re-add it below)
                    if (child.userData && (child.userData.isHeadOutline || child.userData.isHeadFaceMarker || child.userData.isHeadBetweenMarker)) {
                        headMesh.remove(child);
                    }
                }
                // Only add the centered white square and the bottom white line (head/neck separator)
                const geometry = headMesh.geometry;
                if (geometry && (geometry.type === 'CylinderGeometry' || geometry.type === 'BufferGeometry')) {
                    // Centered white cross (two rectangles)
                    const radius = 0.15;
                    const y = 0.125; // center y of the head
                    const rectWidth = 0.25; // larger width
                    const rectHeight = 0.06; // larger height
                    const i = index;
                    const theta1 = (i * Math.PI * 2) / 8 + Math.PI / 8;
                    const theta2 = ((i + 1) * Math.PI * 2) / 8 + Math.PI / 8;
                    const midTheta = (theta1 + theta2) / 2;
                    const nx = Math.sin(midTheta);
                    const nz = Math.cos(midTheta);
                    const cx = nx * radius;
                    const cz = nz * radius;
                    // Horizontal rectangle (dark brown), lowered
                    const rectGeoH = new THREE.PlaneGeometry(rectWidth, rectHeight * 0.75);
                    const rectMat = new THREE.MeshBasicMaterial({ color: 0xFFF8E1 }); // offwhite cream
                    const rectH = new THREE.Mesh(rectGeoH, rectMat);
                    const loweredY = y - 0.08; // Lower the line by 0.08 units (adjust as needed)
                    rectH.position.set(cx, loweredY, cz);
                    rectH.lookAt(cx + nx, loweredY, cz + nz);
                    rectH.userData.isHeadBetweenMarker = true;
                    headMesh.add(rectH);

                    // Bottom white line (head/neck separator)
                    // The bottom edge of the 8-sided cylinder is at y = 0 (since translate(0, 0.125, 0) is used)
                    // We'll draw a white line connecting all bottom vertices
                    const segments = 8;
                    const bottomLinePositions = [];
                    for (let j = 0; j <= segments; j++) {
                        const theta = (j / segments) * Math.PI * 2 + Math.PI / 8;
                        const x = Math.sin(theta) * radius;
                        const z = Math.cos(theta) * radius;
                        bottomLinePositions.push(x, 0, z);
                    }
                    const bottomLineGeometry = new THREE.BufferGeometry();
                    bottomLineGeometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(bottomLinePositions), 3));
                    const bottomLineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff, linewidth: 2 });
                    const bottomLine = new THREE.Line(bottomLineGeometry, bottomLineMaterial);
                    bottomLine.userData.isHeadOutline = true;
                    // headMesh.add(bottomLine);
                }
            }
            
            renderer.render(scene, camera);
            
            let dx = 0, dy = 0;
            if (exportSettings.layout === 'horizontal') {
                dx = (d * frames + f) * cellWidth;
            } else if (exportSettings.layout === 'vertical') {
                dy = (d * frames + f) * cellHeight;
            } else {
                dx = f * cellWidth;
                dy = d * cellHeight;
            }

            if(ctx) {
                // Normalize background: fill only the upper left corner with normalized head color from legend
                if (exportSettings.normalizeBackground && headMesh && shapeState.Head === 0) {
                    // 1. Get Head World Position
                    const headWorldPos = new THREE.Vector3();
                    headMesh.getWorldPosition(headWorldPos);
                    // 2. Calculate View Vector relative to Head
                    const viewVec = new THREE.Vector3().subVectors(camera.position, headWorldPos).normalize();
                    // 3. Transform View Vector into Head's Local Space
                    const headQuat = new THREE.Quaternion();
                    headMesh.getWorldQuaternion(headQuat);
                    const invQuat = headQuat.invert();
                    viewVec.applyQuaternion(invQuat);
                    // 4. Calculate Angle in XZ plane
                    let theta = Math.atan2(viewVec.x, viewVec.z);
                    // 5. Map to 8 segments.
                    const sectorAngle = Math.PI / 4;
                    let index = Math.round(theta / sectorAngle) - 1;
                    if (index < 0) index += 8;
                    index = index % 8;
                    // 6. Fill only the upper left corner (1% width/height)
                    ctx.save();
                    ctx.globalAlpha = 1.0; // Ensure fully opaque
                    ctx.fillStyle = `#${REF_HEAD_COLORS[index].toString(16).padStart(6, '0')}`;
                    const cornerWidth = Math.ceil(cellWidth * 0.01);
                    const cornerHeight = Math.ceil(cellHeight * 0.01);
                    ctx.clearRect(dx, dy, cornerWidth, cornerHeight); // Remove any previous pixels
                    ctx.fillRect(dx, dy, cornerWidth, cornerHeight);
                    ctx.globalAlpha = 1.0;
                    ctx.restore();
                }
                if (exportSettings.useBoundingBox) {
                    // Calculate crop region centered on the source canvas with offsets
                    const sourceWidth = exportSettings.width;
                    const sourceHeight = exportSettings.height;
                    const cropWidth = exportSettings.boundingBoxWidth;
                    const cropHeight = exportSettings.boundingBoxHeight;
                    // Calculate the top-left corner of the crop region (centered with offsets)
                    const cropX = (sourceWidth - cropWidth) / 2 + exportSettings.boundingBoxOffsetX;
                    const cropY = (sourceHeight - cropHeight) / 2 + exportSettings.boundingBoxOffsetY;
                    // Clamp to valid bounds
                    const srcX = Math.max(0, cropX);
                    const srcY = Math.max(0, cropY);
                    const srcW = Math.min(cropWidth, sourceWidth - srcX);
                    const srcH = Math.min(cropHeight, sourceHeight - srcY);
                    // Draw the cropped region
                    ctx.drawImage(canvas, srcX, srcY, srcW, srcH, dx, dy, srcW, srcH);
                } else {
                    ctx.drawImage(canvas, dx, dy);
                }
            }
            await new Promise(r => setTimeout(r, 20));
        }
    }

    const link = document.createElement('a');
    link.download = `sprite_${Date.now()}.png`;
    link.href = exportCanvas.toDataURL('image/png');
    link.click();

    // Restore Materials
    originalMaterials.forEach((mat, mesh) => {
        if(Array.isArray(mesh.material)) mesh.material.forEach(m => m.dispose());
        else (mesh.material as THREE.Material).dispose();
        mesh.material = mat;
    });

    // Restore Settings
    scene.background = originalBackground;
    renderer.setSize(renderWidth, renderHeight, false);
    camera.aspect = renderWidth / renderHeight;
    camera.updateProjectionMatrix();
    camera.position.copy(originalPos);
    camera.lookAt(0,1,0);
    skeletonHelper.visible = true;
    if(grid) grid.visible = true;
    if (originalFrame > -1) selectFrame(originalFrame);
    else if (keyframes.length > 0) selectFrame(0);
    orbitControl.enabled = true;
    isExporting = false;
  }
</script>

<div class="layout">
  <div class="viewport">
    <canvas 
      bind:this={canvas} 
      on:pointerdown={onPointerDown}
      on:pointermove={onPointerMove}
      on:pointerup={onPointerUp}
      style="image-rendering: pixelated; width: 100%; height: 100%; cursor: {isDraggingBone ? 'grabbing' : 'grab'};"
    ></canvas>
    
    <video bind:this={videoElement} style="display:none;"></video>
    
    <div 
        class="mocap-overlay" 
        style="display: {showMocapDebug ? 'block' : 'none'};"
    >
        <canvas bind:this={mocapCanvas} class="mocap-canvas"></canvas>
        <button class="mocap-close" on:click={() => showMocapDebug = false}>×</button>
    </div>
    
    <div class="shape-hud">
        {#each Object.keys(shapeState) as part}
            <div class="shape-row">
                <span class="shape-label">{part}</span>
                <div class="shape-controls">
                    <button class="arrow-btn" on:click={() => cycleShape(part, -1)}>&lt;</button>
                    <span class="shape-value">{SHAPE_REGISTRY[part][shapeState[part]].name}</span>
                    <button class="arrow-btn" on:click={() => cycleShape(part, 1)}>&gt;</button>
                </div>
            </div>
        {/each}
        
        <hr style="border:0; border-top:1px solid #555; margin: 8px 0;" />
        <div style="display:flex; flex-direction:column; gap:2px;">
            <span class="shape-label">Head Align (Deg)</span>
            <input 
                type="range" min="-180" max="180" step="1" 
                bind:value={headTwist} 
                style="width: 100%; margin:0;" 
            />
            <span style="font-size:9px; color:#aaa; text-align:right;">{headTwist}°</span>
        </div>
    </div>

    <div class="color-legend">
        <div class="legend-title">Colors</div>
        {#each Object.entries(boneColors) as [boneName, colorValue]}
            <div class="legend-item" class:selected={selectedBoneName === boneName}>
                <div class="legend-item-header" on:click={() => selectBoneByName(boneName)}>
                    <input type="color" value={colorValue} on:change={(e) => changeBoneColor(boneName, e.currentTarget.value)} class="legend-color-input" on:click|stopPropagation />
                    <span class="legend-label">{boneName}</span>
                </div>
                <input 
                    type="text" 
                    value={colorValue} 
                    on:change={(e) => {
                        const val = e.currentTarget.value;
                        if (/^#[0-9A-F]{6}$/i.test(val)) {
                            changeBoneColor(boneName, val);
                        }
                    }}
                    class="legend-hex-input"
                    placeholder="#000000"
                    on:click|stopPropagation
                />
            </div>
        {/each}
    </div>

    {#if isWebcamRunning}
        <div class="recording-indicator">● WEBCAM ACTIVE</div>
    {/if}
    
    {#if isExporting}
        <div class="recording-indicator" style="color: #f1c40f;">● RENDERING SPRITES...</div>
    {/if}
    
    {#if isProcessingVideo}
        <div class="recording-indicator" style="color: #3498db; width: auto;">
            {processingStatus}
        </div>
    {/if}

    <div class="resolution-badge">
      Res: {renderWidth}x{renderHeight}
    </div>
    
    <div class="controls-hint">
      <strong>Controls:</strong><br/>
      Drag Background: Camera<br/>
      Drag Body: Action (Rotate/Move)<br/>
      Shift + Drag: Alternate Axis
    </div>
  </div>

  <div class="sidebar">
    <div class="sidebar-scroll-area">
      <h2>Pixel Bone Editor</h2>

      <div class="control-group" style="background: #333; padding: 10px; border-radius: 4px; margin-bottom: 10px;">
          <label style="color:white; margin-bottom:5px;">Current Tool</label>
          <div style="display: flex; gap: 5px; margin-bottom: 5px;">
              <button 
                  on:click={() => currentTool = 'rotate'} 
                  style="background: {currentTool === 'rotate' ? '#3498db' : '#444'}; color: white; flex:1;"
              >
                  Bend (R)
              </button>
              <button 
                  on:click={() => currentTool = 'twist'} 
                  style="background: {currentTool === 'twist' ? '#9b59b6' : '#444'}; color: white; flex:1;"
              >
                  Twist (E)
              </button>
              <button 
                  on:click={() => currentTool = 'move'} 
                  style="background: {currentTool === 'move' ? '#f39c12' : '#444'}; color: white; flex:1;"
              >
                  Move (T)
              </button>
          </div>
          
          <div style="display: flex; gap: 5px; margin-bottom:5px;">
              <button on:click={mirrorCurrentPose} style="background: #e67e22; flex:1;">
                  Flip Pose
              </button>
              <button on:click={resetRootPosition} style="background: #e74c3c; flex:1;">
                  Reset Root
              </button>
          </div>
          
          <div style="display: flex; gap: 5px;">
              <button on:click={undoLastAction} style="background: #95a5a6; flex:1;">
                  Undo (Ctrl+Z)
              </button>
              <button on:click={resetPose} style="background: #c0392b; flex:1;">
                  Reset Pose
              </button>
          </div>
      </div>

      <!--<div class="control-group" style="background: #2c3e50; padding: 10px; border-radius: 4px; margin-bottom: 10px; border: 1px solid #34495e;">
          <label style="color:#3498db; margin-bottom:5px;">Motion Capture</label>
          
          <button 
              on:click={toggleWebcam}
              style="background: {isWebcamRunning ? '#e74c3c' : '#2980b9'}; color: white; width: 100%; margin-bottom: 5px;"
          >
              {isWebcamRunning ? 'Stop Webcam' : 'Start Webcam'}
          </button>
          
          <button 
              on:click={triggerPoseImageUpload}
              style="background: #8e44ad; color: white; width: 100%; margin-bottom: 5px;"
          >
              Extract Pose from Image
          </button>
          <input type="file" accept="image/*" bind:this={imagePoseInput} on:change={handlePoseImageSelect} style="display: none;" />

          <button 
              on:click={triggerPoseVideoUpload}
              style="background: #16a085; color: white; width: 100%; margin-bottom: 5px;"
          >
              Extract Pose from Video
          </button>
          <input type="file" accept="video/*" bind:this={videoPoseInput} on:change={handlePoseVideoSelect} style="display: none;" />

          <label style="display: flex; align-items: center; cursor: pointer; margin-top: 5px; color: #ccc; font-size: 11px;">
              <input type="checkbox" bind:checked={isMirrored} style="width: auto; margin-right: 5px;">
              Mirror Pose (Flip X)
          </label>
          
          <div style="margin-top: 10px;">
              <label style="font-size: 10px; color: #ccc;">Z-Depth Scale: {depthAmplitude.toFixed(2)}</label>
              <input 
                  type="range" min="0.5" max="3.0" step="0.1" 
                  bind:value={depthAmplitude} 
                  on:input={onAmplitudeChange}
                  style="width: 100%;" 
              />
          </div>
          
          <div style="margin-top: 5px;">
              <label style="font-size: 10px; color: #ccc;">Arm Forward Offset: {armForwardOffset.toFixed(2)}</label>
              <input 
                  type="range" min="-0.5" max="0.5" step="0.01" 
                  bind:value={armForwardOffset} 
                  on:input={onAmplitudeChange}
                  style="width: 100%;" 
              />
          </div>
          
          <div style="margin-top: 5px;">
              <label style="font-size: 10px; color: #ccc;">Arm Motion Scale: {armAmplitude.toFixed(2)}</label>
              <input 
                  type="range" min="0.1" max="1.5" step="0.1" 
                  bind:value={armAmplitude} 
                  on:input={onAmplitudeChange}
                  style="width: 100%;" 
              />
          </div>
          
          <div style="margin-top: 5px;">
              <label style="font-size: 10px; color: #ccc;">Leg Motion Scale: {legAmplitude.toFixed(2)}</label>
              <input 
                  type="range" min="0.1" max="1.5" step="0.1" 
                  bind:value={legAmplitude} 
                  on:input={onAmplitudeChange}
                  style="width: 100%;" 
              />
          </div>

          {#if isWebcamRunning}
              <button class="add-btn" on:click={addNewFrame} style="background: #27ae60; margin-top: 5px;">
                  Capture Webcam Pose
              </button>
          {/if}
       </div> -->

      <div class="control-group" style="background: #444; padding: 10px; border-radius: 4px; margin-bottom: 15px;">
          <label style="color:white; margin-bottom:5px;">Project Files</label>
          <div style="display: flex; gap: 5px;">
              <button class="small-btn" on:click={exportProject}>Save JSON</button>
              <button class="small-btn" on:click={triggerImport}>Load JSON</button>
          </div>
          <input type="file" accept=".json" bind:this={fileInput} on:change={handleFileSelect} style="display: none;" />
      </div>

      <div class="control-group">
        <label>Internal Res</label>
        <div style="display:flex; gap:5px;">
          <input type="number" bind:value={renderWidth} on:change={updateResolution} style="width:50px"/>
          <span>x</span>
          <input type="number" bind:value={renderHeight} on:change={updateResolution} style="width:50px"/>
        </div>
      </div>
      
      <div class="control-group">
        <label>Active Part</label>
        <div class="info-box">
           {selectedBoneIndex > -1 ? bones[selectedBoneIndex].name : 'None'}
        </div>
      </div>
      
      <div class="control-group">
        <label>Sensitivity</label>
        <input type="range" min="0.001" max="0.1" step="0.001" bind:value={rotationSpeed} />
      </div>

      <div class="control-group">
        <label>
          <input type="checkbox" bind:checked={viewportNoLighting} />
          No Lighting
        </label>
      </div>

      <hr />

      <div style="display:flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
          <h3>Timeline</h3>
          <span style="font-size:11px; color:#888;">
              {activeFrameIndex > -1 ? `Frame ${activeFrameIndex + 1}` : 'Free Pose'}
          </span>
      </div>
      
      <div class="control-group" style="flex-direction: row; gap: 5px; margin-bottom: 10px;">
        <button 
          on:click={togglePlay} 
          style="flex: 1; background: {isPlaying ? '#e74c3c' : '#2ecc71'};"
        >
          {isPlaying ? 'STOP' : 'PLAY LOOP'}
        </button>
        
        <div style="flex:1; display:flex; flex-direction:column;">
           <label style="margin-bottom:2px;">FPS: {playbackSpeed}</label>
           <input type="range" min="1" max="12" step="0.5" bind:value={playbackSpeed} />
        </div>
      </div>

      <div class="action-row">
          {#if activeFrameIndex > -1}
              <button class="update-btn" on:click={updateCurrentFrame} disabled={isPlaying || isWebcamRunning || isProcessingVideo}>
                  Update Frame
              </button>
          {/if}
          {#if !isWebcamRunning}
              <button class="add-btn" on:click={addNewFrame} disabled={isPlaying || isProcessingVideo}>
                  {activeFrameIndex > -1 ? '+ Copy' : '+ New Frame'}
              </button>
          {/if}
      </div>

      <div class="timeline">
        {#each keyframes as frame, i (frame.id)}
          <div 
              class="keyframe-wrapper {dragOverIndex === i ? 'drag-over' : ''} {draggingFrameIndex === i ? 'is-dragging' : ''}"
              draggable={!isPlaying && !isWebcamRunning}
              on:dragstart={(e) => handleDragStart(e, i)}
              on:dragover={(e) => handleDragOver(e, i)}
              on:drop={(e) => handleDrop(e, i)}
          >
              <button 
                  class="keyframe {i === activeFrameIndex ? 'active' : ''}" 
                  on:click={() => selectFrame(i)} 
                  disabled={isPlaying || isWebcamRunning || isProcessingVideo}
              >
                  {i + 1}
              </button>
              {#if !isPlaying && !isWebcamRunning && !isProcessingVideo}
                  <div class="delete-mini" on:click|stopPropagation={() => deleteFrame(i)} title="Delete">×</div>
              {/if}
          </div>
        {/each}
        
        {#if keyframes.length === 0}
          <div class="empty-state">No frames.</div>
        {/if}
      </div>

      <button class="clear-frame" on:click={() => { keyframes = []; activeFrameIndex = -1; }} disabled={isPlaying}>
          Clear All Frames
      </button>
    </div> <div class="sidebar-footer">
      <button class="export-btn" on:click={openExportModal} disabled={isExporting}>
          {isExporting ? 'Generating...' : 'Download Sprite Sheet'}
      </button>
    </div>
  </div>
  
  {#if showExportModal}
      <div class="modal-overlay">
          <div class="modal-content">
              <h3>Export Options</h3>
              
              <div class="modal-row">
                  <label>Resolution</label>
                  <div style="display:flex; gap:5px;">
                      <input type="number" bind:value={exportSettings.width} style="width:60px;" />
                      <span>x</span>
                      <input type="number" bind:value={exportSettings.height} style="width:60px;" />
                  </div>
              </div>
              
              <div class="modal-row">
                  <label>Layout</label>
                  <select bind:value={exportSettings.layout} style="background:#444; color:white; border:1px solid #555; padding:4px;">
                      <option value="grid">Grid (Standard)</option>
                      <option value="horizontal">Horizontal Strip</option>
                      <option value="vertical">Vertical Strip</option>
                  </select>
              </div>
              
              <div class="modal-row">
                  <label>
                      <input type="checkbox" bind:checked={exportSettings.flatShading} />
                      Flat Shading (No Light)
                  </label>
              </div>
              
              <div class="modal-row">
                  <label>
                      <input type="checkbox" bind:checked={exportSettings.transparentBackground} />
                      Transparent Background
                  </label>
              </div>
              

              <div class="modal-row">
                  <label>
                      <input type="checkbox" bind:checked={exportSettings.normalizeHeadColor} />
                      Normalize Head Color
                  </label>
                  <div class="modal-row">
              </div>
                  <label>
                      <input type="checkbox" bind:checked={exportSettings.normalizeBackground} />
                      Normalize Background
                  </label>
              </div>
              
              <div class="modal-row">
                  <label>
                      <input type="checkbox" bind:checked={exportSettings.useBoundingBox} />
                      Center Crop (from center)
                  </label>
              </div>
              
              {#if exportSettings.useBoundingBox}
                  <div class="modal-row">
                      <label>Crop Width</label>
                      <input type="number" bind:value={exportSettings.boundingBoxWidth} style="width:80px;" />
                  </div>
                  <div class="modal-row">
                      <label>Crop Height</label>
                      <input type="number" bind:value={exportSettings.boundingBoxHeight} style="width:80px;" />
                  </div>
                  <div class="modal-row">
                      <label>Offset X</label>
                      <input type="number" bind:value={exportSettings.boundingBoxOffsetX} style="width:80px;" />
                  </div>
                  <div class="modal-row">
                      <label>Offset Y</label>
                      <input type="number" bind:value={exportSettings.boundingBoxOffsetY} style="width:80px;" />
                  </div>
              {/if}
              
              <div class="modal-actions">
                  <button on:click={() => showExportModal = false} style="background:#555;">Cancel</button>
                  <button on:click={doExport} style="background:#27ae60;">Export</button>
              </div>
          </div>
      </div>
  {/if}
</div>

<canvas bind:this={exportCanvas} style="display: none;"></canvas>

<style>
  :global(body) { margin: 0; background: #1a1a1a; color: #ccc; font-family: monospace; overflow: hidden; user-select: none; }
  
  .layout { display: grid; grid-template-columns: 1fr 280px; height: 100vh; }
  .viewport { position: relative; background: #000; display: flex; align-items: center; justify-content: center; border-right: 2px solid #333; }
  .viewport canvas { image-rendering: pixelated; max-width: 95%; max-height: 95%; border: 1px dashed #444; }
  
  .mocap-overlay {
      position: absolute;
      top: 15px;
      right: 15px;
      width: 160px;
      height: 120px;
      background: #000;
      border: 2px solid #3498db;
      border-radius: 4px;
      overflow: hidden;
      z-index: 50;
  }
  .mocap-canvas { width: 100%; height: 100%; object-fit: contain; }
  .mocap-close {
      position: absolute; top: 0; right: 0;
      background: rgba(0,0,0,0.5); color: white;
      border: none; cursor: pointer; padding: 2px 6px;
      font-size: 14px;
  }

  .shape-hud {
      position: absolute;
      top: 50px;
      left: 15px;
      background: rgba(0,0,0,0.7);
      padding: 10px;
      border-radius: 4px;
      z-index: 20;
      width: 140px;
  }
  .shape-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 5px; }
  .shape-label { font-size: 11px; color: #aaa; text-transform: uppercase; }
  .shape-controls { display: flex; align-items: center; gap: 5px; }
  .shape-value { font-size: 11px; color: #fff; width: 50px; text-align: center; }
  .arrow-btn { background: #444; border: none; color: white; width: 15px; height: 15px; font-size: 10px; display: flex; align-items: center; justify-content: center; cursor: pointer; border-radius: 2px; padding: 0; }
  .arrow-btn:hover { background: #666; }

  .color-legend {
      position: absolute;
      top: 50px;
      right: 15px;
      background: rgba(0,0,0,0.7);
      padding: 10px;
      border-radius: 4px;
      z-index: 20;
      width: 160px;
      max-height: 600px;
      overflow-y: auto;
  }
  .legend-title { font-size: 12px; color: #fff; font-weight: bold; margin-bottom: 8px; text-transform: uppercase; }
  .legend-item { display: flex; flex-direction: column; gap: 4px; margin-bottom: 8px; padding: 6px; border-radius: 2px; border: 1px solid transparent; transition: all 0.2s; }
  .legend-item.selected { background: rgba(255,255,255,0.2); border-color: #fff; }
  .legend-item-header { display: flex; align-items: center; gap: 6px; cursor: pointer; }
  .legend-label { font-size: 10px; color: #ccc; flex: 1; }
  .legend-item:hover .legend-label { color: #fff; }
  .legend-color-input { width: 16px; height: 16px; border: none; border-radius: 2px; cursor: pointer; padding: 0; flex-shrink: 0; }
  .legend-color-input:hover { opacity: 0.8; }
  .legend-hex-input { width: 100%; font-size: 9px; padding: 3px; background: #222; border: 1px solid #555; color: #fff; border-radius: 2px; font-family: monospace; text-transform: uppercase; }
  .legend-hex-input:focus { border-color: #3498db; outline: none; }
  .legend-hex-input::placeholder { color: #666; }

  .recording-indicator { position: absolute; top: 15px; right: 185px; color: #e74c3c; font-weight: bold; background: rgba(0,0,0,0.8); padding: 5px 10px; border-radius: 4px; animation: pulse 1s infinite; pointer-events: none; }
  @keyframes pulse { 0% { opacity: 1; } 50% { opacity: 0.5; } 100% { opacity: 1; } }

  .controls-hint { position: absolute; bottom: 15px; left: 15px; background: rgba(0,0,0,0.7); color: #fff; padding: 10px; border-radius: 4px; pointer-events: none; font-size: 11px; border-left: 3px solid #e24a4a; line-height: 1.5; }
  .resolution-badge { position: absolute; top: 15px; left: 15px; background: rgba(0,0,0,0.8); color: #0f0; padding: 5px 10px; font-size: 12px; border-radius: 4px; pointer-events: none; }
  
  .sidebar { 
    background: #222; 
    display: flex; 
    flex-direction: column; 
    height: 100vh;
    overflow: hidden; 
  }
  
  .sidebar-scroll-area {
    flex: 1;
    overflow-y: auto; 
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 15px;
  }

  .sidebar-footer {
    padding: 20px;
    background: #222;
    border-top: 1px solid #444;
    z-index: 10;
  }

  h2, h3 { margin: 0 0 5px 0; color: #fff; font-size: 16px; }
  .control-group { display: flex; flex-direction: column; gap: 5px; }
  .info-box { padding: 10px; background: #333; border-radius: 4px; color: #fff; border: 1px solid #444; }
  label { font-size: 12px; color: #888; text-transform: uppercase; letter-spacing: 1px; }
  input { background: #333; border: 1px solid #444; color: white; padding: 8px; border-radius: 4px; }
  button { background: #444; color: white; border: none; padding: 8px 12px; cursor: pointer; border-radius: 4px; font-size: 12px; }
  button:hover:not(:disabled) { background: #555; }
  button:disabled { opacity: 0.5; cursor: not-allowed; }
  .small-btn { font-size: 11px; padding: 5px; width: 100%; background: #666; }
  .action-row { display: flex; gap: 5px; margin-bottom: 5px; }
  .update-btn { flex: 1; background: #e67e22; color: white; }
  .add-btn { flex: 1; background: #3498db; color: white; }

  .timeline { display: flex; flex-wrap: wrap; gap: 8px; min-height: 50px; }
  .keyframe-wrapper { position: relative; transition: transform 0.2s; }
  .keyframe-wrapper.is-dragging { opacity: 0.3; }
  .keyframe-wrapper.drag-over { transform: scale(1.1); }
  .keyframe-wrapper.drag-over .keyframe { border-color: #fff; background: #555; }
  .keyframe { width: 35px; height: 35px; padding: 0; display: flex; align-items: center; justify-content: center; background: #333; border: 1px solid #555; cursor: grab; }
  .keyframe:active { cursor: grabbing; }
  .keyframe.active { background: #e67e22; border-color: #fff; box-shadow: 0 0 5px rgba(230, 126, 34, 0.5); }
  .delete-mini { position: absolute; top: -5px; right: -5px; width: 14px; height: 14px; background: #e74c3c; color: white; border-radius: 50%; font-size: 10px; display: flex; align-items: center; justify-content: center; cursor: pointer; opacity: 0; transition: opacity 0.2s; }
  .keyframe-wrapper:hover .delete-mini { opacity: 1; }
  .empty-state { font-size: 12px; color: #666; padding: 10px; width: 100%; text-align: center; border: 1px dashed #444; }
  .clear-frame { width: 100%; background: #c0392b; margin-top: 5px; }
  .export-btn { width: 100%; background: linear-gradient(45deg, #ff6b6b, #feca57); color: #000; font-weight: bold; padding: 15px; text-transform: uppercase; }
  hr { width: 100%; border: 0; border-top: 1px solid #444; margin: 5px 0; }
  
  .modal-overlay { position: fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.7); display:flex; align-items:center; justify-content:center; z-index:100; }
  .modal-content { background: #333; padding: 20px; border-radius: 8px; min-width: 300px; box-shadow: 0 10px 25px rgba(0,0,0,0.5); }
  .modal-content h3 { margin-top: 0; color: white; border-bottom: 1px solid #555; padding-bottom: 10px; }
  .modal-row { margin-bottom: 15px; display: flex; justify-content: space-between; align-items: center; }
  .modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 20px; }
</style>