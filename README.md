<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Video Stream via WebRTC (Local Network)</title>
</head>
<body>
    <h2>WebRTC Video Stream (Local Network)</h2>
    <video id="localVideo" width="640" height="480" autoplay muted></video>
    <video id="remoteVideo" width="640" height="480" autoplay></video>

    <script>
        const localVideo = document.getElementById('localVideo');
        const remoteVideo = document.getElementById('remoteVideo');
        
        let localStream;
        let peerConnection;
        const iceServers = { iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] };  // STUN сервер для WebRTC

        // 1. Получаем доступ к камере
        navigator.mediaDevices.getUserMedia({ video: true })
            .then((stream) => {
                localStream = stream;
                localVideo.srcObject = stream;
                createPeerConnection();
            })
            .catch((error) => {
                console.error("Ошибка при доступе к камере: ", error);
            });

        // 2. Создаем WebRTC соединение
        function createPeerConnection() {
            peerConnection = new RTCPeerConnection(iceServers);

            // Добавляем локальный видео поток
            localStream.getTracks().forEach(track => {
                peerConnection.addTrack(track, localStream);
            });

            // Когда кандидат ICE найден, отправляем его
            peerConnection.onicecandidate = function(event) {
                if (event.candidate) {
                    sendSignal('iceCandidate', event.candidate);
                }
            };

            // Когда приходит удаленный поток, показываем его
            peerConnection.ontrack = function(event) {
                remoteVideo.srcObject = event.streams[0];
            };

            // Создаем предложение (offer)
            peerConnection.createOffer()
                .then((offer) => {
                    return peerConnection.setLocalDescription(offer);
                })
                .then(() => {
                    sendSignal('offer', peerConnection.localDescription);
                })
                .catch((error) => {
                    console.error('Ошибка при создании offer: ', error);
                });
        }

        // Функция для отправки сигналов
        function sendSignal(type, data) {
            const signal = {
                type: type,
                data: data
            };

            // Отправка сигнала по локальному IP (замени на IP своего устройства)
            const ip = 'http://192.168.1.2:3000';  // Локальный IP адрес
            fetch(`${ip}/signal`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(signal),
            });
        }

        // Обработка сигнала
        function handleSignal(signal) {
            if (signal.type === 'offer') {
                peerConnection.setRemoteDescription(new RTCSessionDescription(signal.data))
                    .then(() => {
                        return peerConnection.createAnswer();
                    })
                    .then((answer) => {
                        return peerConnection.setLocalDescription(answer);
                    })
                    .then(() => {
                        sendSignal('answer', peerConnection.localDescription);
                    });
            } else if (signal.type === 'answer') {
                peerConnection.setRemoteDescription(new RTCSessionDescription(signal.data));
            } else if (signal.type === 'iceCandidate') {
                peerConnection.addIceCandidate(new RTCIceCandidate(signal.data));
            }
        }
    </script>
</body>
</html>
