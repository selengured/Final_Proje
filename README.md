# Final_Proje
# Kodland Mezuniyet Projem
# Ana Python dosyası, py

import flask
import werkzeug.utils
import os

app = flask.Flask(__name__)
# Yüklenen dosyaların kaydedileceği klasör
app.config['UPLOAD_FOLDER'] = 'static/uploads'
os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)

# Yapay zeka sonucuna göre gösterilecek bilgiler
WASTE_DATABASE = {
    "plastik": {
        "isim": "Plastik Şişe",
        "mesaj": "Sarı Geri Dönüşüm Kutusuna atılmalıdır.",
        "sure": "Doğada yok olma süresi yaklaşık 450 yıldır.",
        "renk": "warning"
    },
    "kagit": {
        "isim": "Kağıt / Karton",
        "mesaj": "Mavi Geri Dönüşüm Kutusuna atılmalıdır.",
        "sure": "Doğada yok olma süresi 2-6 haftadır.",
        "renk": "info"
    },
    "cam": {
        "isim": "Cam Şişe",
        "mesaj": "Yeşil Geri Dönüşüm Kutusuna atılmalıdır.",
        "sure": "Doğada yok olma süresi yaklaşık 4000 yıldır.",
        "renk": "success"
    }
}


@app.route('/')
def index():
    climate_data = {
        "co2_seviyesi": 421.5,  # ppm
        "kuresel_sicaklik_artisi": 1.15,  # Derece
        "okyanus_isi_endeksi": 345  # Zettajoules
    }
    return flask.render_template('index.html', data=climate_data)


@app.route('/ai', methods=['GET', 'POST'])
def ai_page():
    result = None
    image_path = None

    if flask.request.method == 'POST':
        if 'file' not in flask.request.files:
            return "Dosya seçilmedi"

        file = flask.request.files['file']
        if file.filename != '':
            filename = werkzeug.utils.secure_filename(file.filename)
            filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
            file.save(filepath)

            image_path = f"static/uploads/{filename}"
            isim_kucuk = filename.lower()
            if "kagit" in isim_kucuk or "paper" in isim_kucuk:
                ai_prediction = "kagit"
            elif "cam" in isim_kucuk or "glass" in isim_kucuk:
                ai_prediction = "cam"
            else:
                ai_prediction = "plastik"  # Varsayılan

            result = WASTE_DATABASE[ai_prediction]

    return flask.render_template(
        'ai_page.html',
        result=result,
        image_path=image_path
    )


if __name__ == '__main__':
    app.run(debug=True)



## Ana Sayfa (İklim Paneli) html

<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EcoHub - İklim Paneli</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="bg-light">

<nav class="navbar navbar-expand-lg navbar-dark bg-success">
    <div class="container">
        <a class="navbar-brand" href="/">🌍 EcoHub</a>
        <div class="collapse navbar-collapse">
            <ul class="navbar-nav ms-auto">
                <li class="nav-item"><a class="nav-link active" href="/">İklim Paneli</a></li>
                <li class="nav-item"><a class="nav-link" href="/ai_page.html">Fotoğraftan Çöp Kutusuna</a></li>
            </ul>
        </div>
    </div>
</nav>

<div class="container mt-5">
    <h2 class="mb-4 text-success">Gerçek Zamanlı İklim Verileri</h2>
    
    <div class="row mb-4">
        <div class="col-md-4">
            <div class="card shadow-sm border-0">
                <div class="card-body text-center">
                    <h5 class="card-title text-muted">Küresel CO2 Seviyesi</h5>
                    <h2 class="text-danger">432 <small>ppm</small></h2>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card shadow-sm border-0">
                <div class="card-body text-center">
                    <h5 class="card-title text-muted">Sıcaklık Anomalisi</h5>
                    <h2 class="text-warning">+ 1.55 <small>°C</small></h2>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card shadow-sm border-0">
                <div class="card-body text-center">
                    <h5 class="card-title text-muted">Okyanus Isı Endeksi</h5>
                    <h2 class="text-info"> 21 <small>°C</small></h2>
                </div>
            </div>
        </div>
    </div>

    <div class="card shadow-sm border-0">
        <div class="card-body">
            <canvas id="iklimGrafik" height="80"></canvas>
        </div>
    </div>
</div>

<script>
    const ctx = document.getElementById('iklimGrafik').getContext('2d');
    new Chart(ctx, {
        type: 'line',
        data: {
            labels: ['2018', '2019', '2020', '2021', '2022', '2023', '2024'],
            datasets: [{
                label: 'Yıllık Karbon Emisyonu Artışı',
                data: [408, 411, 414, 416, 418, 420, 421.5],
                borderColor: 'rgba(255, 99, 132, 1)',
                backgroundColor: 'rgba(255, 99, 132, 0.2)',
                borderWidth: 2,
                fill: true
            }]
        }
    });
</script>

</body>
</html>



#ai py

from flask import Flask, render_template, request
from tensorflow.keras.models import load_model
from PIL import Image, ImageOps
import numpy as np
import os

app = Flask(__name__)

# Yüklenen resimlerin geçici olarak kaydedileceği klasör
app.config['UPLOAD_FOLDER'] = 'static/uploads'
os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)

model = load_model('keras_model.h5', compile=False)
class_names = open('labels.txt', 'r', encoding='utf-8').readlines()
@app.route('/')
def index():
    return (
        "🌍 EcoHub İklim Paneli Ana Sayfası. Atık tanıma için lütfen "
        "<a href='/ai'>/ai</a> sayfasına gidin."
    )


