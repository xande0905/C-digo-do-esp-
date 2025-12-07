#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <NTPClient.h>
#include <WiFiUdp.h>
#include <EEPROM.h> 

// ===========================================
//          1. CONFIGURAÇÕES GERAIS
// ===========================================

// --- CREDENCIAIS DO WI-FI ---
const char* ssid = "Josiane_ Exa internet"; 
const char* password = "84196619";

// --- PINOS DOS RELÉS ---
const int pinoLed = 5;      // D1 (LED Panel) - CONTROLADO PELO ESP

// --- AJUSTE DE POLARIDADE DO RELÉ ---
const int RELAY_ON = HIGH;  
const int RELAY_OFF = LOW; 

// --- CONFIGURAÇÃO DE TEMPO ---
const long timeOffset = -10800; // -3 horas em segundos (Horário de Brasília)

// --- MEMÓRIA EEPROM (Indices) ---
const int EEPROM_SIZE = 50; 
const int ADDR_CICLO = 0;       
const int ADDR_LAST_TOGGLE = 1; 
const int ADDR_MODE = 5;        
const int ADDR_LED_STATE = 6;   
const int ADDR_MANUAL_STATE = 7;
const int ADDR_EXAUSTOR_MODE = 10; 

// --- OBJETOS ---
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "pool.ntp.org", timeOffset, 60000); 
ESP8266WebServer server(80);

// --- VARIÁVEIS DE ESTADO (ESP) ---
int cicloAtual = 1; 
unsigned long lastToggleTime = 0;
int ledState = RELAY_OFF; 
int currentMode = 0; // 0=AUTO, 1=MANUAL
int systemStatus = 1; 

// --- VARIÁVEIS DO EXAUSTOR (APENAS MODO PARA DISPLAY) ---
int exaustorMode = 2; // Inicia em Auto Intervalado

// --- VARIÁVEIS DO NANO (RECEBIDO VIA SERIAL) ---
int umidadeSolo = 32; // Valor atualizado pelo NANO
String serialBuffer = "";
const char END_MARKER = '\n'; 

// --- VARIÁVEIS PARA COMUNICAÇÃO SERIAL (ENVIO) ---
unsigned long lastSendTime = 0;
const long interval = 5000; // Envia dados a cada 5 segundos

// ===========================================
//          2. FUNÇÕES DE PERSISTÊNCIA (EEPROM)
// ===========================================

void EEPROM_write_ul(int addr, unsigned long val) {
  byte* p = (byte*)&val;
  for (int i = 0; i < 4; i++) {
    EEPROM.write(addr + i, *p++);
  }
}

unsigned long EEPROM_read_ul(int addr) {
  unsigned long val;
  byte* p = (byte*)&val;
  for (int i = 0; i < 4; i++) {
    *p++ = EEPROM.read(addr + i);
  }
  return val;
}

// ===========================================
//          3. LÓGICA DE CONTROLE (LED)
// ===========================================

String secondsToHMS(unsigned long totalSeconds) {
  int hours = totalSeconds / 3600;
  int minutes = (totalSeconds % 3600) / 60;
  int seconds = totalSeconds % 60;
  
  String h = (hours < 10) ? "0" + String(hours) : String(hours);
  String m = (minutes < 10) ? "0" + String(minutes) : String(minutes);
  String s = (seconds < 10) ? "0" + String(seconds) : String(seconds);
  
  return h + ":" + m + ":" + s;
}

void verificarCiclo() {
  timeClient.update();
  
  // --- Lógica do LED ---
  if (currentMode == 0) { 
    unsigned long currentTime = timeClient.getEpochTime();
    unsigned long elapsedTime = currentTime - lastToggleTime;

    unsigned long requiredDuration;
    
    if (cicloAtual == 1) { 
      requiredDuration = (ledState == RELAY_ON) ? 64800 : 21600;
    } else { 
      requiredDuration = 43200;
    }

    if (elapsedTime >= requiredDuration) {
      ledState = (ledState == RELAY_ON) ? RELAY_OFF : RELAY_ON;
      lastToggleTime = currentTime;
      EEPROM_write_ul(ADDR_LAST_TOGGLE, lastToggleTime);
      EEPROM.write(ADDR_LED_STATE, (ledState == RELAY_ON) ? 1 : 0); 
      EEPROM.commit();
      digitalWrite(pinoLed, ledState);
    }
  }
}

