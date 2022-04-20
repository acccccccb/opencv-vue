<template>
    <div>
        <div class="videooutput">
            <video id="videoInput" src="../assets/media/mouse3.mp4" width="180" height="320" controls autoplay loop></video>
            <canvas id="canvasOutput" width="180" height="320"></canvas>
            <button @click="init">init</button>
        </div>
    </div>
</template>

<script>
    import cv from '../assets/js/opencv.js';
    import Utils from '../assets/js/utils.js';
    let utils = new Utils('errorMessage');
    export default {
        name: 'OCVideo',
        data() {
            return {
                modules: [],
                streaming: true,
            };
        },
        mounted() {
            setTimeout(() => {
                this.init();
            }, 2000);
        },
        methods: {
            init() {
                let video = document.getElementById('videoInput');
                let cap = new cv.VideoCapture(video);

                // take first frame of the video
                let frame = new cv.Mat(video.height, video.width, cv.CV_8UC4);
                cap.read(frame);

                // hardcode the initial location of window
                // let trackWindow = new cv.Rect(150, 60, 63, 125);
                let trackWindow = new cv.Rect(49, 57, 110, 87);

                // set up the ROI for tracking
                let roi = frame.roi(trackWindow);
                let hsvRoi = new cv.Mat();
                cv.cvtColor(roi, hsvRoi, cv.COLOR_RGBA2RGB);
                cv.cvtColor(hsvRoi, hsvRoi, cv.COLOR_RGB2HSV);
                let mask = new cv.Mat();
                let lowScalar = new cv.Scalar(0, 0, 80);
                let highScalar = new cv.Scalar(140, 40, 100);
                let low = new cv.Mat(hsvRoi.rows, hsvRoi.cols, hsvRoi.type(), lowScalar);
                let high = new cv.Mat(hsvRoi.rows, hsvRoi.cols, hsvRoi.type(), highScalar);
                cv.inRange(hsvRoi, low, high, mask);
                let roiHist = new cv.Mat();
                let hsvRoiVec = new cv.MatVector();
                hsvRoiVec.push_back(hsvRoi);
                cv.calcHist(hsvRoiVec, [0], mask, roiHist, [180], [0, 180]);
                cv.normalize(roiHist, roiHist, 0, 255, cv.NORM_MINMAX);

                // delete useless mats.
                roi.delete();
                hsvRoi.delete();
                mask.delete();
                low.delete();
                high.delete();
                hsvRoiVec.delete();

                // Setup the termination criteria, either 10 iteration or move by at least 1 pt
                let termCrit = new cv.TermCriteria(cv.TERM_CRITERIA_EPS | cv.TERM_CRITERIA_COUNT, 10, 1);

                let hsv = new cv.Mat(video.height, video.width, cv.CV_8UC3);
                let hsvVec = new cv.MatVector();
                hsvVec.push_back(hsv);
                let dst = new cv.Mat();
                let trackBox = null;

                const FPS = 30;
                const processVideo = () => {
                    try {
                        if (!this.streaming) {
                            // clean and stop.
                            frame.delete();
                            dst.delete();
                            hsvVec.delete();
                            roiHist.delete();
                            hsv.delete();
                            return;
                        }
                        let begin = Date.now();

                        // start processing.
                        cap.read(frame);
                        cv.cvtColor(frame, hsv, cv.COLOR_RGBA2RGB);
                        cv.cvtColor(hsv, hsv, cv.COLOR_RGB2HSV);
                        cv.calcBackProject(hsvVec, [0], roiHist, dst, [0, 180], 1);

                        // apply camshift to get the new location
                        [trackBox, trackWindow] = cv.CamShift(dst, trackWindow, termCrit);

                        // Draw it on image
                        let pts = cv.rotatedRectPoints(trackBox);
                        cv.line(frame, pts[0], pts[1], [255, 0, 255, 255], 3);
                        cv.line(frame, pts[1], pts[2], [255, 0, 255, 255], 3);
                        cv.line(frame, pts[2], pts[3], [255, 0, 255, 255], 3);
                        cv.line(frame, pts[3], pts[0], [255, 0, 255, 255], 3);
                        cv.imshow('canvasOutput', frame);

                        // schedule the next one.
                        let delay = 1000 / FPS - (Date.now() - begin);
                        setTimeout(processVideo, delay);
                    } catch (err) {
                        utils.printError(err);
                    }
                };

                // schedule the first one.
                setTimeout(processVideo, 0);
            },
        },
    };
</script>

<style scoped></style>
