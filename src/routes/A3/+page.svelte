<script lang="ts">
    import { onMount, onDestroy } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";
    import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
    import { base } from "$app/paths";
    import * as d3 from "d3";
    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";

    // Stacked data: each year has counts per genre
    type TStackRow = {
        year: string;
        Comedy: number;
        Romance: number;
        Drama: number;
    };

    let stackData: TStackRow[] = $state([]);
    let loadError = $state("");
    const genres = ["Comedy", "Romance", "Drama"] as const;
    const genreColors: Record<string, number> = {
        Comedy: 0xf4a261,  // warm orange
        Romance: 0xe76f51,  // coral red
        Drama: 0x2a9d8f,   // teal
    };

    // === TODO-1: Load Data ===
    // Loads summer_movies.csv and aggregates real movie counts per
    // release year for the three tracked genres (Comedy, Romance, Drama).
    async function loadCsv() {
        try {
            const rows = await d3.csv(base + "/summer_movies.csv", (row) => {
                const yearNumber = Number(row.year);
                return {
                    year: new Date(yearNumber, 0, 1),
                    genres: !row.genres || row.genres === "NA"
                        ? []
                        : row.genres.split(",").map((g) => g.trim()),
                };
            });

            const valid = rows.filter((d) => Number.isFinite(d.year.getFullYear()));

            const grouped = d3.rollup(
                valid,
                (items) => {
                    const counts = { Comedy: 0, Romance: 0, Drama: 0 };
                    for (const item of items) {
                        for (const genre of genres) {
                            if (item.genres.includes(genre)) {
                                counts[genre] += 1;
                            }
                        }
                    }
                    return counts;
                },
                (d) => d.year.getFullYear(),
            );

            stackData = Array.from(grouped, ([year, counts]) => ({
                year: String(year),
                ...counts,
            })).sort((a, b) => Number(a.year) - Number(b.year));

            console.log(
                `Stack data: ${stackData.length} years, ` +
                `${stackData.reduce((sum, d) => sum + d.Comedy + d.Romance + d.Drama, 0)} unit objects`,
                stackData,
            );
        } catch (e) {
            loadError = e instanceof Error ? e.message : String(e);
            stackData = [];
        }
    }
    // === END TODO-1 ===

    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    const FLOOR = -250;
    const REEL_BASE_Y = FLOOR + 700;

    const morphs: Array<THREE.Mesh> = [];
    let mixer: THREE.AnimationMixer;
    const clock = new THREE.Clock();

    const horseMorphs: Array<THREE.Mesh> = [];
    let horseMixer: THREE.AnimationMixer;
    const ORBIT_RX = 850;
    const ORBIT_RZ = 400;

    let filmReel: THREE.Group | null = null;
    let filmReelTime = 0;

    let animating = $state(false);
    let destroyed = false;
    let handleResize: () => void;

    onMount(async () => {
        await loadCsv();
        init(window.innerWidth, window.innerHeight);
    });

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number) {
        // Renderer
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        container.appendChild(renderer.domElement);

        // Camera
        camera = new THREE.PerspectiveCamera(
            23,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            5000,
        );
        camera.position.set(0, 100, 2800);

        // Scene
        scene = new THREE.Scene();
        new THREE.TextureLoader().load(base + "/3d/sky.jpg", (texture) => {
            if (destroyed) {
                texture.dispose();
                return;
            }
            texture.repeat.set(0.8, 1);
            scene.background = texture;
        });

        // Ambient Light
        const ambient = new THREE.AmbientLight(0xffffff);
        scene.add(ambient);

        // Directional Light
        const light = new THREE.DirectionalLight(0xffffff, 3);
        light.position.set(0, 1500, 1000);
        light.castShadow = true;
        Object.assign(light.shadow.camera, {
            top: 2000,
            bottom: -2000,
            left: -2000,
            right: 2000,
            near: 1200,
            far: 2500,
        });
        light.shadow.bias = 0.0001;
        light.shadow.mapSize.width = 2048;
        light.shadow.mapSize.height = 1024;
        scene.add(light);

        // Ground
        addGround(scene, FLOOR, base + "/3d/grasslight-big.jpg");

        // Title + Unit Visualization
        const fontLoader = new FontLoader();
        fontLoader.load(base + "/3d/helvetiker_bold.typeface.json", (font: Font) => {
            if (destroyed) return;

            // Title
            const textGeo = new TextGeometry("summer movies", {
                font: font,
                size: 30,
                depth: 10,
            });
            textGeo.computeBoundingBox();
            const centerOffset =
                -0.5 *
                (textGeo!.boundingBox!.max.x - textGeo!.boundingBox!.min.x);
            const textMaterial = new THREE.MeshStandardMaterial({
                color: 0x449900,
            });
            const titleMesh = new THREE.Mesh(textGeo, textMaterial);
            titleMesh.position.x = centerOffset;
            titleMesh.position.y = FLOOR + 500;
            titleMesh.castShadow = true;
            scene.add(titleMesh);

            createUnits(scene, font);
            createLegend(scene, font);
        });

        // === TODO-2: Add Expressive 3D Objects ===
        // Decorative animated birds (built-in GLTF animation clips)
        const birdModels = [
            {
                path: base + "/3d/Flamingo.glb",
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 100,
                scale: 0.5,
            },
            {
                path: base + "/3d/Flamingo.glb",
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: -100,
                scale: 0.5,
            },
            {
                path: base + "/3d/Parrot.glb",
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 500,
                z: 700,
                scale: 0.5,
            },
        ];
        mixer = loadModels(birdModels, scene, mixer, morphs);

        // Decorative horses orbiting the data (built-in GLTF animation clips)
        horseMixer = new THREE.AnimationMixer(scene);
        const gltfLoader = new GLTFLoader();
        const horseConfigs = [
            { startAngle: 0, orbitSpeed: 0.4 },
            { startAngle: Math.PI, orbitSpeed: 0.4 },
        ];
        horseConfigs.forEach((config) => {
            gltfLoader.load(base + "/3d/Horse.glb", (gltf) => {
                if (destroyed) return;
                const mesh = gltf.scene.children[0].clone() as THREE.Mesh;
                (mesh as any).material = (mesh as any).material.clone();
                mesh.scale.set(0.5, 0.5, 0.5);
                mesh.castShadow = true;

                horseMixer
                    .clipAction(gltf.animations[0], mesh)
                    .setDuration(1)
                    .startAt(-Math.random())
                    .play();

                mesh.userData.angle = config.startAngle;
                mesh.userData.orbitSpeed = config.orbitSpeed;
                mesh.position.set(
                    ORBIT_RX * Math.cos(config.startAngle),
                    FLOOR,
                    ORBIT_RZ * Math.sin(config.startAngle),
                );

                scene.add(mesh);
                horseMorphs.push(mesh);
            });
        });

        // New expressive object: a procedurally built, spinning + bobbing
        // film reel hovering above the scene (custom animation, no built-in clip).
        filmReel = createFilmReel(scene);
        // === END TODO-2 ===

        // Handle resize
        handleResize = () =>
            onWindowResize(
                camera,
                renderer,
                window.innerWidth,
                window.innerHeight,
            );
        window.addEventListener("resize", handleResize);

        renderer.setAnimationLoop(animate);
    }

    function createFilmReel(scene: THREE.Scene): THREE.Group {
        const group = new THREE.Group();
        const metal = new THREE.MeshStandardMaterial({
            color: 0xd4af37,
            metalness: 0.75,
            roughness: 0.28,
        });
        const dark = new THREE.MeshStandardMaterial({
            color: 0x2b2b2b,
            metalness: 0.4,
            roughness: 0.5,
        });

        const ring = new THREE.Mesh(new THREE.TorusGeometry(70, 11, 16, 48), metal);
        ring.castShadow = true;
        group.add(ring);

        const hub = new THREE.Mesh(new THREE.CylinderGeometry(16, 16, 14, 24), dark);
        hub.rotation.x = Math.PI / 2;
        hub.castShadow = true;
        group.add(hub);

        const spokeCount = 6;
        for (let i = 0; i < spokeCount; i++) {
            const angle = (i / spokeCount) * Math.PI * 2;
            const spoke = new THREE.Mesh(new THREE.BoxGeometry(68, 5, 5), metal);
            spoke.rotation.z = angle;
            spoke.position.set(Math.cos(angle) * 34, Math.sin(angle) * 34, 0);
            spoke.castShadow = true;
            group.add(spoke);
        }

        group.position.set(0, REEL_BASE_Y, -180);
        scene.add(group);
        return group;
    }

    // === TODO-3a: Unique geometry + material per genre ===
    function createHeartGeometry(size: number): THREE.BufferGeometry {
        const heartShape = new THREE.Shape();
        heartShape.moveTo(25, 25);
        heartShape.bezierCurveTo(25, 25, 20, 0, 0, 0);
        heartShape.bezierCurveTo(-30, 0, -30, 35, -30, 35);
        heartShape.bezierCurveTo(-30, 55, -10, 77, 25, 95);
        heartShape.bezierCurveTo(60, 77, 80, 55, 80, 35);
        heartShape.bezierCurveTo(80, 35, 80, 0, 50, 0);
        heartShape.bezierCurveTo(35, 0, 25, 25, 25, 25);

        const geometry = new THREE.ExtrudeGeometry(heartShape, {
            depth: 18,
            bevelEnabled: true,
            bevelThickness: 4,
            bevelSize: 3,
            bevelSegments: 2,
        });
        geometry.rotateX(Math.PI);
        geometry.computeBoundingBox();
        const width = geometry.boundingBox!.max.x - geometry.boundingBox!.min.x;
        const scaleFactor = width > 0 ? size / width : 1;
        geometry.scale(scaleFactor, scaleFactor, scaleFactor);
        geometry.center();
        return geometry;
    }

    function createGenreGeometry(genre: string, unitSize: number): THREE.BufferGeometry {
        switch (genre) {
            case "Comedy":
                // Bubbly sphere - light, playful shape
                return new THREE.SphereGeometry(unitSize * 0.5, 16, 12);
            case "Romance":
                // Heart - immediately recognizable for the genre
                return createHeartGeometry(unitSize * 0.9);
            case "Drama":
                // Sharp cone - a more severe, pointed silhouette
                return new THREE.ConeGeometry(unitSize * 0.5, unitSize * 1.05, 10);
            default:
                return new THREE.BoxGeometry(unitSize, unitSize, unitSize);
        }
    }

    function createGenreMaterial(genre: string): THREE.Material {
        const color = genreColors[genre] ?? 0x999999;
        switch (genre) {
            case "Comedy":
                return new THREE.MeshPhysicalMaterial({
                    color,
                    roughness: 0.35,
                    metalness: 0.05,
                    clearcoat: 0.35,
                    clearcoatRoughness: 0.25,
                });
            case "Romance":
                return new THREE.MeshPhysicalMaterial({
                    color,
                    roughness: 0.1,
                    metalness: 0.05,
                    clearcoat: 0.85,
                    clearcoatRoughness: 0.08,
                });
            case "Drama":
                return new THREE.MeshPhysicalMaterial({
                    color,
                    roughness: 0.55,
                    metalness: 0.45,
                    clearcoat: 0,
                });
            default:
                return new THREE.MeshPhysicalMaterial({ color });
        }
    }
    // === END TODO-3a ===

    // === TODO-3b: Unit visualization - one 3D object per movie, stacked per year ===
    function createUnits(scene: THREE.Scene, font: Font) {
        if (stackData.length === 0) return;

        const barMaxWidth = Math.min(stackData.length * 80, 1400);

        const xScale = d3
            .scaleBand()
            .domain(stackData.map((d) => d.year))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.1);

        const bandwidth = xScale.bandwidth();

        // Size each unit so that the tallest yearly stack stays within a
        // fixed visual budget, regardless of how many movies that year has.
        const maxCount = Math.max(
            1,
            ...stackData.map((d) => d.Comedy + d.Romance + d.Drama),
        );
        const unitSize = Math.min(16, Math.max(7, 260 / maxCount));

        stackData.forEach((row) => {
            const xPos = xScale(row.year)! + bandwidth / 2;
            let currentY = FLOOR;

            genres.forEach((genre) => {
                const count = row[genre];
                for (let i = 0; i < count; i++) {
                    const geometry = createGenreGeometry(genre, unitSize);
                    const material = createGenreMaterial(genre);
                    const mesh = new THREE.Mesh(geometry, material);
                    mesh.position.set(xPos, currentY + unitSize / 2, 0);
                    mesh.castShadow = true;
                    mesh.receiveShadow = true;
                    scene.add(mesh);
                    currentY += unitSize;
                }
            });
        });

        // Year labels
        const labelOffsetZ = Math.min(bandwidth * 0.4, unitSize) + 6;
        stackData.forEach((row) => {
            const x = xScale(row.year)! + bandwidth / 2;
            const textGeo = new TextGeometry(row.year, {
                font: font,
                size: 6,
                depth: 2,
            });
            textGeo.computeBoundingBox();
            const textWidth = textGeo.boundingBox!.max.x - textGeo.boundingBox!.min.x;
            const textMat = new THREE.MeshPhysicalMaterial({ color: 0x000000 });
            const textMesh = new THREE.Mesh(textGeo, textMat);
            textMesh.position.set(x - textWidth / 2, FLOOR + 5, labelOffsetZ);
            textMesh.castShadow = true;
            scene.add(textMesh);
        });
    }
    // === END TODO-3b ===

    // === TODO-3c: Legend uses the same genre shapes + materials as the units ===
    function createLegend(scene: THREE.Scene, font: Font) {
        const legendX = 180;
        const legendStartY = FLOOR + 420;
        const spacing = 44;
        const objectSize = 16;

        genres.forEach((genre, i) => {
            const y = legendStartY - i * spacing;

            const geometry = createGenreGeometry(genre, objectSize);
            const material = createGenreMaterial(genre);
            const swatch = new THREE.Mesh(geometry, material);
            swatch.position.set(legendX, y, 0);
            swatch.castShadow = true;
            scene.add(swatch);

            const textGeo = new TextGeometry(genre, {
                font: font,
                size: 10,
                depth: 3,
            });
            const textMat = new THREE.MeshPhysicalMaterial({ color: 0x000000 });
            const textMesh = new THREE.Mesh(textGeo, textMat);
            textMesh.position.set(legendX + 26, y - 5, 0);
            textMesh.castShadow = true;
            scene.add(textMesh);
        });
    }
    // === END TODO-3c ===

    function animate() {
        const delta = clock.getDelta();

        if (animating) {
            mixer.update(delta);
            horseMixer.update(delta);
            filmReelTime += delta;

            // Birds - linear motion
            morphs.forEach((morph) => {
                morph.position.x += morph.speed * delta;
                if (morph.position.x > window.innerWidth / 2) {
                    morph.position.x = -window.innerWidth / 2 - Math.random() * 200;
                }
            });

            // Horses - orbit around the data
            horseMorphs.forEach((horse) => {
                horse.userData.angle += horse.userData.orbitSpeed * delta;
                const angle = horse.userData.angle;
                horse.position.x = ORBIT_RX * Math.cos(angle);
                horse.position.z = ORBIT_RZ * Math.sin(angle);

                // Face the direction of travel
                const lookAngle = angle + 0.1;
                horse.lookAt(
                    ORBIT_RX * Math.cos(lookAngle),
                    horse.position.y,
                    ORBIT_RZ * Math.sin(lookAngle),
                );
            });

            // Film reel - spins on its hole-axis while bobbing up and down
            if (filmReel) {
                filmReel.rotation.z += delta * 1.4;
                filmReel.position.y = REEL_BASE_Y + Math.sin(filmReelTime * 0.9) * 22;
            }
        }

        renderer.render(scene, camera);
    }

    onDestroy(() => {
        destroyed = true;
        if (handleResize) window.removeEventListener("resize", handleResize);

        if (renderer) {
            renderer.setAnimationLoop(null);

            if (scene) {
                if (scene.background instanceof THREE.Texture) {
                    scene.background.dispose();
                }
                scene.traverse((object) => {
                    if (object instanceof THREE.Mesh) {
                        object.geometry?.dispose();
                        const materials = Array.isArray(object.material)
                            ? object.material
                            : [object.material];
                        materials.forEach((material) => {
                            for (const value of Object.values(material)) {
                                if (value instanceof THREE.Texture) value.dispose();
                            }
                            material.dispose();
                        });
                    }
                });
            }

            renderer.dispose();
            renderer.forceContextLoss();
            renderer.domElement.remove();
        }
    });