void iniciarNovoCiclo(int novoCiclo) {
  currentMode = 0; 
  
  cicloAtual = novoCiclo;
  ledState = RELAY_ON;
  digitalWrite(pinoLed, ledState);
  
  timeClient.update();
  lastToggleTime = timeClient.getEpochTime();

  EEPROM.write(ADDR_CICLO, cicloAtual);
  EEPROM.write(ADDR_MODE, currentMode);
  EEPROM.write(ADDR_LED_STATE, (ledState == RELAY_ON) ? 1 : 0);
  EEPROM_write_ul(ADDR_LAST_TOGGLE, lastToggleTime);
  EEPROM.commit();
}

// ===========================================
//          4. COMUNICAÇÃO SERIAL COM NANO
// ===========================================

// Envia comandos (BOMBA/EXAUSTOR) para o Nano no formato CHAVE:VALOR\n
void enviarComandoParaNano(String key, String value) {
    Serial.print(key);
    Serial.print(":");
    Serial.print(value);
    Serial.print(END_MARKER);
}

// Recebe dados (UMIDADE) do Nano
void receberDadosDoNano() {
  while (Serial.available()) {
    char rc = Serial.read();
    
    if (rc != END_MARKER) {
      serialBuffer += rc;
    } else {
      int separatorIndex = serialBuffer.indexOf(':');
      
      if (separatorIndex > 0) {
        String key = serialBuffer.substring(0, separatorIndex);
        String valueStr = serialBuffer.substring(separatorIndex + 1);
        
        if (key == "UMIDADE") {
          if (valueStr.length() > 0 && isDigit(valueStr.charAt(0))) {
            umidadeSolo = valueStr.toInt();
          }
        }
      }
      serialBuffer = "";
    }
  }
}

// NOVO: Função para enviar telemetria para o Nano (Hora, LED, Ciclo)
void enviarDadosDeTelemetriaParaNano() {
  timeClient.update();
  
  String currentHour = timeClient.getFormattedTime();
  String ledStatus = (digitalRead(pinoLed) == RELAY_ON) ? "1" : "0";
  // Envia "MAN" se estiver em modo manual, ou o ciclo automático
  String cycleName = (currentMode == 1) ? "MAN" : (cicloAtual == 1 ? "18/6" : "12/12"); 
  
  // Formato Esperado pelo Nano: H:00:00:00|LED:0|CICLO:---
  String dataPacket = "H:" + currentHour + "|LED:" + ledStatus + "|CICLO:" + cycleName;
  
  Serial.print(dataPacket);
  Serial.print(END_MARKER); // Envia '\n'
}

// ===========================================
//          5. ROTINAS DO SERVIDOR WEB
// ===========================================

// --- Handlers de LED/Ciclo ---
void handleSet18_6() { iniciarNovoCiclo(1); server.sendHeader("Location", "/"); server.send(303); }
void handleSet12_12() { iniciarNovoCiclo(2); server.sendHeader("Location", "/"); server.send(303); }
void handleManualON() {
  currentMode = 1; ledState = RELAY_ON; digitalWrite(pinoLed, ledState);
  EEPROM.write(ADDR_MODE, 1); EEPROM.write(ADDR_MANUAL_STATE, 1); EEPROM.commit();
  server.sendHeader("Location", "/"); server.send(303);
}
void handleManualOFF() {
  currentMode = 1; ledState = RELAY_OFF; digitalWrite(pinoLed, ledState);
  EEPROM.write(ADDR_MODE, 1); EEPROM.write(ADDR_MANUAL_STATE, 0); EEPROM.commit();
  server.sendHeader("Location", "/"); server.send(303);
}

