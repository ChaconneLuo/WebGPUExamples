<script setup lang="ts">
import shader from './shader.wgsl?raw';
if (!navigator.gpu) {
    console.log("WebGPU is not supported!");
} else {
    const init = async () => {
        const adapter = await navigator.gpu.requestAdapter();
        const device = await adapter!.requestDevice();

        const canvas = document.getElementById('webgpu') as HTMLCanvasElement;
        const context = canvas!.getContext('webgpu') as GPUCanvasContext;

        const format = navigator.gpu.getPreferredCanvasFormat();
        context!.configure({
            device: device,
            format: format
        })
        const vertexArray: Float32Array = new Float32Array([
            0.5, 0.5, 0.0,
            0.5, -0.5, 0.0,
            -0.5, 0.5, 0.0,
            -0.5, -0.5, 0.0
        ]);
        const vertexBuffer: GPUBuffer = device.createBuffer({
            size: vertexArray.byteLength,
            usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
        });

        device.queue.writeBuffer(vertexBuffer, 0, vertexArray);

        const pipeline: GPURenderPipeline = device.createRenderPipeline({
            vertex: {
                buffers: [
                    {
                        //间隔3个点，每个点4字节，每隔12字节取一个点
                        arrayStride: 12,
                        attributes: [{
                            shaderLocation: 0,
                            format: 'float32x3',
                            offset: 0
                        }]
                    }
                ],
                module: device.createShaderModule({
                    code: shader
                }),
                entryPoint: "vs_main",
            },
            fragment: {
                module: device.createShaderModule({
                    code: shader
                }),
                entryPoint: "fs_main",
                targets: [
                    {
                        format: format
                    }
                ]
            },
            primitive: {
                topology: 'triangle-strip'
            },
            layout: 'auto'
        });
        const commandEncoder: GPUCommandEncoder = device.createCommandEncoder();

        const renderPass: GPURenderPassEncoder = commandEncoder.beginRenderPass({
            colorAttachments: [{
                view: context.getCurrentTexture().createView(),
                storeOp: 'store',
                loadOp: 'clear',
                clearValue: { r: 0.5, g: 0.5, b: 0.5, a: 1.0 }
            }]
        })

        renderPass.setPipeline(pipeline);
        renderPass.setVertexBuffer(0, vertexBuffer);

        renderPass.draw(4);
        renderPass.end();

        const commandBuffer: GPUCommandBuffer = commandEncoder.finish();

        device.queue.submit([commandBuffer]);
    }
    init();
}
</script>

<template>
    <canvas id="webgpu" width="500" height="500"></canvas>
</template>

<style scoped></style>
