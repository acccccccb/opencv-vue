<template>
    <div>
        <div class="inputoutput">
            <input type="text" v-model="imgSrc" />
            <button @click="loadImg">加载图片</button>
            <button @click="$refs.fileInput.click()">选择图片</button>
            <input ref="fileInput" @change="inputChange" v-show="false" type="file" />
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
        name: 'OCImg',
        data() {
            return {
                modules: ['haarcascade_frontalface_alt2', 'haarcascade_eye'],
                peopleNum: 0,
                imgSrc: 'http://placekitten.com/500/500',
                maxWidth: 800,
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
            async loadImg() {
                const maxWidth = 1200;
                let canvas = document.getElementById('canvasInput');
                let ctx = canvas.getContext('2d');
                let img = new Image();
                img.crossOrigin = 'anonymous';
                img.onload = () => {
                    const p = img.width / img.height;
                    const width = img.width > maxWidth ? maxWidth : img.width;
                    const height = width / p;
                    canvas.width = width;
                    canvas.height = height;
                    ctx.drawImage(img, 0, 0, width, height);
                    setTimeout(() => {
                        this.analysisImg('canvasInput');
                    }, 16);
                };
                img.src = this.imgSrc;
                // this.analysisImg('img');
            },
            async inputChange(e) {
                if (e.target.files.length === 0) {
                    return false;
                }
                const file = e?.target?.files[0];
                await utils.loadImageToCanvas(URL.createObjectURL(file), 'canvasInput');
                this.analysisImg('canvasInput');
            },
            async analysisImg(img) {
                let src = cv.imread(img);
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

                console.log('src', src.size());
                let imgWidth = src.size().width;
                let imgHeight = src.size().height;
                if (imgWidth > this.maxWidth) {
                    const p = imgWidth / imgHeight;
                    imgWidth = this.maxWidth;
                    imgHeight = imgWidth / p;
                }

                let dst = new cv.Mat();
                let dsize = new cv.Size(imgWidth, imgHeight);

                cv.resize(src, dst, dsize, 0, 0, cv.INTER_AREA);
                cv.imshow('canvasOutput', dst);
                src.delete();
            },
        },
    };
</script>