// --- Handlers de Irrigação (Comandos para o Nano) ---
void handleRegarAgora() { 
  enviarComandoParaNano("BOMBA", "PULSO"); 
  server.sendHeader("Location", "/"); server.send(303); 
}
void handleBombaON() { 
  enviarComandoParaNano("BOMBA", "1"); // 1 = MANUAL ON
  server.sendHeader("Location", "/"); server.send(303); 
}
void handleBombaOFF() { 
  enviarComandoParaNano("BOMBA", "0"); // 0 = MANUAL OFF
  server.sendHeader("Location", "/"); server.send(303); 
}
void handleBombaAuto() {
  enviarComandoParaNano("BOMBA", "AUTO"); 
  server.sendHeader("Location", "/"); server.send(303); 
}

// --- Handlers de Exaustor (Comandos para o Nano) ---
void setExaustorMode(int mode, String value) {
  exaustorMode = mode; // Atualiza a variável de display do ESP
  enviarComandoParaNano("EXAUSTOR", value); // Envia o comando real para o Nano
  EEPROM.write(ADDR_EXAUSTOR_MODE, exaustorMode);
  EEPROM.commit();
}
void handleExaustorOff() { setExaustorMode(0, "0"); server.sendHeader("Location", "/"); server.send(303); }
void handleExaustorOn() { setExaustorMode(1, "1"); server.sendHeader("Location", "/"); server.send(303); }
void handleExaustorIntervalado() { setExaustorMode(2, "AUTO"); server.sendHeader("Location", "/"); server.send(303); }
void handleExaustorCool() { setExaustorMode(1, "1"); server.sendHeader("Location", "/"); server.send(303); } 

// --- Handlers de Sistema ---
void handleManualTotal() { 
  currentMode = 1; EEPROM.write(ADDR_MODE, 1); 
  setExaustorMode(0, "0"); // Desliga Exaustor via Nano
  handleBombaOFF(); // Desliga Bomba via Nano
  EEPROM.commit();
  server.sendHeader("Location", "/"); server.send(303); 
}
void handleReboot() {
  String html = "<h1>Reiniciando ESP8266...</h1><p>Aguarde cerca de 5 segundos.</p><script>setTimeout(function(){window.location.href='/';}, 5000);</script>";
  server.send(200, "text/html", html);
  delay(100);
  ESP.restart();
}

