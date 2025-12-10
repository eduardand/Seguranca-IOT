# Relatório de análise de segurança: ESP32 Web Server

Este documento apresenta uma análise estática de segurança do código-fonte de um servidor web simples para ESP32. O objetivo é identificar vulnerabilidades, descrever vetores de ataque e classificar os riscos associados.

## 1. Introdução

O código analisado implementa um servidor HTTP básico utilizando a biblioteca `WiFi.h` para controlar pinos GPIO (26 e 27) via requisições GET. A análise foca em falhas de implementação que podem comprometer a disponibilidade e a integridade do dispositivo.


## 2. Análise de vulnerabilidades

A inspeção do código revelou as seguintes vulnerabilidades críticas:

1.  **Falta de autenticação e autorização:** O servidor web não implementa nenhum mecanismo de login ou verificação de acesso. Qualquer dispositivo na mesma rede pode enviar comandos para controlar os GPIOs.
2.  **Gerenciamento inseguro de memória:** A variável `header` é um objeto do tipo `String` que concatena caracteres recebidos (`header += c`) indefinidamente dentro do loop `while` até que o cliente desconecte ou ocorra um timeout. Isso pode levar rapidamente à fragmentação do *heap* ou exaustão da memória RAM do microcontrolador.
3.  **Credenciais *Hardcoded*:** As credenciais de Wi-Fi (`ssid` e `password`) são armazenadas em texto plano no código-fonte. Se o código for exposto (ex: GitHub público), a rede local fica comprometida.
4.  **Comunicação não criptografada (HTTP):** O uso de HTTP porta 80 transmite dados em texto claro, permitindo que atacantes na mesma rede interceptem o tráfego (*Sniffing*).


## 3. Detalhamento dos Ataques

Com base nas vulnerabilidades acima, detalhamos dois ataques possíveis.

### Ataque A: Negação de serviço (DoS) via exaustão de memória

Este ataque explora a concatenação irrestrita da variável `header`.

* **Passo-a-passo:**
    1.  O atacante se conecta à porta 80 do ESP32.
    2.  O atacante envia um fluxo contínuo de caracteres aleatórios *sem* enviar o caractere de nova linha (`\n`) ou encerrar a conexão.
    3.  O código executa `header += c` repetidamente.
    4.  A memória RAM do ESP32 enche, ou a fragmentação impede novas alocações.
    5.  O dispositivo trava (crash) ou reinicia devido ao *Watchdog Timer*, tornando o controle físico indisponível.
* **Probabilidade:** **Alta**. O código não possui verificação de tamanho máximo para a string `header`. Ferramentas simples (como `netcat` ou scripts Python) podem automatizar isso.
* **Impacto:** **Alto**. O dispositivo para de responder, impedindo o acionamento legítimo dos atuadores (lâmpadas/motores) conectados aos relés.
* **Risco Resultante:** **Crítico**. A facilidade de execução combinada com a paralisação total do serviço define este risco.

### Ataque B: Controle Não Autorizado (Replay/Direct Access)

Este ataque explora a ausência de autenticação.

* **Passo-a-passo:**
    1.  O atacante conecta-se à rede Wi-Fi (ou já está nela).
    2.  Utiliza um scanner de rede (ex: Nmap ou Fing) para descobrir o IP do ESP32.
    3.  Envia uma requisição HTTP direta via navegador ou terminal: `GET /26/on`.
    4.  O ESP32 processa o comando e ativa o GPIO 26 sem verificar quem enviou a ordem.
* **Probabilidade:** **Alta**. Não requer ferramentas avançadas, apenas um navegador web conectado à mesma rede.
* **Impacto:** **Médio/Alto**. O atacante ganha controle físico sobre o dispositivo. Se o ESP32 controlar uma porta de garagem ou um aquecedor, o impacto físico é alto. Se for apenas um LED, o impacto é médio (perda de integridade do estado do sistema). Consideraremos o pior caso (controle físico).
* **Risco Resultante:** **Alto**. A integridade do sistema é violada trivialmente.


## 5. Matriz de Riscos 

A tabela abaixo consolida os ataques identificados, ordenados do maior risco para o menor.

| Título do Ataque | Probabilidade | Impacto | Risco |
| :--- | :---: | :---: | :---: |
| **DoS por exaustão de memória** | Alta | Alto | **Crítico** |
| **Controle de GPIO não autorizado** | Alta | Médio/Alto | **Alto** |
| **Exposição de credenciais Wi-Fi** | Média | Alto | **Médio** |
