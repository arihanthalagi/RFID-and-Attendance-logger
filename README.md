# Task 4 – Introduction to RFID and Attendance Logger Using ESP32

## 1. Aim

To understand Radio Frequency Identification (RFID) technology, interface an RFID reader with an ESP32 to read card identification data, and develop an RFID-based attendance logging system that records attendance with timestamps in Google Sheets.

## 2. Introduction

Radio Frequency Identification (RFID) is a wireless technology used to identify objects or people using radio waves. An RFID system typically consists of an RFID tag or card and an RFID reader. The reader communicates with the card and retrieves identification data.

In this project, an ESP32 microcontroller is interfaced with an MFRC522 RFID reader. When an RFID card is brought near the reader, the system reads its unique identifier (UID) and displays the hexadecimal value on the Serial Monitor.

In the attendance logger implementation, the card UID is used to identify a registered card. The ESP32 sends the card information to Google Sheets through a cloud-based script, where the attendance record and timestamp are stored for future reference.

## 3. Components Required

1. ESP32 DevKit V1
2. MFRC522 RFID reader module
3. RFID cards or tags
4. Jumper wires
5. USB data cable
6. Wi-Fi connection
7. Computer with Arduino IDE
8. Google account for Google Sheets and Apps Script

## 4. Part I – Introduction to RFID

### 4.1 Working Principle of RFID

RFID works through wireless communication between an RFID reader and an RFID tag. The reader generates a radio-frequency field, and a compatible nearby tag responds with identification data.

The MFRC522 module operates at 13.56 MHz and supports compatible 13.56 MHz RFID cards and tags, including certain MIFARE cards.

When a card is placed near the reader:

1. The reader detects the card.
2. The ESP32 communicates with the reader through the SPI interface.
3. The reader retrieves the card's UID.
4. The UID is converted into hexadecimal format.
5. The hexadecimal UID is displayed on the Serial Monitor.

**Note:** Not every metro card is compatible with the MFRC522. The reader may detect some cards but cannot necessarily read their protected data. This project reads the UID of compatible cards; it does not decode encrypted metro card balances or travel records.

### 4.2 Circuit Connections

The MFRC522 communicates with the ESP32 using the SPI protocol.

| MFRC522 Pin | ESP32 Pin |
|---|---|
| 3.3V | 3.3V |
| GND | GND |
| SDA (SS) | GPIO 5 |
| SCK | GPIO 18 |
| MOSI | GPIO 23 |
| MISO | GPIO 19 |
| RST | GPIO 22 |
| IRQ | Not connected |

**Important:** Power the MFRC522 using 3.3 V. Do not connect its power pin to 5 V.

### 4.3 Arduino IDE Setup

1. Open Arduino IDE.
2. Install the ESP32 board package if it is not already installed.
3. Open Library Manager.
4. Search for and install the **MFRC522** library.
5. Select the appropriate ESP32 board and COM port.

### 4.4 ESP32 Code to Read RFID Card UID

```cpp
#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 5
#define RST_PIN 22

MFRC522 rfid(SS_PIN, RST_PIN);

void setup() {
  Serial.begin(115200);
  SPI.begin(18, 19, 23, SS_PIN);
  rfid.PCD_Init();

  Serial.println("RFID Reader Ready");
  Serial.println("Scan your RFID card");
}

void loop() {
  if (!rfid.PICC_IsNewCardPresent()) {
    return;
  }

  if (!rfid.PICC_ReadCardSerial()) {
    return;
  }

  Serial.print("Card UID: ");

  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] < 0x10) {
      Serial.print("0");
    }
    Serial.print(rfid.uid.uidByte[i], HEX);
    Serial.print(" ");
  }

  Serial.println();

  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();

  delay(500);
}
```

## Part II – RFID Attendance Logger

### 5.1 Objective

To develop an attendance logging system that identifies registered RFID cards and records attendance details, including the card UID and timestamp, in Google Sheets.

### 5.2 Working Principle

1. A registered RFID card is scanned using the MFRC522 reader.
2. The ESP32 reads the card UID.
3. The system checks whether the card belongs to a registered user.
4. The ESP32 sends the UID to a Google Apps Script web application over Wi-Fi.
5. Google Apps Script receives the data and appends an attendance entry to Google Sheets.
6. The spreadsheet records the card UID, user identification, date, and time.

### 5.3 Google Sheets Setup

Create a Google Sheet named **RFID Attendance Logger** with the following column headings:

| Column | Heading |
|---|---|
| A | Timestamp |
| B | Card UID |
| C | Name |
| D | Attendance Status |

Example attendance records:

| Timestamp | Card UID | Name | Attendance Status |
|---|---|---|---|
| 09/10/2026 09:05:12 | A34F219B | Student 1 | Present |
| 09/10/2026 09:08:35 | 7C821546 | Student 2 | Present |

These are illustrative records.

### 5.4 Google Apps Script

Open the Google Sheet and select **Extensions → Apps Script**. Add the following code:

```javascript
function doGet(e) {
  const sheet = SpreadsheetApp
    .getActiveSpreadsheet()
    .getSheetByName("Sheet1");

  const uid = String(e.parameter.uid || "")
    .replace(/[^a-fA-F0-9]/g, "")
    .toUpperCase();

  if (!uid) {
    return ContentService.createTextOutput("Missing UID");
  }

  // Replace these sample UIDs with registered card UIDs.
  const students = {
    "A34F219B": "Student 1",
    "7C821546": "Student 2"
  };

  const name = students[uid];

  if (!name) {
    return ContentService.createTextOutput("Unknown card");
  }

  const lock = LockService.getScriptLock();
  lock.waitLock(10000);

  try {
    sheet.appendRow([
      new Date(),
      uid,
      name,
      "Present"
    ]);
  } finally {
    lock.releaseLock();
  }

  return ContentService.createTextOutput("Attendance recorded");
}
```

**Note:** Replace `"Sheet1"` with the actual worksheet tab name and replace the sample UIDs and student names with your registered details.

### 5.5 Deploy the Script

1. Click **Deploy → New deployment**.
2. Select **Web app** as the deployment type.
3. Set the execution identity to your account.
4. Configure access appropriately for your use case.
5. Deploy and authorize the script.
6. Copy the web app URL ending in `/exec`.

A web app that accepts unauthenticated requests can be abused to insert false records. For a real attendance system, restrict access or add appropriate authentication and validation.

### 5.6 ESP32 Integration

The ESP32 can send the UID to the deployed web application using an HTTP GET request. Install the built-in ESP32 Wi-Fi library and use `HTTPClient` to send the request.

The request has this general form:

`WEB_APP_URL?uid=A34F219B`

Replace `WEB_APP_URL` with your deployed Apps Script URL and append the UID read by the RFID reader. The script then validates the UID against its registration list and records the attendance.

For an operational implementation, configure HTTPS, handle HTTP errors, and prevent repeated scans from creating unwanted duplicate attendance entries.

## 6. Advantages

1. Automates attendance recording.
2. Reduces manual attendance-taking effort.
3. Stores attendance data in a cloud spreadsheet.
4. Provides date and time information for each entry.
5. Makes attendance records easy to review and organize.

## 7. Applications

- College and school attendance systems
- Office employee attendance
- Laboratory access logging
- Library entry tracking
- RFID-based access control prototypes