// Página principal (HTML)
void handleRoot() {
  // ... (código HTML/CSS completo mantido) ...
  timeClient.update();
  unsigned long currentTime = timeClient.getEpochTime();
  
  unsigned long requiredDuration = 0;
  unsigned long remainingTime = 0;
  String modeName;
  String faseAtual;
  String cycleName;
  
  if (currentMode == 1) {
    modeName = "MANUAL OVERRIDE";
    faseAtual = (digitalRead(pinoLed) == RELAY_ON) ? "LUZ" : "ESCURO";
    cycleName = (cicloAtual == 1) ? "18/6" : "12/12";
  } else {
    modeName = "AUTOMÁTICO";
    cycleName = String((cicloAtual == 1) ? "18/6" : "12/12");
    faseAtual = (ledState == RELAY_ON) ? "LUZ" : "ESCURO";

    unsigned long elapsedTime = currentTime - lastToggleTime;
    if (cicloAtual == 1) {
      requiredDuration = (ledState == RELAY_ON) ? 64800 : 21600; 
    } else {
      requiredDuration = 43200; 
    }
    remainingTime = (requiredDuration > elapsedTime) ? requiredDuration - elapsedTime : 0;
  }
  
  String exaustorStatusText = (exaustorMode != 0) ? "ON" : "OFF";
  String exaustorModeText;
  if (exaustorMode == 0) exaustorModeText = "Desligado Total";
  else if (exaustorMode == 1) exaustorModeText = "Contínuo (ON)";
  else if (exaustorMode == 2) exaustorModeText = "Intervalado (Nano)";
  else exaustorModeText = "Comando Manual";
  
  String overallModeText = (currentMode == 1) ? "Manual LED" : "Automático Completo";
  if (currentMode == 1 && exaustorMode == 0) overallModeText = "Manual Total";
  
  String statusText = (systemStatus == 1) ? "Perfeito" : "Alerta";
  String statusColor = (systemStatus == 1) ? "#32e676" : "#ff4747";
  String ledStatusCircle = (digitalRead(pinoLed) == RELAY_ON) ? "#32e676" : "#ff4747";
  String exaustorStatusCircle = (exaustorMode != 0) ? "#32e676" : "#f2c744";
  
  String umidadeSoloStr = String(umidadeSolo) + "%";
  int umidadeSoloPercent = umidadeSolo;
  
  String corSaude;
  String textoSaude;
  if (umidadeSoloPercent > 70) { 
    corSaude = "#f2c744"; 
    textoSaude = "⚠ Atenção (Muito Úmido)";
  } else if (umidadeSoloPercent > 30) {
    corSaude = "#32e676"; 
    textoSaude = "✔ Perfeito";
  } else {
    corSaude = "#ff4747"; 
    textoSaude = "🔥 Crítico (Seco)";
  }
  
  String html = "<!DOCTYPE html><html lang='pt-BR'><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width, initial-scale=1.0'>";
  html += "<title>ZYGROWBOX PRO Dashboard</title>";
  
  // 🎨 Estilos Profissionais
  html += "<style>";
  html += "body{background-color: #1a1a1a; color: #f4f4f9; font-family: 'Roboto', sans-serif; margin: 0; padding: 0;}";
  html += ".header{position: sticky; top: 0; background: linear-gradient(135deg, #222, #1a1a1a); padding: 10px 15px; box-shadow: 0 4px 10px rgba(0,0,0,0.5); z-index: 100;}";
  html += ".header-title{color: #3aaaff; font-size: 1.2em; font-weight: 700; margin-bottom: 5px;}";
  html += ".header-status{display: flex; justify-content: space-around; font-size: 0.8em; margin-top: 5px; opacity: 0.8;}";
  html += ".status-item{text-align: center;}";
  html += ".status-circle{width: 8px; height: 8px; border-radius: 50%; display: inline-block; margin-right: 4px;}";
  html += ".container{padding: 10px; max-width: 600px; margin: 0 auto;}";
  html += ".card{background-color: #222; border-radius: 10px; padding: 15px; margin-bottom: 20px; box-shadow: 0 4px 10px rgba(0,0,0,0.3); border: 1px solid #333;}";
  html += ".card-title{color: #3aaaff; font-size: 1.1em; margin-bottom: 10px; padding-bottom: 5px; border-bottom: 1px solid #333;}";
  html += ".data-item{display: flex; justify-content: space-between; margin-bottom: 8px; font-size: 0.9em;}";
  html += ".data-value{font-weight: bold;}";
  html += ".countdown{font-size: 2.5em; color: #32e676; font-weight: 700; margin: 10px 0;}";
  html += ".btn-group{display: flex; flex-wrap: wrap; justify-content: center; margin-top: 10px;}";
  html += ".btn{padding: 8px 12px; margin: 5px; border: none; border-radius: 20px; color: #1a1a1a; font-weight: 600; cursor: pointer; transition: background-color 0.3s, transform 0.1s;}";
  html += ".btn-on{background-color: #32e676;} .btn-off{background-color: #ff4747;} .btn-cycle{background-color: #f2c744; color: #1a1a1a;} .btn-special{background-color: #3aaaff; color: #1a1a1a;}";
  html += ".btn:active{transform: scale(0.98);}";
  html += ".status-bar{height: 8px; background-color: #333; border-radius: 4px; overflow: hidden; margin-top: 10px;}";
  html += ".bar-fill{height: 100%; transition: width 0.5s;}";
  html += ".status-alert{color: #ff4747; font-weight: bold;} .status-ok{color: #32e676; font-weight: bold;}";
  html += ".chart-container{text-align: center; margin: 10px 0;}";
  html += ".chart{width: 80px; height: 80px; border-radius: 50%; display: inline-flex; align-items: center; justify-content: center; font-weight: bold; font-size: 0.8em; background: conic-gradient(" + corSaude + " " + String(umidadeSoloPercent) + "%, #444 0%);}";
  html += "@media (max-width: 400px) {.btn{font-size: 0.8em; padding: 6px 10px;}}";
  html += "</style>";
  html += "</head><body>";
  
  // ⭐ 1. HEADER FIXO
  html += "<div class='header'>";
  html += "<div class='header-title' style='text-align: center;'>🎛 ZYGROWBOX PRO DASHBOARD</div>";
  html += "<div style='text-align: center; font-size: 1.5em; font-weight: 700; color: white;' id='time-display'>" + timeClient.getFormattedTime() + "</div>";
  html += "<div class='header-status'>";
  html += "<div class='status-item'><span class='status-circle' style='background-color: " + ledStatusCircle + ";'></span>LED</div>";
  html += "<div class='status-item'><span class='status-circle' style='background-color: " + exaustorStatusCircle + ";'></span>Exaustor</div>";
  html += "<div class='status-item'><span class='status-circle' style='background-color: #333;'></span>Irrigação</div>";
  html += "<div class='status-item' style='font-weight: bold; color: " + statusColor + ";'>Modo: " + statusText + "</div>";
  html += "</div>"; 
  html += "</div>"; 
  
  html += "<div class='container'>";
  html += "<div style='height: 1px; background: linear-gradient(to right, #1a1a1a, #3aaaff, #1a1a1a); margin: 20px 0;'></div>";
  
  // 🌱 2. Cartão “Ambiente”
  html += "<div class='card'>";
  html += "<div class='card-title'>🌱 Ambiente e Sensores</div>";
  html += "<div style='display: flex; justify-content: space-around; align-items: center;'>";
  
  // Gráfico Circular de Umidade do Solo
  html += "<div class='chart-container'>";
  html += "Umidade Solo:<br>";
  html += "<div class='chart'>" + umidadeSoloStr + "</div>";
  html += "</div>";
  
  html += "<div>";
  html += "<div class='data-item'><span>Temperatura:</span> <span class='data-value'>--°C</span></div>";
  html += "<div class='data-item'><span>Umidade do Ar:</span> <span class='data-value'>--%</span></div>";
  html += "<div class='data-item'><span>Última Irrigação:</span> <span class='data-value'>--:--</span></div>";
  html += "<div class='data-item'><span>Próxima Prevista:</span> <span class='data-value'>--:--</span></div>";
  html += "</div>"; 
  html += "</div>"; 
  
  // Barra de Saúde
  html += "<div style='margin-top: 15px; font-size: 0.9em;'>Saúde do Ambiente: <span style='color: " + corSaude + ";'>" + textoSaude + "</span></div>";
  html += "<div class='status-bar'><div class='bar-fill' style='width: " + String(umidadeSoloPercent) + "%; background-color: " + corSaude + ";'></div></div>"; 
  html += "</div>"; 
  
  // 💡 3. Ciclo de Luz
  html += "<div class='card'>";
  html += "<div class='card-title'>💡 Ciclo de Luz - Painel LED</div>";
  html += "<div class='data-item'><span>Modo Atual:</span> <span class='data-value' style='color: " + statusColor + ";'>" + modeName + "</span></div>";
  html += "<div class='data-item'><span>Ciclo Ativo:</span> <span class='data-value'>" + cycleName + "</span></div>";
  html += "<div class='data-item'><span>Fase Atual:</span> <span class='data-value' style='color: " + ledStatusCircle + ";'>" + faseAtual + "</span></div>";
  
  if (currentMode == 0) {
    String proximaFaseTexto = (faseAtual == "LUZ" ? "ESCURO" : "LUZ");
    html += "<p style='text-align: center; font-size: 0.9em; margin: 5px 0;'>Tempo Restante:</p>";
    html += "<div class='countdown' id='countdown-display' style='color: " + ledStatusCircle + ";'>" + secondsToHMS(remainingTime) + "</div>";
    html += String("<div style='text-align: center; font-size: 0.8em; opacity: 0.7;'>Até a troca para ") + proximaFaseTexto + "</div>"; 
  } else {
    html += "<p class='status-alert' style='text-align: center; margin: 20px 0;'>CONTROLE MANUAL ATIVO</p>";
  }
  
  html += "<div style='height: 1px; background-color: #333; margin: 15px 0;'></div>";
  html += "<div class='btn-group'>";
  html += "<a href='/manual_on'><button class='btn btn-on'>Ligar Agora</button></a>";
  html += "<a href='/manual_off'><button class='btn btn-off'>Desligar Agora</button></a>";
  html += "<a href='/set18_6'><button class='btn btn-cycle'>Reiniciar 18/6</button></a>";
  html += "<a href='/set12_12'><button class='btn btn-cycle'>Reiniciar 12/12</button></a>";
  html += "<button class='btn btn-special' style='width: 100%;'>Amanhecer/Anoitecer Suave</button>"; 
  html += "</div>";
  html += "</div>"; 
  
  // 💦 4. Irrigação Inteligente
  html += "<div class='card'>";
  html += "<div class='card-title'>💦 Irrigação Inteligente</div>";
  html += "<div class='data-item'><span>Umidade do Solo:</span> <span class='data-value'>" + umidadeSoloStr + "</span></div>";
  html += "<div class='data-item'><span>Status:</span> <span class='data-value status-ok'>Pronto</span></div>";
  html += "<div class='data-item'><span>Modo Bomba:</span> <span class='data-value'>Automático (Nano)</span></div>";
  html += "<div style='height: 1px; background-color: #333; margin: 15px 0;'></div>";
  html += "<div class='btn-group'>";
  html += "<a href='/bomba_on'><button class='btn btn-on'>Bomba ON (Manual)</button></a>";
  html += "<a href='/bomba_off'><button class='btn btn-off'>Bomba OFF (Manual)</button></a>";
  html += "<a href='/regar_agora'><button class='btn btn-special' style='background-color: #3aaaff;'>Regar Agora (Pulso)</button></a>";
  html += "<a href='/bomba_auto'><button class='btn btn-cycle'>Modo Automático</button></a>";
  html += "</div>";
  html += "</div>"; 
  
  // 🌬 5. Exaustor + Ventilação
  html += "<div class='card'>";
  html += "<div class='card-title'>🌬 Exaustor + Ventilação</div>";
  html += "<div class='data-item'><span>Status Atual:</span> <span class='data-value' style='color: " + exaustorStatusCircle + ";'>" + exaustorStatusText + "</span></div>";
  html += "<div class='data-item'><span>Modo:</span> <span class='data-value'>" + exaustorModeText + "</span></div>";
  html += "<div class='data-item'><span>Controlado por:</span> <span class='data-value'>Arduino Nano</span></div>";
  html += "<div style='height: 1px; background-color: #333; margin: 15px 0;'></div>";
  html += "<div class='btn-group'>";
  html += "<a href='/exaustor_on'><button class='btn btn-on'>ON Contínuo</button></a>";
  html += "<a href='/exaustor_off'><button class='btn btn-off'>OFF Total</button></a>";
  html += "<a href='/exaustor_intervalado'><button class='btn btn-special'>Intervalado (Auto)</button></a>";
  html += "<a href='/exaustor_cool'><button class='btn btn-cycle'>Resfriamento Inteligente</button></a>";
  html += "</div>";
  html += "</div>"; 
  
  // 🧠 6. Modos do Sistema
  html += "<div class='card'>";
  html += "<div class='card-title'>🧠 Central de Controle</div>";
  html += "<div class='data-item'><span>Modo Atual:</span> <span class='data-value' style='color: #32e676;'>" + overallModeText + "</span></div>";
  html += "<div style='height: 1px; background-color: #333; margin: 15px 0;'></div>";
  html += "<div class='btn-group'>";
  html += "<a href='/set18_6'><button class='btn btn-on' style='width: 100%;'>Automático Completo</button></a>"; 
  html += "<button class='btn btn-cycle'>Semiautomático</button>"; 
  html += "<a href='/manual_total'><button class='btn btn-off'>Manual Total</button></a>"; 
  html += "<button class='btn btn-special'>Failsafe</button>"; 
  html += "<a href='/reboot'><button class='btn btn-off' style='background-color: #555; color: #ccc;'>Reboot ESP</button></a>";
  html += "</div>";
  html += "</div>"; 
  
  // 📊 7. Logs e Histórico (Placeholder)
  html += "<div class='card'>";
  html += "<div class='card-title'>📊 Logs Recentes</div>";
  html += "<ul style='list-style-type: none; padding: 0; margin: 0; font-size: 0.85em;'>";
  html += "<li style='border-bottom: 1px dotted #333; padding: 5px 0;'>[02:13:00] LED: Fase LUZ iniciada (18h).</li>";
  html += "<li style='border-bottom: 1px dotted #333; padding: 5px 0;'>[01:05:40] Irrigação: Solo < 30%. Bomba ON (8s).</li>";
  html += "<li style='border-bottom: 1px dotted #333; padding: 5px 0;'>[00:50:22] Exaustor: Entrou em modo Intervalado.</li>";
  html += "<li style='padding: 5px 0;'>[00:00:00] Sistema: ESP8266 Reiniciado.</li>";
  html += "</ul>";
  html += "</div>"; 
  
  // --- Script de Contagem Regressiva Suave e Relógio NTP ---
  html += "<script>";
  html += "function updateNTPTime(initialTime) {";
  html += "  let parts = initialTime.split(':');";
  html += "  let now = new Date();";
  html += "  now.setHours(parseInt(parts[0]));";
  html += "  now.setMinutes(parseInt(parts[1]));";
  html += "  now.setSeconds(parseInt(parts[2]));";
  html += "  ";
  html += "  function tick() {";
  html += "    now.setSeconds(now.getSeconds() + 1);";
  html += "    let h = now.getHours().toString().padStart(2, '0');";
  html += "    let m = now.getMinutes().toString().padStart(2, '0');";
  html += "    let s = now.getSeconds().toString().padStart(2, '0');";
  html += "    document.getElementById('time-display').innerHTML = h + ':' + m + ':' + s;";
  html += "  }";
  html += "  tick();"; 
  html += "  setInterval(tick, 1000);"; 
  html += "}";
  
  if (currentMode == 0) {
    html += "var totalSeconds = " + String(remainingTime) + ";"; 
    html += "function updateCountdown() {";
    html += "  var h = Math.floor(totalSeconds / 3600);";
    html += "  var m = Math.floor((totalSeconds % 3600) / 60);";
    html += "  var s = totalSeconds % 60;";
    html += "  h = (h < 10) ? '0' + h : h;";
    html += "  m = (m < 10) ? '0' + m : m;";
    html += "  s = (s < 10) ? '0' + s : s;";
    html += "  document.getElementById('countdown-display').innerHTML = h + ':' + m + ':' + s;";
    html += "  if (totalSeconds > 0) {";
    html += "    totalSeconds--;";
    html += "  } else {";
    html += "    window.location.reload();";
    html += "  }";
    html += "}";
    html += "updateCountdown();"; 
    html += "setInterval(updateCountdown, 1000);";
  }
  
  html += "updateNTPTime('" + timeClient.getFormattedTime() + "');"; 
  html += "</script>";
  html += "</body></html>";
  
  server.send(200, "text/html", html);
}

