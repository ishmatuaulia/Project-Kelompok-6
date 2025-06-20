# Project-Kelompok-6

3.1 Implementasi dan Kode Program 

3.1.1 Kode Rust Modbus Client SHT 20 MAIN DAN CARGO 

Kode program dRust Modbus Client menunjukkan cara membaca data sensor menggunakan protokol komunikasi Modbus RTU melalui koneksi serial pada perangkat berbasis Rust. Proses pembacaan dimulai dengan konfigurasi awal koneksi serial, yaitu menentukan port perangkat (misalnya /dev/ttyUSB0), kecepatan baud rate (dalam hal ini 9600 bps), serta pengaturan parameter komunikasi seperti jumlah data bit, paritas, dan stop bit. Setelah konfigurasi dibuat, port serial dibuka dan koneksi Modbus RTU diinisialisasi menggunakan pustaka tokio-modbus, kemudian ditentukan ID slave dari perangkat sensor yang ingin dibaca, misalnya 0x01. 

Selanjutnya, pembacaan data dilakukan pada alamat register tertentu. Misalnya, untuk membaca suhu digunakan alamat register 0x0001 dan untuk kelembaban pada alamat 0x0002. Kedua register ini dibaca sebagai register input sebanyak satu unit (16-bit). Nilai mentah yang diperoleh dari register berupa bilangan u16 kemudian dikonversi menjadi float agar sesuai dengan skala fisik menggunakan fungsi konversi yang mempertimbangkan kemungkinan nilai bertanda (signed). Dalam kasus ini, nilai dibagi dengan 10.0 agar didapatkan nilai suhu dan kelembaban dalam satuan desimal yang akurat. 

Setelah nilai suhu dan kelembaban berhasil diperoleh dan dikonversi, data ini kemudian dikirim ke InfluxDB — sebuah time-series database — menggunakan format Line Protocol. Data ini dikirim melalui HTTP request menggunakan pustaka reqwest, di mana setiap data menyertakan pengukuran, tag lokasi, field data (temperature dan humidity), serta cap waktu (timestamp). Proses ini dilakukan secara berkala menggunakan perulangan dengan jeda waktu tertentu (2 detik) untuk memastikan pembaruan data sensor yang kontinu dan real-time. Dengan demikian, kode ini menunjukkan alur lengkap mulai dari komunikasi perangkat keras hingga integrasi dengan sistem penyimpanan data berbasis waktu. 

use influxdb2::Client; 
use influxdb2::models::DataPoint; 
use serde_json::Value; 
use std::time::{SystemTime, UNIX_EPOCH}; 
use tokio::io::AsyncReadExt; 
use tokio::net::TcpListener; 
use futures::stream; 

