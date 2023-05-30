<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
    import { RGBELoader } from "./RGBELoader.js";
    import { FlakesTexture } from "./FlakesTexture.js";
    import seedrandom from "seedrandom";

    let seedInput: string = "";
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    let camera: THREE.PerspectiveCamera;
    let controls: OrbitControls;
    let envmaploader: THREE.PMREMGenerator;
    let cubeMaterial: THREE.MeshPhysicalMaterial;
    let centerCubeMaterial: THREE.MeshPhysicalMaterial;
    let beigeMaterial: THREE.MeshPhysicalMaterial;
    let canvas: HTMLCanvasElement;
    let cubes: THREE.Mesh[] = [];

    onMount(() => {
        init();
    });

    function darkerBorder(color: THREE.Color, darknessFactor: number = 0.1) {
        const r = Math.max(0, color.r - darknessFactor);
        const g = Math.max(0, color.g - darknessFactor);
        const b = Math.max(0, color.b - darknessFactor);
        return new THREE.Color(r, g, b);
    }

    function generateBlocks(seed: string) {
        cubes.forEach((cube) => scene.remove(cube));
        cubes = [];
        Math.random = seedrandom(seed);

        let complexityFactor = Math.ceil(seed.length / 10);
        let gridSize = 260;
        let blocksize = 32.5;
        let positions = [];

        for (let i = 0; i < complexityFactor; i++) {
            let position;
            do {
                position = {
                    x:
                        Math.floor((Math.random() * gridSize) / blocksize) *
                            blocksize -
                        gridSize / 2,
                    z:
                        Math.floor((Math.random() * gridSize) / blocksize) *
                            blocksize -
                        gridSize / 2,
                };
            } while (
                positions.find(
                    (pos) => pos.x === position.x && pos.z === position.z
                )
            );
            positions.push(position);

            let blockSizeFactor = Math.random() > 0.5 ? 1 : 0.25;

            let cubeGeometry = new THREE.BoxGeometry(
                blocksize * blockSizeFactor,
                blocksize * blockSizeFactor,
                blocksize * blockSizeFactor
            );
            
            let cubeMesh = new THREE.Mesh(cubeGeometry, centerCubeMaterial);
            let edges = new THREE.EdgesGeometry(cubeGeometry);
            let line = new THREE.LineSegments(
                edges,
                new THREE.LineBasicMaterial({
                    color: darkerBorder(centerCubeMaterial.color),
                })
            );

            cubeMesh.add(line);
            cubeMesh.position.set(position.x, blocksize / 2, position.z);
            cubeMesh.receiveShadow = true;
            cubeMesh.castShadow = true;

            scene.add(cubeMesh);
            cubes.push(cubeMesh);
        }

        generateBlueBlocks();
        generateBeigeIsland(complexityFactor);
    }

    function generateBlueBlocks() {
        for (let i = -4; i <= 4; i++) {
            for (let j = -4; j <= 4; j++) {
                let height = 20 + Math.random() * 20;
                let cubeGeometry = new THREE.BoxGeometry(32.5, height, 32.5);
                let cubeMesh = new THREE.Mesh(cubeGeometry, cubeMaterial);
                let edges = new THREE.EdgesGeometry(cubeGeometry);
                let line = new THREE.LineSegments(
                    edges,
                    new THREE.LineBasicMaterial({
                        color: darkerBorder(cubeMaterial.color),
                    })
                );
                cubeMesh.add(line);
                cubeMesh.position.set(i * 32.5, height / 2, j * 32.5);
                cubeMesh.receiveShadow = true;
                cubeMesh.castShadow = true;
                scene.add(cubeMesh);
                cubes.push(cubeMesh);
            }
        }
    }

    function generateBeigeIsland(complexityFactor) {
        let lastPositions = [
            { x: 0, z: 0 },
            { x: 0, z: 0 },
            { x: 0, z: 0 },
            { x: 0, z: 0 },
        ]; // four directions
        for (let i = 0; i < complexityFactor; i++) {
            let width = 32.5;
            let depth = 32.5;
            let height = 20 + Math.random() * 20;

            let cubeGeometry = new THREE.BoxGeometry(width, height, depth);
            let cubeMesh = new THREE.Mesh(cubeGeometry, beigeMaterial);
            let edges = new THREE.EdgesGeometry(cubeGeometry);
            let line = new THREE.LineSegments(
                edges,
                new THREE.LineBasicMaterial({
                    color: darkerBorder(beigeMaterial.color),
                })
            );

            cubeMesh.add(line);

            let direction = i % 4;

            if (direction === 0) {
                lastPositions[direction].x += width;
            } else if (direction === 1) {
                lastPositions[direction].z += depth;
            } else if (direction === 2) {
                lastPositions[direction].x -= width;
            } else {
                lastPositions[direction].z -= depth;
            }
            cubeMesh.position.set(
                lastPositions[direction].x,
                height + 30,
                lastPositions[direction].z
            );

            cubeMesh.receiveShadow = true;
            cubeMesh.castShadow = true;

            scene.add(cubeMesh);
            cubes.push(cubeMesh);
        }
    }

    function onSeedInput(e) {
        seedInput = e.target.value;
        generateBlocks(seedInput);
    }

    function init() {
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

        let ambientLight = new THREE.AmbientLight(0xeeeeee, 0.);
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

        new RGBELoader()
            .load("../hi.hdr" /* Demo HDR */, function (hdrmap) {
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

                centerCubeMaterial = new THREE.MeshPhysicalMaterial({
                    clearcoat: 1.0,
                    clearcoatRoughness: 0.5,
                    metalness: 0.4,
                    roughness: 0.2,
                    color: new THREE.Color("#fff9f3"),
                    normalMap: texture,
                    normalScale: new THREE.Vector2(0.1, 0.1),
                    envMap: envmap.texture,
                });

                beigeMaterial = new THREE.MeshPhysicalMaterial({
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

    function animate() {
        controls.update();
        renderer.render(scene, camera);
        requestAnimationFrame(animate);
    }
</script>

<input
    type="text"
    bind:value={seedInput}
    on:input={onSeedInput}
    placeholder="Seed"
/>

<canvas bind:this={canvas} />