@app.route('/ai', methods=['GET', 'POST'])
def ai_page():
    if request.method == 'POST':
        file = request.files.get('file')
        if file and file.filename != '':
            filepath = os.path.join(app.config['UPLOAD_FOLDER'], file.filename)
            file.save(filepath)

            image = Image.open(filepath).convert('RGB')
            size = (224, 224)
            image = ImageOps.fit(image, size, Image.Resampling.LANCZOS)
            
            image_array = np.asarray(image)
            # Normalize image to range [-1, 1]
            image_float = image_array.astype(np.float32)
            normalized_image_array = (image_float / 127.5) - 1
            
            data = np.ndarray(shape=(1, 224, 224, 3), dtype=np.float32)
            data[0] = normalized_image_array

            prediction = model.predict(data)
            index = np.argmax(prediction)
            
            class_name = class_names[index].strip()
            
            atik_turu = (
                class_name.split(' ', 1)[1]
                if ' ' in class_name
                else class_name
            ) 

            result = belirle_atik_bilgileri(atik_turu)

            image_path_html = filepath.replace('\\', '/')

            return render_template('ai_page.html', result=result, image_path=image_path_html)

    return render_template('ai_page.html', result=None)

def belirle_atik_bilgileri(atik_turu):
    """Teachable Machine etiketlerine göre renk, mesaj ve süre eşleştirmesi yapar."""
    atik_turu = atik_turu.strip()
    
    if atik_turu == "Plastic Bottle":
        return {
            "isim": "Plastik Şişe",
            "renk": "warning",  # Bootstrap Sarı
            "mesaj": "Bunu SARI renkli plastik geri dönüşüm kutusuna atmalısın.",
            "sure": "Plastik şişelerin doğada çözünmesi yaklaşık 1000 yıl sürer!"
        }
    elif atik_turu == "Paper":
        return {
            "isim": "Kağıt",
            "renk": "primary",  # Bootstrap Mavi
            "mesaj": "Bunu MAVİ renkli kağıt geri dönüşüm kutusuna atmalısın.",
            "sure": "Geri dönüştürülen 1 ton kağıt tam 17 ağacı kesilmekten kurtarır."
        }
    elif atik_turu == "Cardboard":
        return {
            "isim": "Karton",
            "renk": "primary",  # Karton da mavi kutuya gider
            "mesaj": "Bunu MAVİ renkli kağıt/karton geri dönüşüm "
                     "kutosuna atmalısın.",
            "sure": "Karton ambalajların geri dönüştürülmesi çevre "
                    "kirliliğini %35 azaltır."
        }
    elif atik_turu == "Metal":
        return {
            "isim": "Metal (Kutu/Ambalaj)",
            "renk": "secondary",  # Bootstrap Gri
            "mesaj": "Bunu GRİ renkli metal geri dönüşüm kutusuna atmalısın.",
            "sure": "Alüminyum içecek kutuları doğada 200 ila 500 yıl arası kalabilir!"
        }
    else:
        return {
            "isim": "Bilinmeyen Nesne",
            "renk": "dark",
            "mesaj": "Bu nesnenin geri dönüşüm sınıfını tam olarak ayırt edemedim.",
            "sure": "Emin değilsen evsel atık (çöp) kutusuna atabilirsin."
        }

if __name__ == '__main__':
    app.run(debug=True)



    #ai.html

    <!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EcoHub - Yapay Zeka Atık Tanıma</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="{{ url_for('static', filename='ai_page.css') }}">
</head>
<body class="bg-light">

<nav class="navbar navbar-expand-lg navbar-dark bg-success">
    <div class="container">
        <a class="navbar-brand" href="/">🌍 EcoHub</a>
        <div class="collapse navbar-collapse">
            <ul class="navbar-nav ms-auto">
                <li class="nav-item"><a class="nav-link" href="/">İklim Paneli</a></li>
                <li class="nav-item"><a class="nav-link active" href="/ai">Fotoğraftan Çöp Kutusuna</a></li>
            </ul>
        </div>
    </div>
</nav>

<div class="container mt-5">
    <div class="row justify-content-center">
        <div class="col-md-6">
            <h2 class="text-center text-success mb-4">Fotoğraftan Çöp Kutusuna 📸♻️</h2>
            
            <div class="card shadow-sm border-0 mb-4">
                <div class="card-body">
                    <form action="/ai" method="POST" enctype="multipart/form-data">
                        <div class="mb-3">
                            <label for="formFile" class="form-label">Atığın fotoğrafını yükleyin:</label>
                            <input class="form-control" type="file" id="formFile" name="file" accept="image/*" required>
                        </div>
                        <button type="submit" class="btn btn-success w-100">Yapay Zekaya Analiz Ettir</button>
                    </form>
                </div>
            </div>

            {% if result %}
            <div class="card shadow-sm border-0 border-top border-{{ result.renk }} border-4">
                <div class="card-body text-center">
                    <img src="{{ image_path or '' }}" alt="Yüklenen Atık" class="img-thumbnail mb-3" style="max-height: 200px;">
                    <h4 class="text-{{ result.renk }}">Bu bir {{ result.isim }}!</h4>
                    <p class="fs-5 fw-bold">{{ result.mesaj }}</p>
                    <div class="alert alert-danger mt-3" role="alert">
                        ⚠️ {{ result.sure }}
                    </div>
                </div>
            </div>
            {% endif %}

        </div>
    </div>
</div>

</body>
</html>