#[tokio::main] 
async fn main() -> Result<(), Box<dyn std::error::Error>> { 
    // 1. Setup InfluxDB Client 
    let client = Client::new( 
    "http://localhost:8086",  
    "ITS", 
    "DLLJdieOs-jXD4UN3pZWCYd0w1DXITdUDzluvkDdcMeZ0quW8ZAnXocU9ghqv2VJ-dbzWMGG3ZW5-X6CZ2t-eQ==" 
  ); 
 
    // 2. Setup TCP Server 
    let listener = TcpListener::bind("127.0.0.1:8080").await?; 
    println!("Server running on port 8080"); 
    
    loop { 
        let (mut socket, _) = listener.accept().await?; 
        let client = client.clone(); 
  tokio::spawn(async move { 
            let mut buffer = [0; 1024]; 
            match socket.read(&mut buffer).await { 
              Ok(n) => { 
                 // BAGIAN PARSING DATA 
                 let raw_data = String::from_utf8_lossy(&buffer[..n]); 
                 if let Ok(parsed) = serde_json::from_str::<Value>(&raw_data) { 
                 let temp = parsed["temperature"].as_f64().unwrap_or(0.0); 
                 let hum = parsed["humidity"].as_f64().unwrap_or(0.0); 
                 
                 // 3. Create DataPoint 
                 let point = DataPoint::builder("sensor_data") 
                     .field("temperature", temp) 
                     .field("humidity", hum) 
                     .timestamp( 
                       SystemTime::now() 
                       .duration_since(UNIX_EPOCH) 
                       .unwrap() 
                       .as_nanos() as i64 
                      ) 
                  .build() 
                  .unwrap(); 
                  
                // 4. Convert to Stream 
                let points = stream::iter(vec![point]); 
                         
                // 5. Write to InfluxDB 
                match client.write("sensor", points).await { 
                Ok(_) => println!("Data: {:.1}°C, {:.1}%", temp, hum), 
                Err(e) => eprintln!("Write error: {}", e), 
               } 
              } else { 
                eprintln!("Failed to parse JSON data"); 
              } 
            } 
            Err(e) => eprintln!("Socket error: {}", e), 
          } 
       }); 
     } 

3.1.2 Kode Rust TCP Server 

TCP server ini berfungsi sebagai perantara utama dalam sistem pengumpulan dan penyimpanan data sensor. Server ini didesain untuk menerima koneksi dan data dari client Modbus, yaitu perangkat atau sistem yang melakukan pembacaan data dari sensor-sensor di lapangan. Setelah menerima data tersebut, TCP server melakukan proses parsing dan validasi agar data yang diterima sesuai dengan format dan struktur yang diharapkan.  

Langkah kerja TCP Server 

1. Server membuka listener pada alamat 0.0.0.0:7878 untuk menerima koneksi dari client secara terbuka di semua antarmuka jaringan. 
2. Setiap koneksi yang masuk akan ditangani secara asynchronous dengan memanfaatkan tokio::spawn, sehingga server dapat menangani banyak koneksi secara bersamaan tanpa memblokir thread utama. 
3. Data dari client dikirim dalam format JSON per baris (dipisahkan dengan newline \n) dan dibaca secara bertahap menggunakan BufReader untuk efisiensi pembacaan stream. 
4. Setiap baris JSON kemudian di-deserialize menjadi struct SensorData, yang merepresentasikan data sensor dengan field seperti nama sensor, nilai, dan timestamp. 
5. Waktu pengukuran (timestamp) akan dikonversi ke dalam format Unix epoch dalam satuan detik agar kompatibel dengan format penyimpanan time-series di InfluxDB. 
6. Data yang telah diolah akan diformat ke dalam Line Protocol, yaitu format standar penulisan data untuk InfluxDB, lalu dikirim menggunakan HTTP request melalui pustaka reqwest. 

Kode Fungsi Rust TCS Server 
Import dan Inisialisasi 

use tokio_modbus::prelude::*; 
use tokio_serial::SerialStream; 
use std::time::Duration; 
use reqwest::Client; 
use chrono::Utc; 

Fungsi: 
tokio_modbus: komunikasi Modbus RTU 
tokio_serial: membuka port serial 
reqwest: HTTP client untuk mengirim data ke InfluxDB 
chrono: menangani waktu (timestamp) 

Fungsi main() (Asynchronous) 

#[tokio::main] 
async fn main() -> Result<(), Box<dyn std::error::Error>> { 
 
Fungsi utama asynchronous menggunakan runtime Tokio, agar bisa menangani operasi I/O (seperti Modbus dan HTTP) tanpa blocking. 

Konfigurasi Serial 

let tty_path = "/dev/ttyUSB0"; 
let slave_address = 0x01; 
let baud_rate = 9600; 

Fungsi: 
Menentukan port serial tempat sensor terhubung (/dev/ttyUSB0). 
Mengatur baud rate dan alamat slave (Modbus device ID). 
Membuka Koneksi Serial dan Membuat Context Modbus: 

let builder = tokio_serial::new(tty_path, baud_rate) 
  .timeout(...) 
  ... 
let port = SerialStream::open(&builder)?; 
let mut ctx = rtu::connect(port).await?; 
ctx.set_slave(Slave(slave_address)); 

Fungsi:  
Mengatur konfigurasi port serial (data bits, parity, stop bits, timeout). 
Menghubungkan serial stream ke client Modbus RTU. 
Mengatur alamat slave sensor. 

Looping Pembacaan dan Pengiriman Data 

loop { 
... 
tokio::time::sleep(Duration::from_secs(2)).await; 
} 

Fungsi: Perulangan tanpa henti, membaca dan mengirim data setiap 2 detik. 


Membaca Register Temperatur 

let temp_reg = ctx.read_input_registers(0x0001, 1).await?; 
let raw_temp = temp_reg[0]; 
let temperature = convert_to_float(raw_temp, 10.0); 

Fungsi:  
Membaca input register 0x0001 (asumsi untuk suhu). 
Nilai register diubah menjadi float menggunakan fungsi konversi. 

Fungsi convert_to_float() 

fn convert_to_float(raw_value: u16, divisor: f32) -> f32 { 
    if raw_value > 32767 { 
      (raw_value as i16) as f32 / divisor 
    } else { 
      raw_value as f32 / divisor 
    } 
} 

Fungsi:  
Mengubah nilai register (16-bit unsigned) menjadi float 
Jika nilai >32767, dianggap sebagai nilai negatif (signed), lalu dibagi dengan faktor skala (dalam kasus ini: 10) 

Fungsi write_to_influxdb() 

async fn write_to_influxdb(temperature: f32, humidity: f32) -> Result<...> { 
... 
} 


Konfigurasi InfluxDB 

let influx_url = "..."; 
let token = "..."; 

Fungsi: 
URL endpoint untuk menulis data ke InfluxDB v2 
Token autentikasi (harus dijaga kerahasiaannya) 

 
Menyiapkan Format Line Protocol 

let timestamp = Utc::now().timestamp(); 
let line = format!( 
"suhu_kelembaban,location=fermentasi temperature={},humidity={} {}", 
temperature, humidity, timestamp 
); 

Fungsi:  
Membuat satu baris data InfluxDB dengan tag location=fermentasi 
Diisi dengan nilai temperature dan humidity, serta timestamp (Unix epoch dalam detik) 

Mengirim ke InfluxDB 

let res = client 
.post(influx_url) 
.header("Authorization", format!("Token {}", token)) 
.header("Content-Type", "text/plain") 
.body(line) 
.send() 
.await?; 

Fungsi:  
Mengirim data ke InfluxDB melalui HTTP POST. 
Jika berhasil, akan mencetak konfirmasi. Jika gagal, mencetak pesan kesalahan dari response. 


3.1.3 Konfigurasi InfluxDB dan Integrasi 

Server dikonfigurasi untuk berjalan pada port 7878, menerima koneksi dari client secara asynchronous guna memastikan pengolahan data berlangsung secara efisien dan non-blocking. Setiap koneksi yang masuk akan membaca data dalam format JSON, kemudian memprosesnya menggunakan pustaka serde_json untuk diubah menjadi objek terstruktur sesuai dengan skema data Rust. 

Data sensor yang diterima selanjutnya dikirim dan disimpan ke InfluxDB dengan struktur sebagai berikut: 

1. Measurement:  
monitoring — berfungsi sebagai kategori utama untuk pengelompokan data pengukuran. 

2. Tags (digunakan untuk filtering dan identifikasi data):: 
sensor_id: ID unik sensor, misalnya SHT20-PascaPanen-001. 
shipment_id: Identitas kontainer atau lokasi fisik, misalnya KONTAINER-001. 
process_stage: Tahapan proses produksi, misalnya Fermentasi. 

3. Fields (berisi nilai pengukuran aktual yang akan dianalisis): 
temperature_celsius: Suhu yang diukur dalam satuan derajat Celsius. 
humidity_percent: Tingkat kelembaban dalam satuan persen. 

4. Timestamp: 
Mengacu pada waktu pengambilan data oleh sensor, dicatat dalam format RFC3339, kemudian dikonversi ke format Unix timestamp (detik) untuk disesuaikan dengan kebutuhan InfluxDB v2 (dengan presisi s). 

3.1.4 Dashboard Grafana 
Pada Dahsboard Grafana ni adalah menampilkan data sensor secara visual melalui Grafana. Grafana dihubungkan langsung ke InfluxDB sebagai sumber data utama, yang menyimpan parameter suhu dan kelembaban dari berbagai pengiriman buah. 

Dashboard menampilkan dua grafik utama, yaitu: 

1. Grafik suhu terhadap waktu (temperature_celsius) 
2. Grafik kelembaban terhadap waktu (humidity_percent)

Setiap grafik dapat difilter berdasarkan shipment_id, sehingga operator atau pengguna dapat memantau kondisi masing-masing kontainer secara terpisah. Selain grafik, disediakan juga panel statis yang menampilkan nilai sensor terkini. 
Grafana dikonfigurasi dengan fitur filter waktu (seperti 1 jam terakhir, hari ini, atau 7 hari terakhir) untuk memudahkan pemantauan tren kondisi. Sistem ini juga dapat dikembangkan lebih lanjut dengan menambahkan alerting—yaitu pemberitahuan otomatis jika suhu atau kelembaban melebihi ambang batas tertentu, guna mendukung pengambilan keputusan yang lebih cepat dalam pengiriman buah. Berikut tampilan dashboard Grafana:  

Sebuah gambar berisi teks, cuplikan layar, Software multimedia, software

Konten yang dihasilkan AI mungkin salah., Picture 

Sebuah gambar berisi cuplikan layar, teks, Software multimedia, software

Konten yang dihasilkan AI mungkin salah., Picture 

3.1.5 Integrasi Blockchains dan Web 3 

Pengembangan smart contract monitoring.sol (solidity) 

Import dan setup awal  
use ethers::prelude::*; 
use std::sync::Arc; 
use tokio::net::{TcpListener, TcpStream}; 
use tokio::io::{AsyncReadExt, AsyncWriteExt}; 
use dotenv::dotenv; 
use std::env; 
use std::error::Error; 
use serde::Deserialize; 

Fungsi:  
Mengimpor semua utilitas dari library ethers-rs, digunakan untuk mengakses Ethereum lockchain. 
tokio::net dan tokio::io Untuk membuat TCP server asynchronous. 
dotenv Untuk mengambil konfigurasi dari file .env 
env Untuk membaca variabel lingkungan. 
serde::Deserialize Digunakan agar data JSON yang masuk bisa diubah ke bentuk struct Rust. 

Smart Contract Binding (abigen)  

abigen!( 
    SensorData, 
    "./SensorData.json", 
    event_derives(serde::Deserialize, serde::Serialize) 
); 

Fungsi:  
abigen! adalah macro dari ethers-rs untuk membuat binding ke smart contract berdasarkan file ABI JSON. 
event_derives digunakan agar event-event dari smart contract bisa di-serialize/deserialize ke JSON. 

Fungsi Kirim ke Blockchain 

async fn send_reading_to_blockchain(...) -> Result<TxHash, String> 

Fungsi:  
Mengirim data ke method add_reading pada smart contract. 
Menunggu konfirmasi transaksi (await) dan mencetak hash-nya jika sukses. 

Struct untuk Data JSON Masuk 

#[derive(Deserialize, Debug)] 
struct SensorPayload { 
    temperature: f32, 
    humidity: f32, 
} 

Fungsi Utama Main() 
#[tokio::main] 
async fn main() -> Result<(), Box<dyn Error>> 

Mendefinisikan struct DataPoint untuk menyimpan timestamp, location, temperature (int, dikali 10), dan humidity (int, dikali 10). 

contract SensorData { 
  // Struktur data untuk setiap pembacaan sensor 
  struct Reading { 
  uint256 timestamp;   // Waktu pencatatan data 
  string location;     // Lokasi sensor, misal "Gudang-A1" 
  int256 temperature;  // Suhu (misal, 25.5°C disimpan sebagai 255) 
  uint256 humidity;    // Kelembapan (misal, 60.2% disimpan sebagai 602) 
 } 
 
Fungsi:  
Struktur ini digunakan untuk mewakili satu entri data hasil pembacaan sensor. Struct ini menjadi cetakan/format standar data yang akan disimpan atau diproses di blockchain. 
 
Membuat fungsi addDataPoint() dengan modifier onlyOwner untuk menambahkan data baru. 

// Modifier untuk membatasi akses fungsi hanya untuk 'owner' 
    modifier onlyOwner() { 
        require(msg.sender == owner, "Akses ditolak: Anda bukan pemilik."); 
         _; 
        } 
Fungsi;  
Membatasi akses ke fungsi-fungsi tertentu agar hanya bisa dipanggil oleh pemilik kontrak (owner). 

Membuat skrip deployment (scripts/deploy.js) menggunakan ethers.js (via Hardhat) dan Menjalankan skrip deploy (npx hardhat run scripts/deploy.js --network localhost) di terminal lain untuk menerbitkan kontrak ke node Hardhat lokal. 

const hre = require("hardhat"); 
async function main() { 
  console.log("Mempersiapkan deployment..."); 

  // Mengompilasi dan men-deploy kontrak 'SensorData' 
  const sensorData = await hre.ethers.deployContract("SensorData"); 

  // Menunggu hingga proses deployment selesai 
  await sensorData.waitForDeployment(); 

  // Mencetak alamat kontrak yang sudah di-deploy ke konsol 
  console.log(✅ Smart contract 'SensorData' berhasil di-deploy ke alamat: ${sensorData.target}); 
} 

// Pola yang direkomendasikan untuk menggunakan async/await di mana-mana 
// dan menangani error dengan benar. 
main().catch((error) => { 
  console.error(error); 
  process.exitCode = 1; 
}); 

Fungsi: 
Men-deploy smart contract SensorData ke jaringan blockchain yang dikonfigurasi di Hardhat, lalu mencetak alamat kontraknya. 
Membuat file .env di root workspace Rust (TUGAS4) untuk menyimpan PRIVATE_KEY (dari akun Hardhat Node #0) dan CONTRACT_ADDRESS (dari hasil deploy). 

Membuat fungsi asinkron write_to_blockchain yang Terhubung ke Hardhat Node (http://127.0.0.1:8545). 
Membuat instance dari smart contract menggunakan alamat dan ABI. 

# File .env 

# URL RPC dari node Hardhat lokal Anda (biasanya ini) 
RPC_URL="ws://127.0.0.1:8545" 
# Private key dari salah satu akun Hardhat (copy-paste dari terminal 'npx hardhat node') 
# Pastikan diawali dengan "0x" 
PRIVATE_KEY="0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80" 

# Alamat smart contract 'SensorData' yang telah di-deploy 
CONTRACT_ADDR="0x5FbDB2315678afecb367f032d93F642f64180aa3" 

Fungsi: 
Menghubungkan ke node blockchain lokal (Hardhat) 
Alamat kontrak SensorData di blockchain, sebagai target transaksi 

Membuat file index.html (struktur dan sedikit CSS) dan app.js di dalam folder proyek-blockchain. 

Konfigurasi Konstanta 
const contractAddress = "..."; 
const contractABI = [...]; 
const blockExplorerBaseURL = "https://sepolia.etherscan.io/address/"; 

Fungsi: Digunakan untuk membuat link ke blockchain explorer (Etherscan) agar pengguna bisa melihat transaksi kontrak secara publik. 

SOP (Standard Operating Procedure) 
const SOP_MIN_TEMP = 28; 
const SOP_MAX_TEMP = 32; 
Batas suhu dan kelembapan ideal untuk validasi apakah data batch sesuai standar operasional. 

Ambil Elemen HTM 
const connectButton = document.getElementById('connectButton'); 
Fungsi: Mengambil elemen-elemen DOM (tombol, input, label, dll.) agar bisa dikontrol dari JavaScript. 

 Variabel Global 

let provider, signer, contract; 
let allReadingsCache = null; 
let currentChart = null; 

Fungsi: Untuk menyimpan koneksi ke blockchain dan menyimpan cache data sensor agar tidak fetch berkali-kali. 

Fungsi Utama 

async function connectWallet() { ... } 
async function searchBatchData() { ... } 
async function fetchAllReadingsFromChain() { ... } 
function displayResults(batchReadings, batchCode) { ... } 
function renderTable(data) { ... } 
function renderChart(data) { ... } 

Fungsi:  
connectWallet : Hubungkan MetaMask. 
searchBatchData : Ambil data berdasarkan batch. 
fetchAllReadingsFromChain :  Tarik semua data dari kontrak. 
displayResults : Tampilkan ringkasan. 
renderTable dan renderChart :  Tampilkan detail data. 

 3.1.6 Hasil yang dicapai 

Prototipe sistem monitoring suhu dan kelembaban yang berjalan secara real-time dan terintegrasi dari sensor hingga penyimpanan data terpusat dan terdesentralisasi. 
Visualisasi data yang intuitif melalui Grafana (berdasarkan data InfluxDB) dan aplikasi desktop PyQt (berdasarkan data InfluxDB). 
Smart contract (Monitoring.sol) yang berfungsi untuk mencatat data sensor secara transparan dan tidak dapat diubah di blockchain simulasi. 
Konektivitas penuh antara sistem instrumentasi (simulasi sensor), server pengolah data (Rust), penyimpanan data (InfluxDB).