</script>

<svelte:head>
    <title>CSCI 5609 A3 | 3D Visualization</title>
    <meta name="description" content="3D unit visualization of summer-movie counts by year and genre, built with three.js." />
</svelte:head>

<div
    bind:this={container}
    class="container"
    role="img"
    aria-label="3D scene titled summer movies: stacked unit shapes grouped by release year represent movie counts per genre - spheres for Comedy, hearts for Romance, cones for Drama - with animated flamingos, parrots and horses decorating the scene, plus a spinning film reel overhead."
></div>

<div class="hud">
    <nav class="hud-nav">
        <a href={base + "/"}>Course home</a>
        <a href={base + "/A2"}>A2</a>
    </nav>
    <button class="toggle-btn" onclick={() => (animating = !animating)} aria-pressed={animating}>
        {animating ? "Pause" : "Play"} Animation
    </button>
</div>

{#if loadError}
    <p class="status error" role="alert">Could not load the movie dataset: {loadError}</p>
{:else if !stackData.length}
    <p class="status" aria-live="polite">Loading summer movie dataset…</p>
{/if}

<style>
    div.container {
        width: 100%;
        height: 100%;
    }

    .hud {
        position: fixed;
        top: 10px;
        left: 10px;
        right: 10px;
        display: flex;
        justify-content: space-between;
        align-items: flex-start;
        gap: 10px;
        z-index: 10;
        pointer-events: none;
    }

    .hud-nav {
        display: flex;
        gap: 8px;
        pointer-events: auto;
    }

    .hud-nav a {
        padding: 6px 12px;
        font-size: 12px;
        font-weight: 600;
        border-radius: 20px;
        background: rgba(0, 0, 0, 0.3);
        color: white;
        text-decoration: none;
    }

    .hud-nav a:hover {
        background: rgba(0, 0, 0, 0.8);
    }

    .toggle-btn {
        position: fixed;
        bottom: 10px;
        right: 10px;
        padding: 8px 10px;
        font-size: 12px;
        border: none;
        border-radius: 20px;
        background: rgba(0, 0, 0, 0.30);
        color: white;
        cursor: pointer;
        z-index: 10;
        pointer-events: auto;
    }

    .toggle-btn:hover {
        background: rgba(0, 0, 0, 0.8);
    }

    .status {
        position: fixed;
        bottom: 10px;
        left: 10px;
        margin: 0;
        padding: 8px 14px;
        font-size: 13px;
        border-radius: 10px;
        background: rgba(0, 0, 0, 0.55);
        color: white;
        z-index: 10;
        max-width: min(70vw, 480px);
    }

    .status.error {
        background: rgba(153, 27, 27, 0.85);
    }
</style>
