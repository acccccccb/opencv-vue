<template>
    <div>
        <div class="inputoutput">
            <div class="caption"><input @change="inputChange" type="file" /></div>
        </div>
        <div>peopleNum: {{ peopleNum }}</div>
        <div class="inputoutput">
            <canvas id="canvasInput" v-show="false"></canvas>
            <canvas id="canvasOutput"></canvas>
        </div>
    </div>
</template>
<script>
    import cv from '../assets/js/opencv.js';
    import Utils from '../assets/js/utils.js';
    let utils = new Utils('errorMessage');

    export default {
        name: 'OpenCvJs',
        data() {
            return {
                modules: ['haarcascade_frontalface_alt2', 'haarcascade_eye'],
                peopleNum: 0,
            };
        },
        created() {
            // 加载模型
            this.modules.forEach(async (item) => {
                await this.loadCascade(item);
            });
        },
        methods: {
            loadCascade(path) {
                return new Promise((reslove) => {
                    utils.createFileFromUrl(`${path}.xml`, `cascade/${path}.xml`, () => {
                        reslove();
                    });
                });
            },
            cargoCascade(model) {
                const cascadeClassifier = new cv.CascadeClassifier();
                cascadeClassifier.load(`${model}.xml`);
                return cascadeClassifier;
            },
            markTarget(
                src,
                modules,
                params = {
                    color: [255, 0, 0, 255],
                    scaleFactor: 1.1,
                    minNeighbors: 3,
                    flags: 0,
                },
                roi,
                isLast = false
            ) {
                return new Promise((reslove) => {
                    let gray = new cv.Mat();
                    cv.cvtColor(src, gray, cv.COLOR_RGBA2GRAY, 0);
                    let target = new cv.RectVector();
                    let cascade = this.cargoCascade(modules);

                    let msize = new cv.Size(0, 0);
                    cascade.detectMultiScale(gray, target, params.scaleFactor, params.minNeighbors, params.flags, msize, msize);
                    if (target.size() === 0) {
                        reslove([null, 0]);
                    }
                    for (let i = 0; i < target.size(); ++i) {
                        let roiGray = roi ? roi : gray.roi(target.get(i));
                        let roiSrc = src.roi(target.get(i));
                        let point1 = new cv.Point(target.get(i).x, target.get(i).y);
                        let point2 = new cv.Point(target.get(i).x + target.get(i).width, target.get(i).y + target.get(i).height);
                        cv.rectangle(src, point1, point2, params.color);
                        if (!isLast) {
                            reslove([roiGray, target.size()]);
                        } else {
                            if (i === target.size() - 1) {
                                roiGray.delete();
                                reslove([null, target.size()]);
                            }
                        }
                        roiSrc.delete();
                    }
                    cascade.delete();
                    target.delete();
                    gray.delete();
                });
            },
            async inputChange(e) {
                if (e.target.files.length === 0) {
                    return false;
                }
                const file = e.target.files[0];
                await utils.loadImageToCanvas(URL.createObjectURL(file), 'canvasInput');
                let src = cv.imread('canvasInput');

                const faceParams = {
                    color: [255, 0, 0, 255],
                    scaleFactor: 1.1,
                    minNeighbors: 3,
                    flags: 0,
                };
                const eyesParams = {
                    color: [0, 0, 255, 255],
                    scaleFactor: 1.1,
                    minNeighbors: 7,
                    flags: 0,
                };
                const [roiGray, peopleNum] = await this.markTarget(src, this.modules[0], faceParams, null);
                this.peopleNum = peopleNum;
                await this.markTarget(src, this.modules[1], eyesParams, roiGray, null, true);

                cv.imshow('canvasOutput', src);
                src.delete();
            },
        },
    };
</script>