// ===========================================
//          6. SETUP E LOOP
// ===========================================

void setup() {
  Serial.begin(9600); 
  
  pinMode(pinoLed, OUTPUT);
  digitalWrite(pinoLed, RELAY_OFF); 

  EEPROM.begin(EEPROM_SIZE); 
  
  // Carrega variáveis da EEPROM 
  int manualStateCode = 0; 
  cicloAtual = EEPROM.read(ADDR_CICLO);
  lastToggleTime = EEPROM_read_ul(ADDR_LAST_TOGGLE);
  currentMode = EEPROM.read(ADDR_MODE);
  manualStateCode = EEPROM.read(ADDR_MANUAL_STATE); 
  exaustorMode = EEPROM.read(ADDR_EXAUSTOR_MODE); 

  if (cicloAtual != 1 && cicloAtual != 2) { 
    cicloAtual = 1; 
  }

  // --- Conexão Wi-Fi ---
  WiFi.mode(WIFI_STA); 
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }

  // --- Inicializa NTP e Sincroniza ---
  timeClient.begin();
  
  while(!timeClient.update()) {
    delay(500);
  }
  
  // --- Aplica o estado carregado do LED ---
  if (currentMode == 1) {
    ledState = (manualStateCode == 1) ? RELAY_ON : RELAY_OFF; 
    digitalWrite(pinoLed, ledState);
  } else if (lastToggleTime == 0) {
    iniciarNovoCiclo(cicloAtual); 
  } else {
    int stateCode = EEPROM.read(ADDR_LED_STATE); 
    ledState = (stateCode == 1) ? RELAY_ON : RELAY_OFF;
    digitalWrite(pinoLed, ledState);
  }
  
  // Envia o modo do Exaustor ao Nano no boot
  if (exaustorMode == 0) {
      enviarComandoParaNano("EXAUSTOR", "0");
  } else if (exaustorMode == 1) {
      enviarComandoParaNano("EXAUSTOR", "1");
  } else if (exaustorMode == 2) {
      enviarComandoParaNano("EXAUSTOR", "AUTO");
  }

  // --- Configuração das Rotas do Servidor ---
  server.on("/", handleRoot);
  server.on("/set18_6", handleSet18_6);
  server.on("/set12_12", handleSet12_12);
  server.on("/manual_on", handleManualON);
  server.on("/manual_off", handleManualOFF);
  
  server.on("/bomba_on", handleBombaON); 
  server.on("/bomba_off", handleBombaOFF);
  server.on("/bomba_auto", handleBombaAuto);
  server.on("/regar_agora", handleRegarAgora);

  server.on("/exaustor_off", handleExaustorOff); 
  server.on("/exaustor_on", handleExaustorOn); 
  server.on("/exaustor_intervalado", handleExaustorIntervalado); 
  server.on("/exaustor_cool", handleExaustorCool); 
  
  server.on("/manual_total", handleManualTotal); 
  server.on("/reboot", handleReboot); 
  server.begin();
}

void loop() {
  unsigned long tempoAtual = millis();

  // 1. Processa a lógica de tempo do LED
  verificarCiclo();

  // 2. Processa as requisições do Servidor Web
  server.handleClient();
  
  // 3. RECEBE DADOS DO NANO (Umidade)
  receberDadosDoNano();
  
  // 4. ENVIA DADOS DE TELEMETRIA PARA O NANO
  if (tempoAtual - lastSendTime >= interval) {
    enviarDadosDeTelemetriaParaNano();
    lastSendTime = tempoAtual;
  }
  
  delay(10); 
}
