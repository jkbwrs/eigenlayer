<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { OrbitControls } from "./orbitControls";
    import { RGBELoader } from "./RGBELoader.js";
    import { FlakesTexture } from "./FlakesTexture.js";
    import seedrandom from "seedrandom";
    import { GLTFExporter } from 'three/examples/jsm/exporters/GLTFExporter.js';
    import './global.css'


    let seedInput: string = "";
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    let camera: THREE.PerspectiveCamera;
    let controls: OrbitControls;
    let envmaploader: THREE.PMREMGenerator;
    let cubeMaterial: THREE.MeshPhysicalMaterial;
    let darkBlue: THREE.MeshPhysicalMaterial;
    let canvas: HTMLCanvasElement;
    let cubes: THREE.Mesh[] = [];

    onMount(() => {
        init();
    });

    function darkerBorder(color: THREE.Color, darknessFactor: number = 0.1): THREE.Color {
        const r = Math.max(0, color.r - darknessFactor);
        const g = Math.max(0, color.g - darknessFactor);
        const b = Math.max(0, color.b - darknessFactor);
        return new THREE.Color(r, g, b);
    }

    function generateBlocks(seed: string): void {
        cubes.forEach((cube) => scene.remove(cube));
        cubes = [];
        Math.random = seedrandom(seed);

        let complexityFactor = Math.ceil(
            (new Set(seedInput).size * seedInput.length) / 10
        );
        complexityFactor = Math.max(5, Math.min(complexityFactor, 30));

        generateBlueBlocks();
        generateIsland(complexityFactor);
    }

    function generateBlueBlocks(): void {
        for (let i = -4; i <= 4; i++) {
            for (let j = -4; j <= 4; j++) {
                let height = 20 + Math.random() * 20;
                let cubeGeometry = new THREE.BoxGeometry(30, height, 30);
                let cubeMesh = new THREE.Mesh(cubeGeometry, cubeMaterial);
                let edges = new THREE.EdgesGeometry(cubeGeometry);
                let line = new THREE.LineSegments(
                    edges,
                    new THREE.LineBasicMaterial({
                        color: darkerBorder(cubeMaterial.color),
                    })
                );
                cubeMesh.add(line);
                cubeMesh.position.set(i * 30, height / 2, j * 30);
                cubeMesh.receiveShadow = true;
                cubeMesh.castShadow = true;
                scene.add(cubeMesh);
                cubes.push(cubeMesh);
            }
        }
    }

    function generateIsland(complexityFactor: number): void {
        let islandMatrix = Array(9)
            .fill()
            .map(() =>
                Array(5)
                    .fill()
                    .map(() => Array(9).fill(false))
            );
        let x = 4,
            z = 4,
            y = 0;
        islandMatrix[x][y][z] = true;

        for (let i = 0; i < complexityFactor; i++) {
            let width = 32.5;
            let depth = 32.5;
            let height = 20 + Math.random() * 20;

            let cubeGeometry = new THREE.BoxGeometry(width, height, depth);
            let cubeMesh = new THREE.Mesh(cubeGeometry, darkBlue);
            let edges = new THREE.EdgesGeometry(cubeGeometry);
            let line = new THREE.LineSegments(
                edges,
                new THREE.LineBasicMaterial({
                    color: darkerBorder(darkBlue.color),
                })
            );

            cubeMesh.add(line);
            cubeMesh.position.set(
                (x - 4) * width,
                y * height + 30,
                (z - 4) * depth
            ); // Subtract 4 to center the island
            cubeMesh.receiveShadow = true;
            cubeMesh.castShadow = true;

            scene.add(cubeMesh);
            cubes.push(cubeMesh);

            let possiblePositions = [];

            if (x > 0 && !islandMatrix[x - 1][y][z])
                possiblePositions.push({ x: x - 1, y, z });
            if (x < 8 && !islandMatrix[x + 1][y][z])
                possiblePositions.push({ x: x + 1, y, z });
            if (z > 0 && !islandMatrix[x][y][z - 1])
                possiblePositions.push({ x, y, z: z - 1 });
            if (z < 8 && !islandMatrix[x][y][z + 1])
                possiblePositions.push({ x, y, z: z + 1 });
            if (y < 4 && !islandMatrix[x][y + 1][z])
                possiblePositions.push({ x, y: y + 1, z });

            let newPosition =
                possiblePositions[
                    Math.floor(Math.random() * possiblePositions.length)
                ];
            if (newPosition) {
                x = newPosition.x;
                y = newPosition.y;
                z = newPosition.z;
                islandMatrix[x][y][z] = true;
            } else {
                break;
            }
        }
    }

    function onSeedInput(e: Event): void {
        seedInput = (e.target as HTMLInputElement).value;
        generateBlocks(seedInput);
    }

    function init(): void {
        scene = new THREE.Scene();

        renderer = new THREE.WebGLRenderer({
            canvas: canvas,
            alpha: true,
            antialias: true,
            shadowMap: { type: THREE.PCFSoftShadowMap },
            shadowMapBias: 0.006,
            shadowMapDarkness: 2,
            shadowMapWidth: 8192,
            shadowMapHeight: 8192,
        });

        renderer.shadowMap.enabled = true;
        renderer.setSize(650, 650);

        camera = new THREE.PerspectiveCamera(75, 1, 0.1, 1000);
        camera.position.set(-190, 190, -190);
        camera.lookAt(new THREE.Vector3(0, 0, 0));

        controls = new OrbitControls(camera, renderer.domElement);
        controls.enableZoom = true;
        controls.autoRotate = true;
        controls.autoRotateSpeed = 1;
        controls.maxPolarAngle = 1.5;

        let ambientLight = new THREE.AmbientLight(0xeeeeee, 0);
        scene.add(ambientLight);

        let pointlight = new THREE.PointLight(0xffffff, 0.6);
        pointlight.position.set(0, 260, 260);
        pointlight.castShadow = true;
        pointlight.shadow.mapSize.width = 4096;
        pointlight.shadow.mapSize.height = 4096;
        scene.add(pointlight);

        let secondaryLight = new THREE.PointLight(0xffffff, 0.1);
        secondaryLight.position.set(0, 260, -260);
        secondaryLight.castShadow = true;
        secondaryLight.shadow.mapSize.width = 4096;
        secondaryLight.shadow.mapSize.height = 4096;
        scene.add(secondaryLight);

        let texture = new THREE.CanvasTexture(new FlakesTexture());
        texture.wrapS = THREE.RepeatWrapping;
        texture.wrapT = THREE.RepeatWrapping;
        texture.repeat.x = 10;
        texture.repeat.y = 6;
        envmaploader = new THREE.PMREMGenerator(renderer);

        new RGBELoader().load("../hi.hdr" /* Demo HDR */, function (hdrmap) {
            let envmap = envmaploader.fromCubemap(hdrmap);

            cubeMaterial = new THREE.MeshPhysicalMaterial({
                clearcoat: 1.0,
                clearcoatRoughness: 0.9,
                metalness: 0.4,
                roughness: 0.2,
                color: new THREE.Color("#bedbff"),
                normalMap: texture,
                normalScale: new THREE.Vector2(0.1, 0.1),
                envMap: envmap.texture,
            });

            darkBlue = new THREE.MeshPhysicalMaterial({
                clearcoat: 1.0,
                clearcoatRoughness: 0.2,
                metalness: 0.4,
                roughness: 0.2,
                color: new THREE.Color("#1a0c6d"),
                normalMap: texture,
                normalScale: new THREE.Vector2(0.1, 0.1),
                envMap: envmap.texture,
            });
            renderer.outputEncoding = THREE.sRGBEncoding;
            renderer.toneMapping = THREE.ACESFilmicToneMapping;
            renderer.toneMappingExposure = 1.2;

            generateBlocks(seedInput);
        });

        animate();
    }

    function animate(): void {
        controls.update();
        renderer.render(scene, camera);
        requestAnimationFrame(animate);
    }

    function exportGLTF(): void {
        const exporter = new GLTFExporter()

        exporter.parse(scene, (gltf) => { 
            const data = JSON.stringify(gltf);
            const blob = new Blob([data], { type: "application/octet-stream" }); 
            const url = URL.createObjectURL(blob);

            const link = document.createElement("a");
            link.href = url; 
            link.download = "scene.gltf";
            link.click(); 

            setTimeout(() => {
                URL.revokeObjectURL(url); 
            }, 1);
        });
    }

</script>

<div>
    <input
        type="text"
        bind:value={seedInput}
        on:input={onSeedInput}
        placeholder="Seed"
    />
    <button on:click={exportGLTF} class="export">Export as GLTF</button>
</div>

<canvas bind:this={canvas} />