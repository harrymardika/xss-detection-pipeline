# XSS Detection: Pipeline MLOps dengan TFX

Pipeline machine learning end-to-end untuk mendeteksi serangan **Cross-Site Scripting (XSS)** dari potongan teks atau HTML, dibangun dengan **TensorFlow Extended (TFX)**: validasi data, hyperparameter tuning, pelatihan CNN, evaluasi dengan TFMA, dan serving dengan **TensorFlow Serving** di Docker.

Proyek ini adalah submission kelas MLOps Dicoding (*Submission 1: Cross Site Scripting (XSS) Detection*). **Nama:** Harry Mardika · **Username Dicoding:** hkacode

## Ringkasan Hasil

| Metrik | Nilai |
|---|---|
| Accuracy / loss (train) | 0.9988 / 0.0027 |
| Validation accuracy / loss | 0.9897 / 0.0527 |
| Hyperparameter terbaik | vocab 10.000, panjang sekuens 100, embedding 112, Conv1D 96 filter, Dense 192, learning rate ≈ 0.0012 |
| Tuning | Keras Tuner RandomSearch, 30 trial, objective `val_accuracy` (terbaik 0.9917) |

Model terbaik disimpan di epoch 3 dan pelatihan berhenti di epoch 6 (early stopping). Data cukup seimbang (54% XSS), sehingga akurasi 98,97% jauh di atas tebakan kelas mayoritas.

Contoh prediksi dari model yang di-serve:

| Input | Skor | Hasil |
|---|---|---|
| `</span> <span class="reference-text">` | 0.0053 | Bukan XSS |
| `<sup onkeypress="alert(1)" contenteditable>test</sup>` | 0.99999 | XSS |

## Deskripsi Proyek

| | Deskripsi |
| --- | --- |
| **Dataset** | [Cross site scripting XSS dataset for Deep Learning](https://www.kaggle.com/datasets/syedsaqlainhussain/cross-site-scripting-xss-dataset-for-deep-learning): 13.686 kalimat (`Sentence`), label `1` = XSS (7.373) dan `0` = aman (6.313). |
| **Masalah** | XSS memungkinkan penyerang menyisipkan skrip berbahaya ke halaman web yang dilihat pengguna lain, menyebabkan pencurian informasi, pengambilalihan akun, dan celah keamanan lain. |
| **Solusi** | Model deep learning yang mengklasifikasikan input teks sebagai aman atau berbahaya berdasarkan pola yang umum dipakai dalam serangan XSS. |
| **Pengolahan** | Lowercasing dan penghapusan tanda petik satu dan dua di Transform, lalu tokenisasi dan vektorisasi dengan TextVectorization. |
| **Arsitektur** | Input string → TextVectorization → Embedding → Conv1D (kernel 3, ReLU) → GlobalMaxPooling1D → Dense (ReLU) → Dense sigmoid (0 = aman, 1 = berbahaya). |
| **Metrik** | Binary crossentropy dan accuracy untuk pelatihan; Evaluator (TFMA) juga menghitung AUC, true/false positives, true/false negatives, dan example count. Model di-*bless* jika binary accuracy ≥ 0.5 dan tidak lebih buruk dari baseline. |

## Arsitektur Pipeline

```
data/XSS_dataset.csv
  → CsvExampleGen (train:eval = 8:2)
  → StatisticsGen → SchemaGen → ExampleValidator
  → Transform (lowercase, hapus tanda kutip)
  → Tuner (30 trial) → Trainer (EarlyStopping, ModelCheckpoint)
  → Resolver (latest blessed) + Evaluator (TFMA)
  → Pusher → serving_model_dir/xss-detection-model/1
  → TensorFlow Serving (Docker)
```

Pipeline dijalankan langkah demi langkah dengan `InteractiveContext` di `hkacode-training.ipynb`.

## Tech Stack

TFX 1.11.0, TensorFlow Transform, TensorFlow Model Analysis, Keras Tuner, TensorFlow/Keras, TensorFlow Serving, Docker, scikit-learn, Jupyter.

## Struktur Proyek

```
xss-detection-pipeline/
├── data/XSS_dataset.csv
├── xss_transform.py / xss_tuner.py / xss_trainer.py
├── hkacode-training.ipynb       # Pipeline TFX interaktif
├── hkacode-testing.ipynb        # Request prediksi ke TF Serving
├── hkacode-pipeline/            # Artefak komponen TFX dan metadata
├── serving_model_dir/xss-detection-model/1/
├── model-metadata.png           # Metadata model di TF Serving
├── Dockerfile
└── requirements.txt
```

## Cara Menjalankan

```bash
git clone https://github.com/harrymardika/xss-detection-pipeline.git
cd xss-detection-pipeline
pip install -r requirements.txt
jupyter notebook hkacode-training.ipynb      # jalankan pipeline
```

Serving lokal:

```bash
docker build -t xss-detection-model . && docker run -p 8080:8501 xss-detection-model
curl http://localhost:8080/v1/models/xss-detection-model
```

Model menerima `tf.train.Example` ter-serialisasi dan di-encode base64 (contoh lengkap di `hkacode-testing.ipynb`):

```python
import base64, requests, tensorflow as tf

def predict(text: bytes):
    feat = tf.train.Feature(bytes_list=tf.train.BytesList(value=[text]))
    ex = tf.train.Example(features=tf.train.Features(feature={"Sentence": feat}))
    payload = {"signature_name": "serving_default",
               "instances": [{"examples": {"b64": base64.b64encode(ex.SerializeToString()).decode()}}]}
    return requests.post("http://localhost:8080/v1/models/xss-detection-model:predict", json=payload).json()

print(predict(b'<sup onkeypress="alert(1)" contenteditable>test</sup>'))   # skor > 0.5 berarti XSS
```

## Author

**Harry Mardika** · [GitHub](https://github.com/harrymardika)
