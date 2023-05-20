# Face

### 얼굴인식 출석 시스템


얼굴인식 출석 시스템 (Face-Api와 Java Script 활용) 사진을 업로드 하고 업로드 된 사진과 실시간 웹캠에 나오는 얼굴에서 눈, 눈썹, 코, 입 등 특징을 추출하여 비교후 일치 하는 정도가 일정 임계값을 넘으면 출석, 넘지 못하면 미출석. 이후 출석데이터를 배열로 저장. 이름을 적고 이름 또한 배열로 저장. 출석 버튼과 다운로드 버튼을 누르면 저장 되었던 배열들이 모두 엑셀 파일로 출석부(이름, 출석시간, 출석여부)로 출력 가능


## 실행방법
1. 파일을 다운 받고 압축을 푼다.
2. 크롬 브라우저에서 Web Server for Chrome 을 설치하고
3. [압축을 푼 폴더]를 선택한다.
4. http://127.0.0.1:8887 로 접속


## 구현 영상
https://youtu.be/spfE8jnRjAU


## 소스 코드
```ruby
<!DOCTYPE html>
<html>
  <head>
    <title>안동대학교 얼굴인식 출석 시스템</title>
    <link href="https://raw.githubusercontent.com/kmans1123/img/main/%EB%A1%9C%EA%B3%A0.png" rel="shortcut icon" type="image/x-icon">
    <style>
      body {
        background-image: url('https://raw.githubusercontent.com/kmans1123/img/main/%EC%95%88%EB%8F%99%EB%8C%80.png');
        background-size: cover;
      }
    </style>
    <script defer src="face-api.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.16.9/xlsx.full.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/FileSaver.js/2.0.5/FileSaver.min.js"></script>
  </head>
  <body>
    <h1>안동대학교 얼굴인식 출석 시스템</h1>
    <input type="file" id="imageInput" accept="image/*" />
    <button onclick="extractFeatures()">사진에서 추출하기</button>
    <button onclick="startWebcamExtraction()">웹캠에서 추출하기</button>
    <p id="message"></p>
    <video id="video" width="640" height="480" autoplay></video>
    <canvas id="canvas" width="640" height="480"></canvas>
    <br />
    <label for="nameInput">Name:</label>
    <input type="text" id="nameInput" />
    <button onclick="markAttendance()">출석</button>
    <button onclick="downloadAttendance()">다운로드</button>
    <p id="attendanceMessage"></p>

    <script>
      let attendanceData = [];
      let uploadedDescriptors = null;
      let messageElement = document.getElementById('message');
      let attendanceMessageElement = document.getElementById('attendanceMessage');

      async function loadModels() {
        try {
          await Promise.all([
            faceapi.nets.faceRecognitionNet.loadFromUri('./models'),
            faceapi.nets.faceLandmark68Net.loadFromUri('./models'),
            faceapi.nets.ssdMobilenetv1.loadFromUri('./models')
          ]);
          messageElement.textContent = '모델이 로딩 완료되었습니다.';
        } catch (error) {
          console.error('모델 로딩 중 오류가 발생했습니다:', error);
          messageElement.textContent = '모델 로딩에 실패하였습니다.';
        }
      }

      async function detectFace() {
        const video = document.getElementById('video');
        const canvas = document.getElementById('canvas');
        const displaySize = { width: video.width, height: video.height };
        faceapi.matchDimensions(canvas, displaySize);

        messageElement.textContent = '모델 로딩 중...';

        await loadModels();

        setInterval(async () => {
          const detections = await faceapi.detectAllFaces(video).withFaceLandmarks().withFaceDescriptors();
          const resizedDetections = faceapi.resizeResults(detections, displaySize);
          canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
          faceapi.draw.drawDetections(canvas, resizedDetections);
          faceapi.draw.drawFaceLandmarks(canvas, resizedDetections);

          if (uploadedDescriptors) {
            resizedDetections.forEach((detection) => {
              const descriptor = detection.descriptor;
              const name = findBestMatch(descriptor).toString();
              const box = detection.detection.box;

              const drawBox = new faceapi.draw.DrawBox(box, { label: name });
              drawBox.draw(canvas);
            });
          }
        }, 100);
      }

      function findBestMatch(queryDescriptor) {
        const faceMatcher = new faceapi.FaceMatcher(uploadedDescriptors, 0.6);
        const bestMatch = faceMatcher.findBestMatch(queryDescriptor);
        return bestMatch.toString();
      }

      async function getFaceDescriptorsFromImages(images) {
        return Promise.all(
          images.map(async (image) => {
            const img = await faceapi.fetchImage(URL.createObjectURL(image)); // 이미지 URL로 변환
            const detections = await faceapi.detectSingleFace(img).withFaceLandmarks().withFaceDescriptor();
            return detections.descriptor;
          })
        );
      }

      async function extractFeatures() {
        const imageInput = document.getElementById('imageInput');
        const images = Array.from(imageInput.files);

        if (images.length === 0) {
          messageElement.textContent = '이미지를 업로드해주세요.';
          return;
        }

        messageElement.textContent = '이미지에서 얼굴 특징 추출 중...';

        const descriptors = await getFaceDescriptorsFromImages(images);
        uploadedDescriptors = new faceapi.LabeledFaceDescriptors('custom', descriptors);
        messageElement.textContent = '사진에서 얼굴 특징 추출이 완료되었습니다.';
      }

      function startWebcamExtraction() {
        const video = document.getElementById('video');
        const canvas = document.getElementById('canvas');
        const messageElement = document.getElementById('message');

        navigator.mediaDevices
          .getUserMedia({ video: true })
          .then((stream) => {
            video.srcObject = stream;
            messageElement.textContent = '웹캠에서 얼굴 특징 추출 중...';

            const detectionsInterval = setInterval(async () => {
              const detections = await faceapi.detectAllFaces(video).withFaceLandmarks().withFaceDescriptors();
              if (detections.length > 0) {
                clearInterval(detectionsInterval);
                uploadedDescriptors = detections[0].descriptor;
                console.log('웹캠에서 얼굴 특징 추출 완료:', uploadedDescriptors);
                messageElement.textContent = '웹캠에서 얼굴 특징 추출이 완료되었습니다.';
              }
            }, 100);
          })
          .catch((error) => {
            console.error('웹캠 스트림 시작 중 오류가 발생했습니다:', error);
          });
      }

      function markAttendance() {
        const nameInput = document.getElementById('nameInput');
        const name = nameInput.value;

        if (!name) {
          attendanceMessageElement.textContent = '이름을 입력해주세요.';
          return;
        }

        const timestamp = new Date().toLocaleString();
        const attendanceRecord = { name, timestamp };
        attendanceData.push(attendanceRecord);
        attendanceMessageElement.textContent = `출석이 완료되었습니다. 이름: ${name}, 시간: ${timestamp}`;
        nameInput.value = '';
      }

      function downloadAttendance() {
  const wb = XLSX.utils.book_new();
  const wsData = attendanceData.map(record => ({ Name: record.name, Timestamp: record.timestamp, '출석 여부': '출석' }));
  const ws = XLSX.utils.json_to_sheet(wsData);
  XLSX.utils.book_append_sheet(wb, ws, '출석부');
  const wbout = XLSX.write(wb, { bookType: 'xlsx', type: 'binary' });

  function s2ab(s) {
    const buf = new ArrayBuffer(s.length);
    const view = new Uint8Array(buf);
    for (let i = 0; i < s.length; i++) {
      view[i] = s.charCodeAt(i) & 0xff;
    }
    return buf;
  }

  const blob = new Blob([s2ab(wbout)], { type: 'application/octet-stream' });
  saveAs(blob, '출석부.xlsx');
}


      window.onload = function () {
        detectFace();
      };
    </script>
  </body>
</html>
```
