<template>
    <Lunchbox
        :cameraPosition="[3.2, 4, 3.2]"
        :cameraLook="[0, 0, 0]"
        background="mistyrose"
        shadow
    >
        <!-- ground -->
        <mesh :rotation-x="-Math.PI * 0.5">
            <planeGeometry :args="[side, side, segments, segments]" />
            <shaderMaterial
                :args="[{ vertexShader, fragmentShader, uniforms }]"
            />
        </mesh>

        <!-- south plane -->
        <mesh :position-z="side * 0.5">
            <planeGeometry :args="[side, 1, segments, 1]" />
            <shaderMaterial
                ref="south"
                :args="[
                    {
                        vertexShader: nsVertex,
                        fragmentShader: sidesFragment,
                        uniforms,
                    },
                ]"
            />
        </mesh>

        <!-- east plane -->
        <mesh :position-x="side * 0.5" :rotation-y="Math.PI * 0.5">
            <planeGeometry :args="[side, 1, segments, 1]" />
            <shaderMaterial
                ref="east"
                :args="[
                    {
                        vertexShader: ewVertex,
                        fragmentShader: sidesFragment,
                        uniforms,
                    },
                ]"
            />
        </mesh>

        <!-- shape -->
        <mesh
            :position-x="0.5"
            :position-y="2 + Math.sin(now * 0.001) * 0.1"
            :position-z="1"
            :rotation-y="Math.sin(now * 0.001) * 0.2"
            :scale="0.25"
        >
            <octahedronGeometry />
            <meshBasicMaterial color="white" />

            <mesh :scale="1.001">
                <octahedronGeometry />
                <meshBasicMaterial color="black" wireframe />
            </mesh>
        </mesh>
    </Lunchbox>
</template>

<script lang="ts" setup>
import { ref } from '@vue/reactivity'
import { Lunch, onBeforeRender } from 'lunchboxjs'
import { noise } from './noise'
import * as THREE from 'three'

// based on https://twitter.com/erindale_xyz/status/1501378871068921856

const side = 3
const segments = 128
const uvScale = 2
const now = ref(Date.now())
const color1 = `vec3(0., 0.7, 0.7)`
const color2 = `vec3(1., 0., 0.7)`
const vHeightFunction = `vHeight = (snoise(vUv * ${uvScale}.) * 0.5 + 0.5) + (snoise(vUv * 100.) * 0.5 + 0.5) * 0.1;`

const east = ref<Lunch.Node<THREE.ShaderMaterial>>()
const south = ref<Lunch.Node<THREE.ShaderMaterial>>()

onBeforeRender(() => {
    uniforms.time.value += 0.001666 * 4
    now.value = Date.now()
})

const uniforms = {
    time: {
        value: 0.0,
    },
}

const vertexShader = `
${noise}
varying float vHeight;
varying vec2 vUv;
uniform float time;
varying vec2 screenUv;

void main() {
    vUv = uv + vec2(time, 0.);
    vHeight = (snoise(vUv * ${uvScale}.) * 0.5 + 0.5);
    // vHeight = mix(vHeight, vHeight + (snoise(vUv * 10.) * 0.5 + 0.5) * 0.1, min(0.5 - (abs(vec2(0.5) - uv)).x, 0.5 - (abs(vec2(0.5) - uv)).y));
    vec3 pos = position + vec3(0., 0., vHeight);
    vec4 finalPos = projectionMatrix * modelViewMatrix * vec4( pos, 1.0 );
    screenUv = ((finalPos.xyz / finalPos.w) * 0.5 + 0.5).xy;
    gl_Position = finalPos;
}
`

const fragmentShader = `
varying float vHeight;
varying vec2 vUv;
varying vec2 screenUv;

void main() {
    float heightScale = 15.;
    float altitude = vHeight * heightScale;
    float str = step(0.2, pow(fract(altitude) - 0.5, 10.) * 400.);
    vec3 color = mix(${color1}, ${color2}, screenUv.x);
    gl_FragColor = vec4(str * color, 1.);
}
`

// NORTH-SOUTH
// ====================
const nsVertex = `
${noise}
varying float vHeight;
varying vec2 vUv;
varying vec2 standardUv;
uniform float time;
varying vec2 screenUv;

void main() {
    standardUv = uv;
    vUv = uv + vec2(time, 0.);
    vHeight = snoise(vec2(vUv.x * ${uvScale}., 0.)) * 0.5 + 0.5;
    vec3 pos = mix(position, vec3(position.x, vHeight, position.z), vUv.y);
    vec4 finalPos = projectionMatrix * modelViewMatrix * vec4( pos, 1.0 );
    screenUv = ((finalPos.xyz / finalPos.w) * 0.5 + 0.5).xy;
    gl_Position = finalPos;
}
`

// EAST-WEST
// ====================
const ewVertex = `
${noise}
varying float vHeight;
varying vec2 vUv;
uniform float time;
varying vec2 standardUv;
varying vec2 screenUv;

void main() {   
    standardUv = uv;
    vUv = vec2(1. + time, uv.x);    
    vHeight = snoise(vec2(vUv.x * ${uvScale}., vUv.y * ${uvScale}.)) * 0.5 + 0.5;
    vec3 pos = mix(position, vec3(position.x, vHeight, position.z), uv.y);
    vec4 finalPos = projectionMatrix * modelViewMatrix * vec4( pos, 1.0 );
    screenUv = ((finalPos.xyz / finalPos.w) * 0.5 + 0.5).xy;
    gl_Position = finalPos;
}
`

const sidesFragment = `
varying vec2 vUv;
varying vec2 standardUv;
varying vec2 screenUv;

void main() {
    gl_FragColor = vec4(mix(${color1}, ${color2}, screenUv.x), 1.);
}
`
</script>